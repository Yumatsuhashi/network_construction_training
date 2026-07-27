# QA①：セキュリティ（VACL / DHCP Snooping / VLAN60）

対象config: `config/⑤セキュリティ/DSW1.txt`・`DSW2.txt`
関連: 設計書 `06_セキュリティ.md` / 設問 `09_DHCPスヌーピング`・`12_TelnetアクセスリストACL`

---

## Q1. Telnet制御を「SVIに拡張ACL(in)」ではなく VACL で書く方がよい？

**A. 用途によってはVACLの方が網羅的。今回の設計では妥当な選択なので採用した。**

- **SVI ACL（`ip access-group in`）= RACL** は、**SVIを通過するルーティング済み(L3)トラフィックだけ**をフィルタする。同一VLAN内でL2スイッチングされる通信（intra-VLAN）はSVIを通らないので素通りする。
- **VACL（`vlan filter`）** は、そのVLANでスイッチが見る**全トラフィック（L2スイッチング分を含む）**に効く。
- 要件「クライアントPCから**任意の宛先**へのTelnet禁止」を厳密に読むと、ASW跨ぎでDSWをL2通過する同一VLAN Telnet も止められるVACLの方が網羅的。

### 注意（限界）
- 同一VLAN・**同一ASW配下**のPC間Telnetは、そもそもDSWに到達しない（ASWがローカルスイッチング）ため、SVI ACLでもDSW-VACLでも捕捉できない。要件が「**DSWで**制御」と限定しているので、この限界は許容範囲。

---

## Q2. VACLの意味論（match の効き方）

`vlan access-map` の `match ip address <ACL>` は、**ACLで permit されたパケットが「マッチ」**する。deny されたものは非マッチで次シーケンスへ落ち、どこにもマッチしなければ**末尾の暗黙drop**になる。

採用した構成（DSW1/DSW2共通）:
```
ip access-list extended TELNET
 permit tcp 10.1.10.0 0.0.0.255 any eq 23
 permit tcp 10.1.20.0 0.0.0.255 any eq 23
 permit tcp 10.1.30.0 0.0.0.255 any eq 23
 permit tcp 10.1.40.0 0.0.0.255 any eq 23
 permit tcp 10.1.50.0 0.0.0.255 any eq 23
!
vlan access-map BLOCK_TELNET 10
 match ip address TELNET
 action drop            ← Telnet(permit)にマッチ → 破棄
vlan access-map BLOCK_TELNET 20
 action forward         ← 残り全部 → 転送（末尾の暗黙dropを避ける明示）
!
vlan filter BLOCK_TELNET vlan-list 10,20,30,40,50,60
```

---

## Q3. 送信元を `any` にせず `/24` に限定した理由

`permit tcp any … eq 23` は送信元が広すぎる。要件は「**クライアントPCから**任意の宛先へのTelnet禁止」なので、送信元を各クライアントセグメントの `/24`（`10.1.X0.0/24`）に限定した。
（DHCPプールは `.101-.199` だが、採番は `/24` 単位なのでサブネット指定とした。）

これにより「クライアント発（src=クライアントサブネット, dst-port 23）のTelnetだけ」を遮断する。旧 `deny tcp any any eq 23` はVLAN内の全方向Telnetを落としていたため、より要件どおり（クライアント起点のみ）の挙動になった。

---

## Q4. VACLの ACL に `permit tcp 10.1.60.0 ... eq 23` は要る？

**A. 要らない。入れてはいけない。**

`TELNET` ACL は「**遮断する対象のリスト**」（permitで拾って `seq10` で drop する）。ここに `10.1.60.0/24` を書くと **VLAN60のTelnetがマッチ→drop** され、「システム部（VLAN60）は全許可」という要件に反する。

VLAN60のパケットはACLのどのpermit行にもマッチしない → `seq10`非マッチ → `seq20 action forward` で全許可になる。よってACLには 10/20/30/40/50 だけを並べるのが正しい。

---

## Q5. `vlan filter` の `vlan-list` に 60 を入れる理由は？ なくてもよいのでは

**A. 機能的には不要。入れても入れなくてもVLAN60の挙動（全許可）は同一。** 好みで決める。

- VLAN60が「遮断されない」のを実際に決めているのは、**vlan-listのメンバーシップではなく、TELNET ACLに 10.1.60.0/24 を入れていないこと**。フィルタは送信元サブネットで効くため。
- 60を **入れない**：VLAN60にVACLが適用されず全通信そのまま転送（Ciscoの慣例：除外VLANはvlan-listに入れない）。
- 60を **入れる**：VACLは適用されるがACL非マッチで `seq20 forward` → 同じく全許可。

| | 60を入れる | 60を入れない（一般的） |
|---|---|---|
| VLAN60の挙動 | 全許可 | 全許可（同じ） |
| 設定の読み | 「対象だが何も落とさない」 | 「対象外」 |
| 将来リスク | access-mapに汎用dropを足すと巻き込む | 影響を受けない |

> 現状のconfigは `vlan-list 10,20,30,40,50,60`（60を含める＝遮断対象をACLに一元化する方針）。「入れない」に戻す判断もあり得る。

---

## Q6. DHCP Snooping は DSW に必要？（→ 削除した）

**A. 脅威モデル上ほぼ冗長。DSWからは削除した。**

- 不正DHCPサーバが繋がるのは**アクセスポート＝ASW**。実効的な防御は**ASWのSnooping**（クライアントポートをuntrustにして不正なOFFER/ACKを落とす）。
- DSWのSnoopingを外しても**DHCPは壊れない**（ASWがsnoop、DSWがDHCPサーバとしてserve）。
- DSWでSnoopingが意味を持つのは、(a) ASW跨ぎで不正OFFERがDSWをL2通過する場合の多層防御、(b) 将来 DAI / IP Source Guard でバインディングテーブルを使う場合のみ。
- 要件（設問09）は「**ASWで**不正DHCPサーバを防ぐ」。よってDSWのSnoopingは要件外と判断し削除（DAI/IPSG導入時は再検討）。

**参考：v1/v2c/v3 と同様に「plaintextかどうか」の話ではない。DHCP Snooping自体はレイヤ2の不正サーバ対策であり、削除理由は「防御点がアクセス層で足りているから」。**
