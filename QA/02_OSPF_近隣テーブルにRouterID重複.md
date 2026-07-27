# QA②：OSPF 近隣テーブルに同じ Router ID が2つ出る

関連: 設計書 `03_論理設計.md`（OSPF / アドレス設計）

---

## Q. `CSW1# show ip ospf neighbor` で `4.4.4.4` が2行あるのはなぜ？

実際の出力例:
```
Neighbor ID   Pri  State       Dead Time  Address       Interface
5.5.5.5        0   FULL/  -    00:00:37   10.1.255.21   FastEthernet0/6
4.4.4.4        1   FULL/DR     00:00:33   10.1.100.254  FastEthernet0/5
4.4.4.4        0   FULL/  -    00:00:31   10.1.255.10   Port-channel3
2.2.2.2        0   FULL/  -    00:00:30   10.1.255.14   Port-channel2
```

**A. `4.4.4.4` は CSW2 の Router ID。OSPFの近隣テーブルは「ルータ単位」ではなく「隣接(アジャセンシー)単位＝リンク/インターフェイス単位」で1行になる。CSW1とCSW2は2本の別セグメントで繋がっているので、同じCSW2に対して隣接が2つでき、2行出る（正常）。**

| Address | Interface | セグメント | ネットワークタイプ | State |
|---|---|---|---|---|
| 10.1.100.254 | Fa0/5 | サーバセグメント 10.1.100.0/24 | broadcast（DR/BDR選出あり） | `FULL/DR`（CSW2がDR） |
| 10.1.255.10 | Po3 | CSW1-CSW2相互接続 10.1.255.8/30 | point-to-point（DR無し） | `FULL/-`（Pri 0） |

### 見分けのポイント
- **Neighbor ID**（4.4.4.4）＝ どのルータか（Router IDなので同一機器なら同じ値）
- **Address / Interface** ＝ どのリンク上の隣接か（ここが違えば別エントリ）
- **State の `/DR` `/-`**：
  - `FULL/DR` … サーバセグメントはbroadcastなのでDR/BDRを選出し、対向CSW2がDRになっている
  - `FULL/-` … Po3は設計⑨「/30はDR/BDR選出を無くす」に従いpoint-to-point化、DRの概念が無い（`Pri 0`と整合）

### 参考（他の行）
- `5.5.5.5`=R1（10.1.255.20/30, Fa0/6）
- `2.2.2.2`=DSW2（10.1.255.12/30, Po2）
- ※この出力に `1.1.1.1`（DSW1）が無い場合、CSW1–DSW1（10.1.255.0/30）の隣接が張れているか別途確認する価値あり。

**Router ID 一覧**（設計書03）: DSW1=1.1.1.1 / DSW2=2.2.2.2 / CSW1=3.3.3.3 / CSW2=4.4.4.4 / R1=5.5.5.5 / R2=6.6.6.6
