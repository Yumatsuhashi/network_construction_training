# 設問：その他ネットワーク設計④（出典：設計書_md/05_可用性.md）

> 出典：①ネットワーク要件・詳細設計書(TC).pptx スライド42

## 該当する要件
- DSW間、DSW-CSW間のリンクはトラフィックが集中するため、**物理リンクを最大の帯域で使用できるようにする**
- **標準化されているプロトコルを使用する**
- DSW間のリンクは**送信元、宛先MACアドレスに基づいて負荷分散**させる仕様とする

## 【設問】
> どのような技術を使用しますか？　その技術に関わるパラメータは、具体的にどのような値にしますか？（コマンドでも説明でも可）

## 回答

### 使用する技術：EtherChannel（LACP：IEEE 802.3ad）

物理構成図の通り、DSW1-CSW1（Po1）、DSW2-CSW2（Po1）、DSW1-CSW2／DSW2-CSW1（Po2）、CSW1-CSW2（Po3）、DSW1-DSW2（Po3）は、いずれも複数物理リンクを束ねたEtherChannelとして構成されている。

- 「物理リンクを最大帯域で使用」＝複数リンクを論理的に1本の太い帯域として扱う **EtherChannel（リンクアグリゲーション）**
- 「標準化されているプロトコル」＝EtherChannelのネゴシエーションプロトコルには **LACP（IEEE 802.3ad、標準規格）** と **PAgP（Cisco独自）** の2種類があるが、要件が「標準化」を明示しているためLACPを選択する
- 「送信元・宛先MACアドレスに基づく負荷分散」＝EtherChannelの束内でのフレーム振り分けアルゴリズムとして `src-dst-mac` を指定する

### 設定コマンド例（DSW1 Po1の場合）

```
! 負荷分散アルゴリズムはスイッチ単位のグローバル設定
DSW1(config)# port-channel load-balance src-dst-mac

! メンバーポートにLACPを設定
DSW1(config)# interface range GigabitEthernet1/0/1 - 2
DSW1(config-if-range)# channel-protocol lacp
DSW1(config-if-range)# channel-group 1 mode active
```

対向のCSW1側も同様に設定する。

```
CSW1(config)# port-channel load-balance src-dst-mac
CSW1(config)# interface range FastEthernet0/1 - 2
CSW1(config-if-range)# channel-protocol lacp
CSW1(config-if-range)# channel-group 1 mode active
```

同様の設定をPo2（DSW1-CSW2 / DSW2-CSW1）、Po3（CSW1-CSW2、DSW1-DSW2）の各メンバーポートにも適用する。

### 補足
- `channel-group mode active` は自ら能動的にLACPネゴシエーションを開始するモード。対向を `passive` にしても成立するが、両端が `passive` だとネゴシエーションが開始されず折り合わないため、少なくとも一方は `active` にする必要がある（本設計では両端 `active` を推奨）。
- `port-channel load-balance` は **スイッチ全体に対するグローバル設定**であり、Port-channelインターフェースごとに個別のアルゴリズムを指定するものではない点に注意（プラットフォームによっては例外あり）。
