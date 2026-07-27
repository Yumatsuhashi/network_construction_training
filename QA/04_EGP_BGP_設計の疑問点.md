# QA④：EGPルーティング（BGP）の疑問点

対象config: `config/⑦EGPルーティング/R1.txt`・`R2.txt`・`DSW1.txt` ほか
関連: 設計書 `08_拠点間接続設計.md` / 設問 `17〜22`

---

## Q1. `route-map SET-LOCALPREF-IN permit 20`（空のpermit）が必要な理由

**A. route-mapには末尾に暗黙のdenyがあるため。** このroute-mapはeBGPネイバーに **in（受信）** で適用されている。

`permit 10`（10.3.0.0/16にマッチ→LP200）だけだと:
```
10.3.0.0/16     → permit 10 → LP200で許可
それ以外の全経路 → permit 10 に非マッチ → 暗黙deny で破棄！（10.2.0.0/16 も他も全部消える）
```
`permit 20`（match句なし＝全マッチ）を置くことで「10.3以外はLPをいじらず（既定100のまま）そのまま通す」catch-allになる。設問21の補足にも明記の必須行。

---

## Q2. `ip route 10.1.0.0 255.255.0.0 Null0` が必要な理由

**A. BGPの `network` は「RIBに完全一致するエントリ」が無いと広告しない仕様だから。**

R1のRIBには `10.1.10.0/24` 等の詳細はあるが `10.1.0.0/16` ちょうどは無い。そのままでは `network 10.1.0.0 mask 255.255.0.0` は何も広告しない。
`ip route 10.1.0.0 255.255.0.0 Null0` でRIBに/16の完全一致を作り、`network` が拾えるようにする。

**Null0にする理由**:
- 集約は実体ある単一next-hopを持たない「まとめ経路」だから
- 実トラフィックはより長い一致（詳細経路）で正しく転送され、**存在しないサブネット宛だけNull0で破棄**（ループ/ブラックホール防止＝summary discard route）
- Null0は落ちないのでAD1で常時UP＝集約広告が安定

---

## Q3. `next-hop-self` が必要な理由（iBGPのnext-hop問題）

**A. iBGPは既定でnext-hopを書き換えず、eBGPのnext-hop(192.168.x.1)を内部で解決できないから。**

- R1がeBGPで学ぶ外部経路のnext-hopは `192.168.11.1`。iBGPでそのまま内部ピアへ渡ると、DSW/CSWは `192.168.11.1` を解決できない。
- `192.168.11.0/30` は **④OSPF非対象** かつ **設問19で内部に出してはいけない**。→ 経路が `next-hop inaccessible` で無効化。
- `next-hop-self` で next-hop を **R1自身のLo0（10.1.254.5、OSPF到達可）** に書き換える → 内部で解決でき経路が使える。

next-hop解決の手段は本来2つ（①next-hop-self ／ ②192.168.x.0/30をOSPFに入れる）だが、②は設問19で禁止のため①が必須。

---

## Q4. `network 10.1.0.0 mask 255.255.0.0` の代わりに `aggregate-address` でもよい？

**A. 一般論ではOK（設問20の方式2）だが、今回の設計ではそのままだと集約が生成されない。**

- `aggregate-address` は **BGPテーブルに集約対象の詳細経路（より長いprefix）が1つ以上ある**ときだけ集約を生成する。
- 本設計は詳細経路を**OSPFだけに持たせ、BGPへredistributeもnetworkもしない**ため、**BGP表に部品が無く→aggregate-addressは何も作らない**。
- **Null0静的を足しても効かない**（aggregate-addressが見るのはRIBではなく**BGPテーブル**）。
- aggregate-addressで書くなら別途 `redistribute ospf …` か各サブネットの `network` で詳細をBGP表に載せ、`summary-only` で抑制する必要がある。ただしOSPF→BGP再配布は不要経路流入リスクがあり設問20補足でも注意喚起。

| | 方式1: network + Null0（採用） | 方式2: aggregate-address |
|---|---|---|
| 前提 | RIBに完全一致（Null0静的）があればよい | **BGP表に詳細経路が必要** |
| 今回の設計での可否 | そのまま動く | **単体では不可**（詳細をBGPに載せる追加設定が要る） |
| 詳細経路の抑制 | 元々BGPに出していないので自然に集約のみ | `summary-only` で明示抑制 |

→ 「BGPに詳細を載せない」本設計方針では **方式1が素直で正解**。

---

## Q5. 「10.1.0.0/16 を広告する」根拠は問題文のどこ？

**A. 明記されている（推測ではない）。**

- 設計書08 拠点間接続設計③: 「**IP-VPN網へは集約ルート10.1.0.0/16をアドバタイズする**」（設問20の該当要件にも同文）。
- なぜ/16が妥当か → 設計書03 論理設計①「**内部で使用するアドレスレンジは10.1.0.0/16とする**」。拠点内全セグメントがこの/16に収まる採番なので、まとめると10.1.0.0/16になる。

---

## Q6. ピアグループは作る必要がある？

**A. 必須ではない（要件・設問に指定なし）。任意の最適化。** 5つのiBGPネイバーに同じ3行を繰り返しているのでまとめる価値はある。

```
router bgp 65001
 neighbor IBGP peer-group
 neighbor IBGP remote-as 65001
 neighbor IBGP update-source Loopback0
 neighbor IBGP next-hop-self
 neighbor 10.1.254.6 peer-group IBGP
 neighbor 10.1.254.3 peer-group IBGP
 neighbor 10.1.254.4 peer-group IBGP
 neighbor 10.1.254.1 peer-group IBGP
 neighbor 10.1.254.2 peer-group IBGP
```
- 利点: 設定の簡素化＋アップデート生成の効率化（グループ単位で1回生成）。動作・結果は個別記述と同一。
- 注意: **eBGPネイバー(192.168.x.1)は別**（remote-as/password/route-mapが異なるので入れない）。**アウトバウンドポリシーはグループ内で共通**である必要（今回のiBGPは全員同一なので問題なし）。
- 現行IOSでは後継の **peer-template（peer-session/peer-policy）** が推奨。学習用途ならpeer-groupで十分。

> 現状は個別記述のまま（設問の回答としては正しい）。書き換えるかは好み。

---

## Q7. フルメッシュで、DSW1 は R2 を next-hop 属性として持つ？

**A. 持つ。** R2がDSW1向けに `next-hop-self` を設定しているため、R2がDSW1へ広告する経路の **NEXT_HOP は R2のLo0＝10.1.254.6** に書き換えられる。

- R2がeBGPで学ぶ外部経路(10.2/10.3)の本来のnext-hopは 192.168.12.1 だが、それはOSPF非対象で解決不能 → `next-hop-self` で 10.1.254.6 に書き換え → DSW1はOSPFで再帰解決してR2方向へ転送できる。
- フルメッシュなのでDSW1は R1(.5) とも R2(.6) とも**直接**ピア。外部プレフィックスを両方から受け取り、**それぞれのnext-hopを保持**する。どれを使うかはベストパス選択（Local Preference）で決まる:
```
DSW1# show ip bgp （イメージ）
*>i 10.2.0.0/16  10.1.254.6  LP200  … ← R2経由がベスト
* i 10.2.0.0/16  10.1.254.5  LP100  … ← R1経由（劣後）
*>i 10.3.0.0/16  10.1.254.5  LP200  … ← R1経由がベスト
* i 10.3.0.0/16  10.1.254.6  LP100  … ← R2経由（劣後）
```
- **DSW1自身にnext-hop-selfは不要**。DSW1は外部経路を発信せず、iBGPで学んだ経路を他iBGPピアへ再広告しない（iBGPスプリットホライズン）。この性質こそ、全ノードがR1/R2両方と直接ピアを張る=フルメッシュが必要な理由（設問19）。

---

## Q8. 再配送無しのiBGPだからnext-hop-selfが必要なの？ BGP→OSPF再配送で外部経路を流し込む時は要る？

**A1（現設計）**: 正確には「**DSW/CSWがiBGPスピーカーとして外部経路を受け取り、そのBGP next-hopを内部で解決できないから**」next-hop-selfが必要（Q3参照）。

**A2（BGP→OSPF再配送する場合）**: **DSW/CSWにnext-hop-selfは不要／無関係。**
- この構成ではDSW/CSWはBGPスピーカーでなくなり、10.2/10.3を**OSPF外部経路(Type-5)**として学ぶ。BGPのnext-hop属性を持たないので、BGPの機能であるnext-hop-selfは効かない。
- next-hop解決はOSPFがやる。再配送元のnext-hop(192.168.x.1)の出口I/FはOSPF非対象なので、Type-5 LSAの**Forwarding Address(FA)は0.0.0.0**になり、外部経路は「**ASBR(=R1/R2)自身へ向けて転送**」される。内部ルータはOSPFでR1/R2まで到達でき、R1/R2が最終的に192.168.x.1へ転送する。

### ただし再配送方式には大きな代償（＝今あえてiBGPにしている理由）
OSPFはBGPの経路属性を運ばない。特に **Local Preference が失われる** → 設問21の「10.2はR2経由・10.3はR1経由」を全ノードに一貫させる仕掛けが使えない（OSPFメトリック/外部タイプ/metric調整で似せる必要があり崩れやすい）。AS-PATH等も消える。
このため本設計は再配送ではなく**iBGPフルメッシュ**を選び、その代償として next-hop-self が必要になる、という整理。

| | 経路の学び方 | next-hop解決 | next-hop-self | Local Pref伝播 |
|---|---|---|---|---|
| 現設計(iBGPフルメッシュ) | DSW/CSWがiBGPで学習 | BGP next-hop（要書換） | **必要** | ○ できる |
| 再配送(BGP→OSPF) | DSW/CSWがOSPF外部で学習 | OSPFがASBRへ誘導(FA=0) | **不要/無関係** | ✕ 失われる |

> 補足: 再配送方式でもR1↔R2間にiBGPを残すなら、そのR1↔R2セッションにはnext-hop-selfが依然として意味を持つ（OSPFに落としてDSW/CSWへ配る分には影響しない）。
