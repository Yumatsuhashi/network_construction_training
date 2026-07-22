# 設問：その他ネットワーク設計⑨ ②（出典：設計書_md/07_管理.md）

> 出典：①ネットワーク要件・詳細設計書(TC).pptx スライド49

## 該当する要件
すべてのデバイスに以下の管理機能を設定する
- VTYパスワード、特権パスワード（enable secret）
  - パスワード：cisco123

## 【設問】
> 設定するコマンドを示してください。

## 回答

### 特権モードパスワード（enable secret）

```
Device(config)# enable secret cisco123
```

`enable password`（平文相当・脆弱なハッシュ）ではなく、より強固に暗号化される **`enable secret`** を明示的に使う点がポイント（要件文でも「enable secret」と明記されている）。

### VTY（Telnet/SSHでのリモートログイン）パスワード

```
Device(config)# line vty 0 4
Device(config-line)# password cisco123
Device(config-line)# login
```

- `line vty 0 4` … 5本の仮想端末回線（同時リモートログインセッション数）全てに適用
- `password cisco123` … ログイン時に要求するパスワード
- `login` … パスワード認証を要求する状態にする（`login` を入れないとパスワードチェック自体が働かない）

### 補足
- ここでは特定のユーザー名を使わない「ラインパスワード方式」のログインとしている（AAA/ローカルユーザー認証を使う場合は `login local` ＋ `username` の設定になるが、それは別設問「オプション設定」で扱う内容であり、本設問はシンプルなVTYパスワードのみを求めている）。
- `enable secret` を設定すると、たとえ `enable password` が別途設定されていても `enable secret` が優先される。
