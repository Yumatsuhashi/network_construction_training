# QA③：管理（SNMPバージョン / VTYパスワードとTelnet）

対象config: `config/⑥管理項目/`（各機器）
関連: 設計書 `07_管理.md` / 設問 `14_SNMP`・`16_VTY・enable`

---

## Q1. `snmp-server host … version 2c` の 2c にする理由は？ v1 や v3 でもよい？

**A. 要件（設計書07/設問14）はバージョンを指定していない**（community名 TSHOOT / RW / Trap先 10.1.100.1 / 送信元Loopback のみ）。`version 2c` は妥当な選択で、要件が縛っているわけではない。

- このconfigは `snmp-server community TSHOOT RW` の**コミュニティ方式**。コミュニティで認証するのは **v1 と v2c** のモデル。その中で v2c が現代的な標準。
- **v2c が v1 に対して優れる点**：GetBulk（一括取得＝ポーリング効率化）／ Inform（到達確認付き通知）／ より詳細なエラー応答・改良trapフォーマット。
- `snmp-server host` でバージョン省略時のIOS既定は **v1**。`version 2c` は「あえて高機能側を使う」明示指定。

### v1 でもよい？ → 動く
`version 1` にしても同じコミュニティで成立し、要件は満たせる。ただしGetBulk/Informが使えず旧式。**機能面でv2cが上位互換**なので通常はv2cを選ぶ。

### v3 は？ → 「使えるが別物」。今のconfigのままでは不可
v3は**コミュニティ文字列を使わない**（ユーザ/グループのUSM方式）。使うなら書き換えが必要:
```
snmp-server group  MON v3 priv
snmp-server user   monuser MON v3 auth sha <PW> priv aes 128 <PW>
snmp-server host 10.1.100.1 version 3 priv monuser
```
v3は**認証＋暗号化**が付く唯一のバージョンで実運用推奨。ただし要件が「コミュニティ TSHOOT」と書かれている以上、コミュニティ方式から外れるため演習の回答としてはv1/v2cが素直。

### セキュリティ注意
**v2c は v1 より安全ではない**。v1もv2cもコミュニティを**平文送信**するので盗聴耐性は同じ。v2cの利点は機能（GetBulk/Inform）だけ。本当のセキュリティ（認証・暗号化）が欲しいなら v3 一択。

| version | 今のconfigで動くか | 位置づけ |
|---|---|---|
| 1 | ○ | 動くが旧式。GetBulk/Inform無し |
| **2c** | ○（現状） | コミュニティ方式の標準・推奨 |
| 3 | △ | 動作するが**要書き換え**。最も安全・実運用向き |

---

## Q2. VLAN60 から R1 に Telnet できるか？ `line vty` に password を設定しないとTelnetできない？

**A. `login`（既定）が有効でpassword未設定だとTelnetは拒否される**（`% Password required, but none set`）。R1は要件どおり `password cisco123 + login` が入っているのでTelnet可能。

### Cisco IOS の vty 認証パターン
| vty設定 | Telnet時の挙動 |
|---|---|
| `login`（既定）＋ password無し | **拒否** |
| `password xxx` ＋ `login` | ラインパスワードを聞かれる（R1は**これ**。cisco123） |
| `login local` ＋ `username` | ユーザ名/パスワードを聞かれる |
| `no login` | パスワード無しで接続（＝あえて無効化すれば不要／非推奨） |

「passwordが無いと入れない」のは **`login`が有効なとき**の話。`no login`なら無しでも入れる。

### 効くのは「R1側（Telnetを受ける側）」の設定
Telnetサーバは R1 なので、認証設定はR1のvtyの話。VLAN60クライアント側は普通のtelnetクライアントで設定不要。
R1接続後、特権モードは `enable secret cisco123`。

### VLAN60→R1 を成立させる他の条件
1. **⑤のVACL**：VLAN60は全許可（vlan-listに入れてもACLに60を入れていない）なのでTelnetは遮断されない（VLAN10〜50からだと `seq10 drop` で落ちる）。
2. **transport input**：R1 vtyがTelnetを許可しているか確認（`transport input ssh` だとTelnet不可）。
3. **到達性**：VLAN60(10.1.60.x)からR1宛への経路。おすすめ宛先は R1ループバック `10.1.254.5`（OSPF広告されていれば常時到達可）。

### テスト手順（VLAN60クライアントから）
```
telnet 10.1.254.5      → Password: cisco123   （vtyパスワード）
enable                 → Password: cisco123   （enable secret）
```
うまくいかない時のR1側確認: `show run | section line vty` / `show users` / VLAN60から10.1.254.5へping。
