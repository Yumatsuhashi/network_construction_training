# 設問：その他ネットワーク設計① ②（出典：設計書_md/04_耐障害性.md）

> 出典：①ネットワーク要件・詳細設計書(TC).pptx スライド38

## 該当する要件
- **クライアントPCが接続されるポートはコンバージェンスを早める仕様とする**

## 【設問】
> どのような技術を使用しますか？　コマンドも示してください。

## 回答

### 使用する技術：PortFast

通常のSTPでは、ポートがリンクアップしてから実際にフォワーディング状態になるまで、Blocking → Listening → Learning → Forwarding という遷移を経るため、Rapid PVST+でも数秒、従来PVSTでは30秒以上かかることがある。クライアントPCが接続されるポートはループを形成しない末端（エッジポート）であることが構成上確定しているため、この遷移プロセス自体が不要である。

**PortFast** を設定すると、リンクアップ後に即座にForwarding状態へ遷移する（DHCPによるIPアドレス取得等の待ち時間が発生しない）。

### 設定コマンド例

```
! ASW1/ASW2のクライアント接続ポートに設定
ASW1(config)# interface FastEthernet0/1
ASW1(config-if)# spanning-tree portfast
```

複数ポートに一括設定する場合：

```
ASW1(config)# interface range FastEthernet0/1 - 6
ASW1(config-if-range)# spanning-tree portfast
```

グローバルでアクセスポート全体に適用する場合：

```
ASW1(config)# spanning-tree portfast default
```
（この場合、トランクポートやアップリンクポートには自動適用されない設定もあるため、意図した範囲か確認すること）

### 補足
- PortFastはあくまで「エッジポートの収束を早める」機能であり、ループ防止機能ではない。誤って別のスイッチをそのポートに接続するとループが発生しうるため、実運用ではBPDU Guard（別設問参照）と組み合わせるのが一般的なベストプラクティスである。
