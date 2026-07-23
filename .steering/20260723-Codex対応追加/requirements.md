# 要求内容

## 概要

Claude 用に整備されているリポジトリ指示と作業記録ワークフローを、Codex の公式な配置と仕様に合わせて追加する。作業記録は既存の `.steering/` を Claude と Codex で共有する。

## 背景

- 現在は `CLAUDE.md` と `.claude/skills/steering/` のみが存在し、Codex が自動検出する `AGENTS.md` と `.agents/skills/` がない。
- 作業記録をエージェント別に分けると、進捗と判断履歴が分散する。
- 現行ルールは全タスク完了を強制しており、ユーザー判断待ちや環境制約による正当な中断を表現しにくい。

## 今回の作業内容

### 1. Codex 用リポジトリ指示の追加
- ルートに `AGENTS.md` を追加する。
- プロジェクト構造、参照資料、成果物、検証、作業記録の運用を Codex に伝える。

### 2. Codex 用 steering skill の追加
- `.agents/skills/steering/` に `SKILL.md` と UI メタデータを追加する。
- 作業計画、実施、進捗更新、中断、再開、振り返りの手順を定義する。

### 3. テンプレートの共通化
- `.steering/templates/` をテンプレートの正本にする。
- Claude と Codex の両 skill から共通テンプレートを参照する。

## 受け入れ条件

- [x] Codex がルートの `AGENTS.md` をリポジトリ指示として利用できる
- [x] Codex が `steering` skill を明示・暗黙に選択できる
- [x] 両エージェントが同じ `.steering/` とテンプレートを利用する
- [x] tasklist の逐次更新と、理由を明記した中断・再開が定義されている
- [x] Codex skill の構造検証が成功する
- [x] 既存の未コミット作業内容が変更されていない

## スコープ外

- `.codex/config.toml` の追加
- `.claude/settings.local.json` の Codex 向け移植
- 既存の config 作成作業の継続や内容修正
- プラグイン化やユーザー全体への skill インストール

## 参照ドキュメント

- `CLAUDE.md`
- `.claude/skills/steering/SKILL.md`
- OpenAI Codex の AGENTS.md・skills 公式仕様
