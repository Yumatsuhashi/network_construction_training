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

## 📌 次回作業の再開ポイント（2026-07-22 中断）

**現在地**: ②③④⑤⑥⑦ のカテゴリ別config・確認コマンドは作成完了。フェーズ7（統合config）・8（検証）は未着手。

**次回はまず ④〜⑦ レビューの指摘への対応判断から始める**（下記3件をユーザーと相談 → 決定後に反映 → その後フェーズ7へ）。

### 保留中のレビュー指摘（要判断）

- [ ] **指摘1〔推奨・動作に直結〕BGPに `no synchronization` / `no auto-summary` を追加するか**
  - 理由: 本設計はOSPF↔BGPを相互redistributeしない。古いIOS（12.2メインライン等、同期デフォルトON）ではiBGP学習の拠点間経路(10.2.0.0/16, 10.3.0.0/16)がOSPFに無いためRIBに載らず、クライアントが拠点2/3へ到達不能になる恐れ。
  - 元pptxは2012-2013作成のため機材が古いIOSの可能性あり。15.x系ではデフォルトOFFで無害。
  - 対応する場合: `config/⑦EGPルーティング/` の全6ファイルの `router bgp 65001` 直下に `no synchronization` と `no auto-summary` を追記。
- [ ] **指摘2〔軽微〕VTY回線を `line vty 0 4` のままにするか `0 15` に広げるか**
  - 現状は設問16準拠で 0-4 のみ。5-15はlogin/password未設定で使用不可（=安全）。全回線を明示的に塞ぐなら `⑥管理項目/` 全機の `line vty 0 4` を `0 15` に変更。
- [ ] **指摘3〔確認のみ〕設問と意図的に変えた3点をこの方針で確定してよいか**
  - ④ Po3(DSW間)はL2トランクのためOSPF隣接なし（設問02の例と相違。CSW経由で経路交換）
  - ⑤ Telnet ACLを両DSWの Vlan10,20,30,40,50 に適用（設問12の機器分けと相違。HSRPフェイルオーバー対応）
  - ⑦ next-hop-self付与・集約は方式1(network+Null0)（redistributeしない設計の必然。設問に無い補完）

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

- [x] 根拠読み込み（設問09,10,11,12 / 設計書06セキュリティ）
- [x] 機器別config作成
  - [x] ASW1.txt（DHCPスヌーピング/ポートセキュリティ/BPDUフィルタ）
  - [x] ASW2.txt
  - [x] DSW1.txt（DHCPスヌーピング/Telnet制御ACL）
  - [x] DSW2.txt
- [x] 確認コマンド.txt（show ip dhcp snooping / show port-security / show access-lists 等）

### ⑤整合メモ
- **Telnet ACL【設問12から意図的変更】**: 設問12はDSW1=VLAN10-30/DSW2=VLAN40-50の機器分けだが、本設計はHSRPで両DSWが全クライアントVLANのSVIを保持→フェイルオーバー後も遮断が効くよう両DSWのVlan10,20,30,40,50にin適用。VLAN60(システム部)除外。各txtに明記。
- **DHCP Snooping**: ASWはtrust=FE0/7-8（設問例のGi1/0/5-6は本設計のポートに読替）。DSWもDHCPサーバのため有効化しtrust=Po3。ASW+DSWのDHCPサーバ混在構成でoption82による正規配布破棄を防ぐため`no ip dhcp snooping information option`を全対象機に付与。
- Port Security/BPDU Filterの対象ポート(FE0/1-6)は②のPortFast対象と一致。

## フェーズ5: ⑥管理項目

- [x] 根拠読み込み（設問13,14,15,16 / 設計書07管理）
- [x] 機器別config作成
  - [x] ASW1.txt（NTP/SNMP/Syslog/VTY・enable、Loopback非保有）
  - [x] ASW2.txt（Loopback非保有）
  - [x] DSW1.txt（+ Loopback0 source指定）
  - [x] DSW2.txt
  - [x] CSW1.txt
  - [x] CSW2.txt
  - [x] R1.txt
  - [x] R2.txt
  - [x] SVSW.txt（Loopback非保有）
- [x] 確認コマンド.txt（show ntp status / show snmp / show logging / show run | include 等）

### ⑥整合メモ
- 全9機器共通: `ntp server 10.1.100.1` + `service timestamps log/debug datetime msec localtime` / `snmp-server community TSHOOT RW`+`enable traps`+`host 10.1.100.1 version 2c TSHOOT` / `logging host 10.1.100.1` / `enable secret cisco123` + `line vty 0 4` password+login。
- Loopback保有(DSW/CSW/R)のみ `snmp-server trap-source Loopback0` + `logging source-interface Loopback0`。ASW/SVSWは付けない（設問14,15）。
- 監視サーバ10.1.100.1への到達は③④の疎通が前提。clock timezoneは未設定（必要ならJST 9を追加）。
- ⑤のBLOCK_TELNET ACLはVLAN60除外のため管理Telnetはシステム部から可能（整合確認済み）。

## フェーズ6: ⑦EGPルーティング

- [x] 根拠読み込み（設問17,18,19,20,21,22 / 設計書08拠点間接続設計）
- [x] 機器別config作成
  - [x] R1.txt（eBGP/iBGP/集約/local-pref拠点3/next-hop-self）
  - [x] R2.txt（eBGP/iBGP/集約/local-pref拠点2/AS-path prepend/next-hop-self）
  - [x] CSW1.txt（iBGPフルメッシュ参加）
  - [x] CSW2.txt
  - [x] DSW1.txt（iBGPフルメッシュ参加）
  - [x] DSW2.txt
- [x] 確認コマンド.txt（show ip bgp summary / show ip bgp / show ip route bgp 等）

### ⑦整合メモ（重要な設計補完2件）
- **next-hop-self【設問に無いが必須】**: R1/R2のIP-VPN側リンク(192.168.x/30)は④でOSPF非対象のため、eBGP学習経路のnext-hop(192.168.x.1)を内部で解決不可。R1/R2の全iBGPネイバーに`next-hop-self`を付与し、Lo0(OSPF到達可)をnext-hopに書換。これが無いとCSW/DSWで10.2/10.3が無効経路になる。
- **集約方式【設問20】**: OSPFをBGPへredistributeしない設計では`aggregate-address`は元経路がBGP表に無く生成されない。動作する方式1(`network 10.1.0.0 mask 255.255.0.0`+`ip route 10.1.0.0 255.255.0.0 Null0`)を採用。方式2は注記のみ。
- iBGPフルメッシュ=6ノード(DSW1,DSW2,CSW1,CSW2,R1,R2)×Loopback、認証なし。ピアアドレスは③のLo0(10.1.254.1〜.6)と一致。
- 経路調整: R1=拠点3宛LocalPref200(in)、R2=拠点2宛LocalPref200(in)+戻りAS-path prepend(out)。192.168はBGP表に出さない(設問19条件)。

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
