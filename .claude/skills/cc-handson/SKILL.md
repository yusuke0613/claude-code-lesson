---
name: cc-handson
description: Claude Code公式ドキュメントと関連記事からハンズオン形式の習得用Markdownを生成する。既存ファイルがあれば差分判定し、内容に実質的な変更がない場合は再生成しない。トピックを引数で指定（例：skills, mcp, hooks）。
argument-hint: [topic]
context: fork
agent: general-purpose
allowed-tools: WebFetch Read Write Bash(mkdir *) Bash(date *) Bash(curl *) Bash(ls *) Bash(sha256sum *) Bash(test *)
---

# Claude Code ハンズオン生成（差分対応版）

ユーザーが指定したトピック `$ARGUMENTS` について、ハンズオンMarkdownを**冪等に**生成する。

## 実行時に取得する最新コンテキスト

### 公式ドキュメント全INDEX

!`curl -s https://code.claude.com/docs/llms.txt | head -200`

### 既存ファイルの有無

!`test -f ./handson/cc-handson-$ARGUMENTS.md && echo "EXISTS" || echo "NEW"`

### 既存ファイルのソースハッシュ（あれば）

!`grep -E "^source_hash:" ./handson/cc-handson-$ARGUMENTS.md 2>/dev/null || echo "source_hash: none"`

### 生成日

!`date +%Y-%m-%d`

## 出力先（固定・日付なし）

`./handson/cc-handson-$ARGUMENTS.md`

**ファイル名に日付は入れない**。日付は frontmatter の `last_updated` で管理する。

## 手順

### 1. 既存ファイルの読み込み

上の "EXISTS" 判定が `EXISTS` の場合、`./handson/cc-handson-$ARGUMENTS.md` を Read する。
冒頭の frontmatter から以下を取得：

- `last_updated`
- `source_hash`
- `source_urls`

### 2. 公式ドキュメント＋関連記事の取得

`llms.txt` から該当URLを抽出し WebFetch。
`${CLAUDE_SKILL_DIR}/references/sources.md` の関連記事も WebFetch。

### 3. ソースハッシュの計算

取得した公式ドキュメント本文（HTMLタグ・ナビゲーション・フッターを除いた本文のみ）を連結し、
`sha256sum` で短縮ハッシュ（先頭12文字）を計算。

これを `new_source_hash` とする。

### 4. 差分判定（重複防止の核心）

| 既存ファイル | source_hash 比較 | アクション                               |
| ------------ | ---------------- | ---------------------------------------- |
| 無い (NEW)   | —                | **新規生成**                             |
| 有る         | 既存 == new      | **スキップ**（"変更なし"を報告して終了） |
| 有る         | 既存 != new      | **更新生成**（更新履歴を追記）           |

スキップ時の出力例：

> 📄 `cc-handson-skills.md` は既に最新です（前回更新: 2026-05-01）。公式ドキュメントに変更が検出されませんでした。

### 5. 生成 or 更新

**新規生成の場合**：
`${CLAUDE_SKILL_DIR}/references/output-template.md` のテンプレでフル生成。
frontmatter に以下を必ず含める：

## \`\`\`yaml

title: "Claude Code ハンズオン：<topic>"
topic: <topic>
last_updated: <今日の日付>
source_hash: <new_source_hash>
source_urls:

- <fetchしたURL1>
- <fetchしたURL2>
  revision: 1

---

\`\`\`

**更新生成の場合**：

- 本文を最新内容で書き直す
- `last_updated` を更新
- `source_hash` を `new_source_hash` に更新
- `revision` を +1
- 本文末尾の `## 更新履歴` セクションに1行追記：
  `- YYYY-MM-DD: 公式ドキュメントの変更を反映（revision N→N+1）。主な変更点：<差分の要約3点>`

### 6. 品質基準（前回と同じ）

- コピペで動くコマンド
- 各セクションに ✅ 確認ポイント
- 演習3つ（基礎/応用/実戦）
- 参照リンクを末尾に列挙
- 公式と記事情報を分離

## 出力後

ファイルパス、アクション種別（NEW / UPDATED / SKIPPED）、source_hash、revision を表示して終了。
