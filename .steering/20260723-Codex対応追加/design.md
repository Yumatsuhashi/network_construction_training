# 設計書

## アプローチ概要

Codex の恒久指示は `AGENTS.md`、再利用ワークフローは `.agents/skills/steering/` に配置する。記録とテンプレートはエージェント固有ディレクトリから分離し、`.steering/` 配下を共通の正本にする。

## 対象・変更点

### Codex リポジトリ指示

- `CLAUDE.md` と同じプロジェクト知識を Codex 向けに整理する。
- steering を使う作業と、記録を作らない軽微な作業の境界を定義する。
- 既存の未完了記録は依頼内容が一致する場合だけ再開する。

### Steering skill

- skill 名は `steering` とする。
- 複数工程の config 作成、設計検討、設問回答、トラブルシュート、作業再開をトリガーに含める。
- `tasklist.md` を正式な進捗源とし、Codex の計画表示は補助扱いにする。
- ユーザー判断待ち、外部機材不足、権限・環境制約、明示的中断では、未完了項目と再開点を記録して終了できる。

### 共通テンプレート

- requirements、design、tasklist の3種類を `.steering/templates/` に置く。
- Claude skill の旧テンプレート参照を共通パスへ変更し、重複ファイルを削除する。

## 検証方法

- `skill-creator` の `quick_validate.py` で skill を検証する。
- 検索により旧テンプレート参照が残っていないことを確認する。
- `git diff` と `git status` で今回の変更範囲を確認する。
- 作業開始前から存在する未コミットファイルの内容差分が増えていないことを確認する。

## 実施の順序

1. 今回の作業記録を作成
2. 共通テンプレートを追加
3. Codex steering skill を初期化・編集
4. `AGENTS.md` を追加
5. Claude skill のテンプレート参照と運用ルールを調整
6. 検証と振り返りを実施
