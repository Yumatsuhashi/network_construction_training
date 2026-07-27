# QA（質問と回答のまとめ）

設計・コンフィグを進める中で出てきた「なぜこうするのか」という疑問と、その回答をトピック別にまとめたもの。
`設問/`（演習で与えられた設問）とは別に、対話の中で生じた確認事項を記録する。

## 索引

| ファイル | 主な内容 |
|---|---|
| [01_セキュリティ_VACL・DHCPスヌーピング・VLAN60.md](01_セキュリティ_VACL・DHCPスヌーピング・VLAN60.md) | Telnet制御をVACL化する理由 / DHCP SnoopingはDSWに要るか / VACLの送信元とVLAN60の扱い |
| [02_OSPF_近隣テーブルにRouterID重複.md](02_OSPF_近隣テーブルにRouterID重複.md) | `show ip ospf neighbor` で同じRouter IDが2つ出る理由 |
| [03_管理_SNMPバージョン・VTYパスワード.md](03_管理_SNMPバージョン・VTYパスワード.md) | SNMP version 2cにする理由（v1/v3でもよいか） / VTYパスワードとTelnet可否 |
| [04_EGP_BGP_設計の疑問点.md](04_EGP_BGP_設計の疑問点.md) | route-map末尾のpermit / Null0静的 / next-hop-self / aggregate-address / 集約の根拠 / ピアグループ / next-hop属性 / 再配送との違い |

> 出典表記の「設計書」は `設計書_md/`、「設問」は `設問/`、「config」は `config/` を指す。
