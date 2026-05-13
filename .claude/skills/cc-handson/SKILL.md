---
name: cc-handson
description: Claude Code公式ドキュメントと関連記事からハンズオン形式の習得用Markdownを生成する。トピックを引数で指定（例：skills, mcp, hooks, subagents, plugins）。コピペで実行できるコマンド・手順・トラブルシューティング・参照リンクを必ず含める。
argument-hint: [topic]
context: fork
agent: general-purpose
allowed-tools: WebFetch Read Write Bash(mkdir *) Bash(date *) Bash(curl *)
---

# Claude Code ハンズオン生成

ユーザーが指定したトピック `$ARGUMENTS` について、Claude Codeを習得するためのハンズオンMarkdownを生成する。

## 実行時に取得する最新コンテキスト

### 公式ドキュメント全INDEX（最新）

!`curl -s https://code.claude.com/docs/llms.txt | head -200`

### 生成日

!`date +%Y-%m-%d`

## 手順

### 1. トピックに対応するURLを決定

上記の `llms.txt` から `$ARGUMENTS` に関連するページのURLを抽出する。
同時に `${CLAUDE_SKILL_DIR}/references/sources.md` を Read して、補強用の日本語記事URLも取得する。

### 2. 公式ドキュメントの取得（必須）

抽出したURL（`code.claude.com/docs/en/...`）を WebFetch で取得。最低でも以下を抽出：

- 概要・できること
- 最小構成のセットアップ手順
- frontmatter / 設定項目のリファレンス
- 公式コード例（そのまま動くもの）
- トラブルシューティング

### 3. 関連記事の取得（補強）

sources.md にある zenn / qiita / Anthropic engineering blog などから2〜3本を WebFetch。
公式に書いていない**実戦的なTips**、**ハマりどころ**、**応用パターン**を拾う。

### 4. テンプレートで整形

`${CLAUDE_SKILL_DIR}/references/output-template.md` を Read し、その構成に従って出力を組み立てる。

### 5. 品質基準（必ず守る）

- すべてのコマンド・コードは**そのままコピペで動く形**にする（プレースホルダは `<>` で明示）
- 各セクション末尾に「✅ 確認ポイント」を入れる
- セットアップは**最小→拡張**の順
- 「やってみよう」演習を最低3つ含める（基礎/応用/実戦）
- 参照リンクは末尾の「## 参照」セクションにURL付きで列挙
- 公式情報と記事情報は明確に分ける

### 6. ファイル出力

出力先：`./handson/cc-handson-$ARGUMENTS-<日付>.md`
Bash で `mkdir -p ./handson` した後、Write で保存する。

## 出力後

生成したファイルのパスを伝え、冒頭サマリ（3行）を表示して終了。
