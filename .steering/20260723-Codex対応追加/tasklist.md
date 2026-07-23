# タスクリスト

## 運用ルール

- 完了したタスクは、その完了直後に `[x]` へ更新する。
- Codex のセッション内タスク表示は補助であり、このファイルを正式な進捗記録とする。
- ユーザー判断待ち、外部機材不足、権限・環境制約、ユーザーによる明示的中断では未完了を許可する。その場合は「中断・再開情報」に現在地、理由、再開手順を残す。
- 不要になったタスクは、理由を明記して取り消し線付きで完了扱いにする。

## フェーズ1: 作業記録

- [x] 今回の requirements.md、design.md、tasklist.md を作成

## フェーズ2: 共通テンプレート

- [x] `.steering/templates/` に3種類のテンプレートを追加
- [x] 正当な中断・再開を表現できる tasklist テンプレートへ調整

## フェーズ3: Codex 対応

- [x] `.agents/skills/steering/` を公式の skill 初期化手順で作成
- [x] Codex 用 `SKILL.md` を実装
- [x] `agents/openai.yaml` を整備
- [x] ルートに `AGENTS.md` を追加

## フェーズ4: Claude 側の共通化

- [x] Claude skill のテンプレート参照を `.steering/templates/` へ変更
- [x] Claude skill の中断・再開ルールを共通方針に合わせる
- [x] 旧テンプレートの重複ファイルを削除

## フェーズ5: 検証

- [x] Codex skill の構造検証を実行（`quick_validate.py`: `Skill is valid!`）
- [x] 旧テンプレート参照が残っていないことを確認（`rg` で該当なし）
- [x] `AGENTS.md`、両 skill、共通テンプレートの整合性を確認
- [x] 既存の未コミット作業が変更されていないことを確認（開始時からの旧 tasklist と config 差分は対象外として保持）
- [x] 実装後の振り返りを記録

---

## 中断・再開情報

- 状態: 完了
- 現在地: 全フェーズと受け入れ条件の確認を完了
- 中断理由: なし
- 再開手順: なし

## 実装後の振り返り

### 実施完了日

2026-07-23

### 計画と実績の差分

- 計画どおり `AGENTS.md`、Codex skill、共通テンプレートを追加し、Claude skill も共通パスへ切り替えた。
- skill 初期化時、`short_description` が24文字で最小要件の25文字に届かなかった。説明を修正し、公式ジェネレーターで `agents/openai.yaml` を生成した。
- 作業中に今回の対象外である `講義メモ.txt` の変更が現れたため、既存の config／旧 tasklist と同様に変更せず保持した。

### 学んだこと

- Codex のリポジトリ固有指示は `AGENTS.md`、リポジトリ固有 skill は `.agents/skills/`、共有する作業データは中立な `.steering/` に分けると責務が明確になる。
- `quick_validate.py` により skill の frontmatter と命名を機械的に検証できる。

### 次回への改善提案

- skill 初期化前に UI メタデータの文字数制約を確認する。
- テンプレート変更時は `.steering/templates/` のみを正本として更新し、エージェント別コピーを再作成しない。
