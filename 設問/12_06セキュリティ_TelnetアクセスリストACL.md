# 設問：その他ネットワーク設計⑥ ②（出典：設計書_md/06_セキュリティ.md）

> 出典：①ネットワーク要件・詳細設計書(TC).pptx スライド45

## 該当する要件
- **クライアントPCから任意の宛先に対してTelnetセッションを禁止し、その他のトラフィックは全て許可するようDSWで制御する（システム部クライアントは全てのアクセスを許可する）**

## 【設問】
> どのような技術を使用しますか？　その技術に関わるパラメータは、具体的にどのような値にしますか？（コマンドでも説明でも可）

## 回答

### 使用する技術：拡張アクセスリスト（Extended ACL）

DSWの各クライアントVLAN SVIに拡張ACLを適用し、TCPポート23（Telnet）宛の通信のみを拒否し、それ以外は全て許可する。「システム部クライアント（VLAN60）は全てのアクセスを許可する」という除外条件があるため、**VLAN60のSVIにはこのACLを適用しない**。

### 設定コマンド例

```
! ACL本体（DSW1・DSW2共通で同じ内容を用意）
ip access-list extended BLOCK_TELNET
 deny tcp any any eq 23
 permit ip any any
```

**DSW1**（VLAN10, 20, 30を収容・人事部/総務部/営業部）

```
DSW1(config)# interface Vlan10
DSW1(config-if)# ip access-group BLOCK_TELNET in
DSW1(config)# interface Vlan20
DSW1(config-if)# ip access-group BLOCK_TELNET in
DSW1(config)# interface Vlan30
DSW1(config-if)# ip access-group BLOCK_TELNET in
```

**DSW2**（VLAN40, 50を収容・商品企画開発部/マーケティング部。**VLAN60（システム部）には適用しない**）

```
DSW2(config)# interface Vlan40
DSW2(config-if)# ip access-group BLOCK_TELNET in
DSW2(config)# interface Vlan50
DSW2(config-if)# ip access-group BLOCK_TELNET in
! Vlan60には ip access-group を設定しない
```

### パラメータのポイント
| 項目 | 値 |
|---|---|
| 拒否対象 | `tcp any any eq 23`（Telnet、宛先を問わず） |
| 許可 | `permit ip any any`（ACL末尾は暗黙のdeny allなので明示的に許可文を置く） |
| 適用方向 | `in`（クライアントセグメントからDSWへ入ってくる方向＝トラフィックの発生源側で早期に遮断） |
| 適用除外VLAN | VLAN60（システム部）のみ |

### 補足
- ACLは上から順に評価され、最初にマッチした行で処理が確定する。`deny tcp any any eq 23` を先に置き、その後に `permit ip any any` を置く順序を誤ると意図通りに動作しないため注意。
- `in` 方向で適用することで、クライアントから見て最初にトラフィックを処理するDSWの入口でフィルタが効き、無駄な内部転送を避けられる。
