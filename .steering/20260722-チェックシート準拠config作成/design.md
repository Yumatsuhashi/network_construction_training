# 設計書

## アプローチ概要

`④チェックシート.xlsx` のカテゴリ構造（②〜⑦）をそのままディレクトリ構造に写し取り、「カテゴリ × 機器」でコンフィグを分割生成する。各コンフィグの技術的な根拠は `設計書_md/`（パラメータの正）と `設問/`（コマンドの正）の2つに置き、両者と矛盾しない値を生成する。最後にカテゴリ別configを機器単位で結合して統合running-configを作る。

分割方式の利点：チェックシートの「L2 → L3 → IGP → セキュリティ → 管理 → EGP」という段階に沿って、設定・確認・トラブルシュートを一段ずつ進められる。

## カテゴリ → 設定内容 → 対象機器 → 設問対応

| カテゴリ(フォルダ) | 主な設定内容 | 対象機器 | 根拠となる設問 |
|---|---|---|---|
| ②初期設定・L2項目 | ホスト名 / VLAN定義 / アクセスポート / トランク(802.1q) / 管理VLAN99+IP / 未使用ポートshutdown / STP(RapidPVST負荷分散) / PortFast / EtherChannel(LACP) | ASW1,ASW2,DSW1,DSW2,SVSW | 01, 04, 05, 08 |
| ③L３項目 | SVI+IP / ループバック / 機器間/30 L3リンク / HSRP(クライアントGW・サーバGW) / DHCPサーバ(DSW) | DSW1,DSW2,CSW1,CSW2,R1,R2 | 06, 07 (+08のL3ポートチャネル) |
| ④IGPルーティング | OSPF(エリア0/1設計, MD5認証, router-id手動, passive-interface, P2Pネットワークタイプ) | DSW1,DSW2,CSW1,CSW2,R1,R2 | 02, 03 |
| ⑤セキュリティ | DHCPスヌーピング / ポートセキュリティ / BPDUフィルタ / Telnet制御ACL | ASW1,ASW2,DSW1,DSW2 | 09, 10, 11, 12 |
| ⑥管理項目 | NTP+タイムスタンプ / SNMP(コミュニティ・Trap) / Syslog / VTY・enableパスワード | ASW1,ASW2,DSW1,DSW2,CSW1,CSW2,R1,R2,SVSW | 13, 14, 15, 16 |
| ⑦EGPルーティング | BGP(eBGP対IP-VPN / iBGPフルメッシュ / 集約10.1.0.0/16 / local-pref・AS-path prepend経路調整) | DSW1,DSW2,CSW1,CSW2,R1,R2 | 17, 18, 19, 20, 21, 22 |

## 対象・変更点（カテゴリ別の設計要点）

### ② 初期設定・L2項目
**内容**:
- ホスト名、VLAN 10/20/30/40/50/60（クライアント）、100（サーバ）、99（管理）を定義。
- ASW: FE0/1〜0/6 を部署別アクセスVLAN、FE0/7〜0/8 を DSW接続トランク(802.1q)。管理VLAN99にIP付与（ASW1=10.1.99.1、ASW2=10.1.99.2、/29）。未使用ポートshutdown。
- DSW: ASW接続・DSW間接続トランク。DSW間(Gi0/7-0/8)はL2 EtherChannel(LACP)。
- SVSW: 全ポートVLAN100アクセス、CSW接続もVLAN100（L2）。
- STP: RapidPVST(rapid-pvst)。VLANごとに root を DSW1/DSW2 に振り分けて負荷分散（設問04準拠）。クライアントポートは PortFast（設問05準拠）。

**実施の要点**:
- 論理設計②「DSW1=VLAN10,20,30 / DSW2=VLAN40,50,60 のDHCPサーバ」と整合するよう、STPのroot配置（=トラフィックをDHCPサーバ側に寄せる）を設問04の指定に合わせる。
- 管理VLAN99は /29（論理設計⑤）。ASWのSVIは管理用のみ。

### ③ L３項目
**内容**:
- DSW: 各クライアントVLANのSVIにIP（.253=DSW1 / .254=DSW2）。HSRPでVIP .200（設問06）。ループバック（DSW1=10.1.254.1, DSW2=10.1.254.2）。DSW-CSW間 /30 L3リンク。DHCPプール(.101-.199, DSW1=VLAN10/20/30, DSW2=VLAN40/50/60)。
- CSW: 機器間/30（対DSW・対CSW・対R）。サーバセグメント側 HSRP（CSW1=Active、VIP 10.1.100.200、設問07）。ループバック(CSW1=10.1.254.3, CSW2=10.1.254.4)。
- R: CSW接続/30、IP-VPN側アドレス（R1=192.168.11.2, R2=192.168.12.2）、ループバック(R1=10.1.254.5, R2=10.1.254.6)。

**実施の要点**:
- HSRPの条件を設問06/07どおりに：ダウンタイム短縮（hello/hold短縮）、preempt有効、VLANごとにActiveを分散、DSWはCSW1,2両断時のみ切替（トラッキング設計）。
- IPアドレスはすべて論理設計③〜⑦の表と一致させる。

### ④ IGPルーティング
**内容**:
- OSPF：DSW/CSW/R間をエリア0、DSW配下（クライアント/サーバ側SVI）をエリア1（論理設計⑧）。
- MD5認証（パスワード cisco123）。router-id手動（DSW1=1.1.1.1 … R2=6.6.6.6）。
- クライアント側へアップデートを流さない passive-interface（設問02）。/30リンクは point-to-point ネットワークタイプでDR/BDR排除（設問03）。

**実施の要点**:
- ループバック /32 と機器間 /30、クライアント/サーバセグメントを漏れなく OSPF に載せる。
- エリア境界（DSW）の area 割り当てを設計書どおりに。

### ⑤ セキュリティ
**内容**:
- DHCPスヌーピング：ASWで実施、DSW向けトランクを trust（設問09）。
- ポートセキュリティ：クライアントアクセスポート（設問10）。
- BPDUフィルタ：クライアントポート（設問11）。
- Telnet制御ACL：クライアントからのTelnetを禁止・他は許可、システム部(VLAN60)は全許可、DSWで適用（設問12）。

**実施の要点**:
- PortFast有効ポートとBPDUフィルタ/ポートセキュリティの対象ポートを一致させる。
- ACLはDSWのどのインターフェイス/方向に適用するかを設問12に合わせる。

### ⑥ 管理項目
**内容**:
- NTP：server 10.1.100.1、log/debugのタイムスタンプ有効（設問13）。
- SNMP：community TSHOOT RW、trap先 10.1.100.1、Loopback保有機器はsource=Loopback（設問14）。
- Syslog：logging host 10.1.100.1、Loopback保有機器はsource=Loopback（設問15）。
- VTY/enable：enable secret cisco123、VTYパスワード cisco123（設問16）。

**実施の要点**:
- 全機器共通で投入。Loopbackを持つのはDSW/CSW/R（source-interface指定あり）、ASW/SVSWは持たない。
- 監視サーバ(10.1.100.1)は設定先ではなく宛先。

### ⑦ EGPルーティング
**内容**:
- eBGP：R1(AS65001)↔192.168.11.1(AS64999)、R2(AS65001)↔192.168.12.1(AS64999)、認証 cisco123（設問17,18）。
- iBGP：拠点内フルメッシュ、ループバックでピア（DSW/CSW/R）、拠点内は認証不要（設問19）。192.168ルートはDSW/CSWのBGPテーブルに載せない。
- 集約：IP-VPNへ 10.1.0.0/16 をアドバタイズ（設問20、2方式）。
- 経路調整：10.2.0.0/16→R2経由、10.3.0.0/16→R1経由、戻りはR1（local-pref=設問21、AS-path prepend=設問22）。

**実施の要点**:
- iBGPフルメッシュのピア構成（loopback source, next-hop-self等）を設問19に沿って。
- 集約とfilterで192.168が内部に漏れないようにする。

## 作業の流れ

### カテゴリ1つ分の生成フロー
```
1. 対応する 設問/ ファイルと 設計書_md/ の該当節を読む
2. 対象機器ごとに .txt を作成（そのカテゴリの設定コマンドのみ）
3. 確認コマンド.txt を作成（機器ごとの show + 確認ポイント）
4. 設計書のIP表・パラメータと値を突き合わせ
```

## 検証方法

### 整合性確認
- IPアドレス：`設計書_md/03_論理設計.md` の各表（③〜⑦）と全アドレスを突き合わせ。
- OSPF：エリア・router-id を `03_論理設計.md 論理設計⑧` と突き合わせ。
- HSRP：条件・VIPを `04_耐障害性.md` と設問06/07で突き合わせ。
- BGP：AS番号・ネイバー・集約・経路調整を `08_拠点間接続設計.md` と設問17〜22で突き合わせ。
- コマンド：各カテゴリの 設問/ 回答と投入コマンドを突き合わせ。

### 動作確認（実投入時の参考）
- 各 `確認コマンド.txt` の show コマンドで、設計どおりの状態（VLAN/trunk, SVI/HSRP, OSPFネイバー・ルート, スヌーピング, NTP同期, BGPネイバー・集約）を確認する。

## ディレクトリ構造

```
config/
├── ②初期設定・L2項目/
│   ├── ASW1.txt  ASW2.txt  DSW1.txt  DSW2.txt  SVSW.txt
│   └── 確認コマンド.txt
├── ③L３項目/
│   ├── DSW1.txt  DSW2.txt  CSW1.txt  CSW2.txt  R1.txt  R2.txt
│   └── 確認コマンド.txt
├── ④IGPルーティング/
│   ├── DSW1.txt  DSW2.txt  CSW1.txt  CSW2.txt  R1.txt  R2.txt
│   └── 確認コマンド.txt
├── ⑤セキュリティ/
│   ├── ASW1.txt  ASW2.txt  DSW1.txt  DSW2.txt
│   └── 確認コマンド.txt
├── ⑥管理項目/
│   ├── ASW1.txt  ASW2.txt  DSW1.txt  DSW2.txt  CSW1.txt  CSW2.txt  R1.txt  R2.txt  SVSW.txt
│   └── 確認コマンド.txt
├── ⑦EGPルーティング/
│   ├── DSW1.txt  DSW2.txt  CSW1.txt  CSW2.txt  R1.txt  R2.txt
│   └── 確認コマンド.txt
└── _統合/
    └── ASW1.txt  ASW2.txt  DSW1.txt  DSW2.txt  CSW1.txt  CSW2.txt  R1.txt  R2.txt  SVSW.txt
```

## 実施の順序

1. config/ 配下に6カテゴリフォルダ + _統合フォルダを作成
2. ②→③→④→⑤→⑥→⑦ の順にカテゴリ別config + 確認コマンドを作成（レイヤの依存順）
3. 各機器のカテゴリ別configを結合して _統合/ に統合config生成
4. 設計書・設問との整合性を最終確認
5. 振り返り記録

## 補足・注意点・要検討事項

- **EtherChannel(設問08)のL2/L3切り分け**：可用性設計は「DSW間・DSW-CSW間」をEtherChannel対象とするが、論理設計⑥ではDSW-CSW間が個別の /30 L3リンクになっている。② では DSW間のL2 EtherChannel を、DSW-CSW間のポートチャネル（L3化する場合）は ③ 側で扱う方針とし、②着手時に設問08の回答を読んで最終確定する。
- パスワードは演習指定の cisco123 を使用（OSPF/BGP認証・enable/VTY）。演習用のため平文管理でよいが、実機では要注意。
- 監視サーバ(10.1.100.1)はNTP/Syslog/SNMPの宛先であり設定対象外。

## 将来の拡張性

- 拠点増設時は、同じカテゴリ構造のフォルダを拠点別に追加すれば横展開できる。
- DSWの空きポート予約（物理設計④）に合わせ、ASW増設時のトランク設定をテンプレ化しておくと拡張が容易。
