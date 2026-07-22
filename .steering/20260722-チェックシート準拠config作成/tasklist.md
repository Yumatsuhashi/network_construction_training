# タスクリスト

## 🚨 タスク完全完了の原則

**このファイルの全タスクが完了するまで作業を継続すること**

### 必須ルール
- **全てのタスクを`[x]`にすること**
- 「時間の都合により別タスクとして実施予定」は禁止
- 「複雑すぎるため後回し」は禁止
- 未完了タスク（`[ ]`）を残したまま作業を終了しない

### タスクスキップが許可される唯一のケース
- 要件・設計の変更により、そのタスク自体が不要になった
- 機材・環境上の制約により、タスクが実行不可能になった

スキップ時は必ず理由を明記:
```markdown
- [x] ~~タスク名~~（要件変更により不要: 具体的な理由）
```

---

## フェーズ0: 準備

- [x] `config/` 配下にカテゴリフォルダを作成
  - [x] `②初期設定・L2項目/`
  - [x] `③L３項目/`
  - [x] `④IGPルーティング/`
  - [x] `⑤セキュリティ/`
  - [x] `⑥管理項目/`
  - [x] `⑦EGPルーティング/`
  - [x] `_統合/`

## フェーズ1: ②初期設定・L2項目

- [x] 根拠読み込み（設問01,04,05,08 / 設計書02物理設計・03論理設計②）
- [x] EtherChannelのL2/L3切り分けを設問08で確定（design.md補足の要検討事項）
  - 確定: DSW間=L2 EtherChannel(②) / DSW-CSW・CSW-CSW=L3ルーテッドEtherChannel(③)
- [x] 機器別config作成
  - [x] ASW1.txt（ホスト名/VLAN/アクセス/トランク/管理VLAN99 IP/PortFast/未使用shut/STP）
  - [x] ASW2.txt
  - [x] DSW1.txt（VLAN/トランク/DSW間EtherChannel/STP root配置）
  - [x] DSW2.txt
  - [x] SVSW.txt（VLAN100アクセス/CSW接続）
- [x] 確認コマンド.txt（show vlan / show interfaces trunk / show etherchannel / show spanning-tree 等）

## フェーズ2: ③L３項目

- [x] 根拠読み込み（設問06,07 / 設計書03論理設計③〜⑦・04耐障害性）
- [x] Po番号を設問06/08に統一（Po1=CSW1向, Po2=CSW2向, Po3=DSW間）。②のDSW L2 EtherChannelをPo3に修正
- [x] 機器別config作成
  - [x] DSW1.txt（SVI+IP/ループバック/DSW-CSW間/30/HSRP/DHCPプール VLAN10,20,30）
  - [x] DSW2.txt（同上、DHCPプール VLAN40,50,60）
  - [x] CSW1.txt（機器間/30/ループバック/サーバGW HSRP Active/VLAN100下地）
  - [x] CSW2.txt
  - [x] R1.txt（CSW接続/30/IP-VPN側IP/ループバック）
  - [x] R2.txt
- [x] 確認コマンド.txt（show ip interface brief / show standby / show ip dhcp binding 等）

## フェーズ3: ④IGPルーティング

- [x] 根拠読み込み（設問02,03 / 設計書03論理設計⑧⑨）
- [x] 整合性確認: Po3(DSW間)はL2のためOSPF隣接なし→DSW間はCSW経由で経路交換（設問02のPo3例は本設計では非該当と各txtに明記）
- [x] 機器別config作成
  - [x] DSW1.txt（OSPF area0/1境界/MD5/router-id 1.1.1.1/passive/P2P）
  - [x] DSW2.txt（router-id 2.2.2.2）
  - [x] CSW1.txt（router-id 3.3.3.3）
  - [x] CSW2.txt（router-id 4.4.4.4）
  - [x] R1.txt（router-id 5.5.5.5）
  - [x] R2.txt（router-id 6.6.6.6）
- [x] 確認コマンド.txt（show ip ospf neighbor / show ip route ospf / show ip ospf interface 等）

### ④整合メモ
- エリア0=機器間/30(Po1,Po2,Po3,Fa0/6,Gi0/0)+全ループバック+サーバVLAN100。エリア1=クライアントVLAN10-60,管理VLAN99。DSWがABR。
- MD5: `area 0 authentication message-digest` + 各エリア0隣接IFに `ip ospf message-digest-key 1 md5 cisco123`。Vlan100/Lo0はpassiveのため鍵不要。
- /30は全て `ip ospf network point-to-point`。両端一致必須。
- R Gi0/1(IP-VPN 192.168.x)はOSPF非対象（⑦のeBGP側）。ループバックはiBGP(⑦)到達性のためOSPFで広告。

## フェーズ4: ⑤セキュリティ

- [ ] 根拠読み込み（設問09,10,11,12 / 設計書06セキュリティ）
- [ ] 機器別config作成
  - [ ] ASW1.txt（DHCPスヌーピング/ポートセキュリティ/BPDUフィルタ）
  - [ ] ASW2.txt
  - [ ] DSW1.txt（Telnet制御ACL）
  - [ ] DSW2.txt
- [ ] 確認コマンド.txt（show ip dhcp snooping / show port-security / show access-lists 等）

## フェーズ5: ⑥管理項目

- [ ] 根拠読み込み（設問13,14,15,16 / 設計書07管理）
- [ ] 機器別config作成
  - [ ] ASW1.txt（NTP/SNMP/Syslog/VTY・enable）
  - [ ] ASW2.txt
  - [ ] DSW1.txt（+ Loopback source指定）
  - [ ] DSW2.txt
  - [ ] CSW1.txt
  - [ ] CSW2.txt
  - [ ] R1.txt
  - [ ] R2.txt
  - [ ] SVSW.txt
- [ ] 確認コマンド.txt（show ntp status / show snmp / show logging / show run | include 等）

## フェーズ6: ⑦EGPルーティング

- [ ] 根拠読み込み（設問17,18,19,20,21,22 / 設計書08拠点間接続設計）
- [ ] 機器別config作成
  - [ ] R1.txt（eBGP/iBGP/集約/local-pref/AS-path prepend）
  - [ ] R2.txt
  - [ ] CSW1.txt（iBGPフルメッシュ参加）
  - [ ] CSW2.txt
  - [ ] DSW1.txt（iBGPフルメッシュ参加）
  - [ ] DSW2.txt
- [ ] 確認コマンド.txt（show ip bgp summary / show ip bgp / show ip route bgp 等）

## フェーズ7: 統合config生成

- [ ] 各機器のカテゴリ別configを結合して `_統合/` に作成
  - [ ] ASW1.txt（②⑤⑥）
  - [ ] ASW2.txt（②⑤⑥）
  - [ ] DSW1.txt（②③④⑤⑥⑦）
  - [ ] DSW2.txt（②③④⑤⑥⑦）
  - [ ] CSW1.txt（③④⑥⑦）
  - [ ] CSW2.txt（③④⑥⑦）
  - [ ] R1.txt（③④⑥⑦）
  - [ ] R2.txt（③④⑥⑦）
  - [ ] SVSW.txt（②⑥）

## フェーズ8: 検証

- [ ] 要件・設計書との整合性を確認
  - [ ] IPアドレスを `03_論理設計.md` の表と全機器突き合わせ
  - [ ] OSPFエリア・router-id を突き合わせ
  - [ ] HSRP条件・VIP を `04_耐障害性.md`＋設問06,07 と突き合わせ
  - [ ] BGP(AS/ネイバー/集約/経路調整) を `08_拠点間接続設計.md`＋設問17-22 と突き合わせ
  - [ ] 各カテゴリのコマンドを対応する 設問/ 回答と突き合わせ
- [ ] 統合configとカテゴリ別configの内容一致を確認

## フェーズ9: ドキュメント更新

- [ ] 作業で判明した設計書の訂正・追記があれば `設計書_md/` を更新
- [ ] 実装後の振り返り（このファイル下部に記録）

---

## 実装後の振り返り

### 実施完了日
{YYYY-MM-DD}

### 計画と実績の差分

**計画と異なった点**:
-

**新たに必要になったタスク**:
-

**理由があってスキップしたタスク**（該当する場合のみ）:
-

### 学んだこと

**技術的な学び**:
-

**プロセス上の改善点**:
-

### 次回への改善提案
-
