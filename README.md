# Maieutics

*「正しい問いが、正しい答えを引き出す」*

[English version](README.en.md)

## なぜ「問い」が最も重要なのか

ソフトウェア開発における失敗の多くは、コードの品質ではなく「何を作るべきか」の理解不足から生まれます。要件を聞いたエージェントがすぐにコードを書き始める — これは一見効率的に見えますが、問いかけなしに書かれたコードは、的を外した解決策になりがちです。

**Maieutics は「問いかけ」にフォーカスした Claude Code 向けスキルライブラリです。** プロダクト、セキュリティ、保守性、UX、アーキテクチャの5つの視点から問いを投げかけ、対話を通じてものごとの本質を明らかにしてから、はじめてコードを書き始めます。

## ソクラテスと産婆術

> 「私は何も教えない。ただ問いかけるだけだ。」 — ソクラテス

紀元前5世紀のアテナイ。ソクラテスは市場や体育場で人々に問いかけ続けた哲学者でした。彼は自らを「知恵の産婆」と呼びました。産婆が赤子を取り上げるように、自分は問いかけによって相手の中に眠る知恵を引き出すのだ、と。

ソクラテスの手法は独特でした。彼は答えを与えません。「勇気とは何か？」と問われれば「あなたはどう思いますか？」と返す。相手が答えると「では、こういう場合はどうですか？」とさらに問う。こうして対話を重ねるうちに、相手は自分の思い込みに気づき、より深い理解に到達します。

プラトンの対話篇『テアイテトス』で、ソクラテスはこう語ります — 「私の産婆術のもっとも重要な点は、若者の心が生み出そうとしているものが虚像であるか真実であるかを見分ける力にある」。正しい問いは、曖昧な思考を鋭くし、見落としを浮かび上がらせ、本質を照らし出します。

**Maieutics**（マイユーティクス）という名前は、このソクラテスの産婆術に由来します。AIエージェントにも同じ哲学を持たせたい — まず問いかけ、本質を見極め、それからはじめて手を動かす。このライブラリはその思想を実装したものです。

## 特徴

- **マルチ視点の問いかけ**: プロダクト、セキュリティ、保守性、UX、アーキテクチャの5つの視点から `codex exec` で問いをバッチ生成 — 見落としがちな観点を構造的に洗い出す
- **Discovery Log**: 対話の記録をすべて永続ログに残し、以降のすべてのステージの信頼できる情報源とする — 問いかけの成果を失わない
- **Codex によるレビュー**: 設計・計画・実装のレビューを `codex exec` で実行 — 独立したモデルが客観的に評価する
- **レビューループ**: Critical または Important な問題がなくなるまでレビューを繰り返す — 妥協しない

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
