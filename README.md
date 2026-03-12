# Maieutics

*ソクラテスの産婆術「マイユーティクス」にちなんで — 規律ある問いかけを通じて真理に到達する手法。*

[English version](README.en.md)

Maieutics は Claude Code 向けのマルチ視点設計・実装ワークフローです。エージェントがいきなりコードを書き始めるのではなく、まず複数の視点から適切な質問を行い、正しいものを作ることを保証します。

## 特徴

- **マルチ視点の質問生成**: プロダクト、セキュリティ、保守性、UX、アーキテクチャの5つの視点から `codex exec` 経由で質問をバッチ生成（メインエージェントが1つずつ考えるのではなく）
- **Discovery Log**: ユーザーの回答をすべて永続ログに記録し、以降のすべてのステージの信頼できる情報源とする
- **Codex によるレビュー**: 設計レビュー、計画レビュー、実装レビューを `codex exec` で実行 — 独立したモデルによる客観的な評価
- **レビューループ**: Critical または Important な問題がなくなるまでレビューを繰り返す

## 前提条件

- [Claude Code](https://claude.com/claude-code)
- [Codex CLI](https://github.com/openai/codex) — 質問生成とレビューに必要

## インストール

### Claude Code プラグイン

```bash
# リポジトリをクローン
git clone https://github.com/rarilurelo/maieutics.git ~/.claude/plugins/maieutics
```

### 手動（プロジェクトローカル）

```bash
# skills をプロジェクトにコピー
cp -r path/to/maieutics/skills/ .claude/skills/
```

## ワークフロー

```
brainstorming → writing-plans → subagent-driven-development → finishing-a-development-branch
```

1. **brainstorming** — `codex exec` で複数視点からバッチ質問を生成。すべての回答を Discovery Log に記録。承認済みの Design Doc を作成。

2. **writing-plans** — 設計から小さな実装タスク（各2〜5分）を作成。`codex exec` でレビューし、修正ループを実行。TDD、DRY、YAGNI の原則に従う。

3. **subagent-driven-development** — タスクごとに新しい Claude Code サブエージェントを起動。タスクごとに2段階レビュー（仕様準拠 + コード品質）。最終的に `codex exec` でマルチ視点の実装レビューを実施。

4. **finishing-a-development-branch** — テストを検証し、オプションを提示（マージ/PR/保持/破棄）、クリーンアップ。

## カスタマイズ

### 視点設定

プロジェクトルートに `.maieutics/multi-perspective.json` を作成して、視点、質問バッチサイズ、レビュー設定をカスタマイズできます。デフォルト設定は `skills/brainstorming/references/multi-perspective.default.json` を参照してください。

## Superpowers との共存

Maieutics は `maieutics:` ネームスペースを使用し、`superpowers` プラグインと共存できます。両プラグインともセッション開始時にエントリポイントスキルを注入します。タスクに合ったワークフローを選んでください。

## スキル一覧

| スキル | 用途 |
|---|---|
| `brainstorming` | マルチ視点の質問生成、Discovery Log、Design Doc |
| `writing-plans` | Codex レビューループ付き実装計画 |
| `subagent-driven-development` | タスクごとのサブエージェント実行とレビューゲート |
| `executing-plans` | 人間のチェックポイント付き別セッション計画実行 |
| `using-git-worktrees` | 隔離されたワークスペースのセットアップ |
| `finishing-a-development-branch` | ブランチ完了ワークフロー |

## ライセンス

MIT License — 詳細は [LICENSE](LICENSE) を参照。
