# Maieutics

*「正しい問いが、正しい答えを引き出す」*

[English version](README.en.md)

## なぜ「問い」が最も重要なのか

ソフトウェア開発における失敗の多くは、コードの品質ではなく「何を作るべきか」の理解不足から生まれます。要件を聞いたエージェントがすぐにコードを書き始める — これは一見効率的に見えますが、問いかけなしに書かれたコードは、的を外した解決策になりがちです。

**Maieutics は「問いかけ」にフォーカスした Claude Code 向けスキルライブラリです。** プロダクト、セキュリティ、保守性、UX、アーキテクチャの5つのレンズから問いを投げかけ、対話を通じてものごとの本質を明らかにしてから、はじめてコードを書き始めます。

## ソクラテスと産婆術

> 「私は何も教えない。ただ問いかけるだけだ。」 — ソクラテス

紀元前5世紀のアテナイ。ソクラテスは市場や体育場で人々に問いかけ続けた哲学者でした。彼は自らを「知恵の産婆」と呼びました。産婆が赤子を取り上げるように、自分は問いかけによって相手の中に眠る知恵を引き出すのだ、と。

ソクラテスの手法は独特でした。彼は答えを与えません。「勇気とは何か？」と問われれば「あなたはどう思いますか？」と返す。相手が答えると「では、こういう場合はどうですか？」とさらに問う。こうして対話を重ねるうちに、相手は自分の思い込みに気づき、より深い理解に到達します。

プラトンの対話篇『テアイテトス』で、ソクラテスはこう語ります — 「私の産婆術のもっとも重要な点は、若者の心が生み出そうとしているものが虚像であるか真実であるかを見分ける力にある」。正しい問いは、曖昧な思考を鋭くし、見落としを浮かび上がらせ、本質を照らし出します。

**Maieutics**（マイユーティクス）という名前は、このソクラテスの産婆術に由来します。AIエージェントにも同じ哲学を持たせたい — まず問いかけ、本質を見極め、それからはじめて手を動かす。このライブラリはその思想を実装したものです。

## 特徴

- **per-lens の問いかけ**: 各レンズを独立した `codex exec` で走らせ、利用側セッションが並列化と集約を担う
- **エビデンス付き質問**: 各レンズは repo を探索し、観測したファイルや設定を根拠に質問を返す
- **Inquiry Record**: 対話の記録に加え、confirmed / working / invalidated assumptions を永続ログとして保持する
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
inquiry → using-git-worktrees → execution-planning → delegated-execution → closing-the-branch
```

1. **inquiry** — `codex exec` をレンズごとに独立実行し、repo 探索に基づく質問を生成する。利用側セッションが結果を集約し、raw question + supplement と assumption state を Inquiry Record に記録する。全 active lens が `meaningful-questions-exhausted` になるまで続ける。

2. **using-git-worktrees** — feature branch 用の隔離された git worktree を作成。プロジェクトの規約に沿って保存場所を選び、セットアップとベースライン検証を行う。

3. **execution-planning** — 設計から小さな実装タスク（各2〜5分）を作成。`codex exec` でレビューし、修正ループを実行。TDD、DRY、YAGNI の原則に従う。

4. **delegated-execution** — タスクごとに新しい Claude Code サブエージェントを起動。タスクごとに2段階レビュー（仕様準拠 + コード品質）。最終的に `codex exec` でレンズベースの実装レビューを実施。

5. **closing-the-branch** — テストを検証し、オプションを提示（マージ/PR/保持/破棄）、クリーンアップ。

## カスタマイズ

### レンズ設定

プロジェクトルートに `.maieutics/lenses.json` を作成して、レンズ、表示ポリシー、レビュー設定をカスタマイズできます。デフォルト設定は `skills/inquiry/references/lenses.default.json` を参照してください。

## スキル一覧

| スキル | 用途 |
|---|---|
| `inquiry` | per-lens 質問生成、Inquiry Record、Design Synthesis |
| `execution-planning` | Codex レビューループ付き実装計画 |
| `delegated-execution` | タスクごとのサブエージェント実行とレビューゲート |
| `guided-execution` | 人間のチェックポイント付き別セッション計画実行 |
| `using-git-worktrees` | 隔離されたワークスペースのセットアップ |
| `closing-the-branch` | ブランチ完了ワークフロー |

## ライセンス

MIT License — 詳細は [LICENSE](LICENSE) を参照。
