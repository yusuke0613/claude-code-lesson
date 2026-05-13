# Claude Code ハンズオン：Skills（カスタムスキル）

> 生成日：2026-05-14 / 対象：Claude Code 最新版（Opus 4.7 / Sonnet 4.6 / Haiku 4.5 対応）
> 推奨読了時間：20分 / 所要ハンズオン時間：45〜60分

## 0. このハンズオンで身につくこと
- `SKILL.md` を1ファイル書くだけでカスタムスキルを作れるようになる
- 個人 / プロジェクト / プラグインの3スコープを使い分けて配布できる
- frontmatter（`description` / `allowed-tools` / `disable-model-invocation` / `context: fork` など）を実用レベルで設定できる
- `!`bash`` による動的コンテキスト注入と `references/`・`scripts/` を使った Progressive Disclosure を実装できる
- スキルが「発火しない／発火しすぎる」典型トラブルを自力で切り分けられる

## 1. 前提
- Claude Code がインストール済み（`claude --version` で確認）
- Git リポジトリで作業できる環境（ハンズオン後半で利用）
- Python 3 または Node.js（応用パートで `scripts/` を試す場合）

セットアップ確認：

```bash
# Claude Code のバージョン確認
claude --version

# 個人スキル用ディレクトリの作成（既にあればスキップされる）
mkdir -p ~/.claude/skills

# 動作確認用のテストプロジェクト
mkdir -p ~/cc-skills-handson && cd ~/cc-skills-handson
git init -q && echo "# demo" > README.md && git add . && git commit -qm "init"
```

✅ 確認ポイント：`claude --version` がバージョン文字列を返し、`~/.claude/skills` ディレクトリが存在すること。

---

## 2. 5分でわかる概要

**Skills とは何か。** `SKILL.md` というMarkdownファイル1枚で、Claude Code に新しいコマンドや専門知識を追加できる仕組み。`description` を読んだ Claude が「今このスキルを使うべきだ」と判断すると自動でロードし、ユーザーは `/skill-name` で明示的に呼び出すこともできる。

**CLAUDE.md との違い。** CLAUDE.md は**常時コンテキストに載る**ので、内容が増えるとトークン代がかかり続ける。SKILL.md は**呼び出されるまで本体がロードされない**（`description` のみ常駐）。よって「長い手順書」や「分量の多いリファレンス」は Skills に切り出すのが定石。これを **Progressive Disclosure（段階的開示）** と呼ぶ。

**bundled skills と custom skills。** Claude Code には `/simplify`、`/debug`、`/loop`、`/claude-api` などのバンドル済みスキルが最初から入っている。ユーザーが追加するのが custom skills で、本ハンズオンの主題。

**配置場所で適用範囲が決まる。** `~/.claude/skills/`（個人・全プロジェクト適用） / `.claude/skills/`（プロジェクト固有・gitで共有可） / プラグイン内 / Enterprise managed の4スコープ。同名なら Enterprise > Personal > Project の順で優先。

**Custom commands は Skills に統合済み。** 以前の `.claude/commands/deploy.md` は引き続き動くが、新規は `.claude/skills/deploy/SKILL.md` 形式が推奨。supporting files やサブエージェント実行などの拡張機能が使える。

---

## 3. ハンズオン

### 3-1. 基礎：5分で最小のスキルを動かす

git の未コミット差分を要約するスキル `/summarize-changes` を作る。公式ドキュメントの最初のチュートリアルそのまま。

```bash
# 1. スキルディレクトリを作成
mkdir -p ~/.claude/skills/summarize-changes

# 2. SKILL.md を書き込む
cat > ~/.claude/skills/summarize-changes/SKILL.md <<'EOF'
---
description: Summarizes uncommitted changes and flags anything risky. Use when the user asks what changed, wants a commit message, or asks to review their diff.
---

## Current changes

!`git diff HEAD`

## Instructions

Summarize the changes above in two or three bullet points, then list any risks you notice such as missing error handling, hardcoded values, or tests that need updating. If the diff is empty, say there are no uncommitted changes.
EOF
```

動作確認：

```bash
cd ~/cc-skills-handson
echo "console.log('hello')" > app.js  # 適当な変更を加える
claude
```

Claude のプロンプトで以下のどちらかを試す：

```text
What did I change?
```

または：

```text
/summarize-changes
```

✅ 確認ポイント：3行以内の差分サマリと、リスク（テスト未整備など）の指摘が返ってくる。`/` メニューに `summarize-changes` が候補表示される。

**今起きていること：** ``!`git diff HEAD` `` は Claude が見る前にシェルで実行され、その出力で置き換えられる（**動的コンテキスト注入**）。つまり Claude は「コマンド」ではなく「実際の diff」を受け取る。

---

### 3-2. 応用：frontmatter と supporting files を使いこなす

PR レビュー用スキル `/review-pr` を作る。**ポイントは3つ：**
1. `disable-model-invocation: true` で「ユーザーが明示的に呼んだ時だけ」発火させる
2. `allowed-tools` で `gh` コマンドを事前承認し、毎回の権限ダイアログを消す
3. `references/checklist.md` に詳細チェックリストを切り出し、`SKILL.md` 本体を短く保つ（500行以下が目安）

```bash
mkdir -p ~/.claude/skills/review-pr/references
```

`~/.claude/skills/review-pr/SKILL.md`：

```yaml
---
description: Review a GitHub pull request with our team's checklist. Use when the user explicitly asks to review a PR.
disable-model-invocation: true
allowed-tools: Bash(gh pr *) Bash(gh api *)
argument-hint: [pr-number]
---

# PR review: #$ARGUMENTS

## Live PR context
- Diff: !`gh pr diff $ARGUMENTS`
- Comments: !`gh pr view $ARGUMENTS --comments`
- Files changed: !`gh pr diff $ARGUMENTS --name-only`

## Instructions

1. Read the checklist in [references/checklist.md](references/checklist.md).
2. Walk through each item against the diff above.
3. Output: a verdict (approve / request-changes / comment), then specific findings with file:line references.
4. Skip checklist items that don't apply rather than padding output.
```

`~/.claude/skills/review-pr/references/checklist.md`：

```markdown
# PR Review Checklist

## Correctness
- [ ] Edge cases handled (null, empty, boundary values)
- [ ] Error paths tested
- [ ] Race conditions considered for concurrent code

## Security
- [ ] No secrets in code or logs
- [ ] User input validated at boundaries
- [ ] SQL/shell injection vectors closed

## Style
- [ ] Matches existing conventions
- [ ] No premature abstractions
- [ ] Comments explain WHY, not WHAT
```

実行：

```text
/review-pr 42
```

✅ 確認ポイント：
- `gh` コマンドの実行で権限ダイアログが出ない（`allowed-tools` が効いている）
- 「PR を見て」と曖昧に頼んでも Claude が勝手にこのスキルを発火しない（`disable-model-invocation: true` の効果）
- レスポンスに `references/checklist.md` の各項目が反映されている

**Progressive Disclosure の意味：** `SKILL.md` 本体は短く、詳細は `references/` に置く。Claude は必要なときだけ `references/` を読みに行くので、スキル一覧の常駐コストを抑えつつ、いざ使うときには深い知識を引き出せる。

---

### 3-3. 実戦：プロジェクト固有スキル＋サブエージェント実行

自分のリポジトリに「コミットメッセージ生成」スキルを置き、チームで共有する。さらに `context: fork` で**サブエージェント側で実行**してメインの会話コンテキストを汚さないようにする。

```bash
cd ~/cc-skills-handson
mkdir -p .claude/skills/draft-commit
```

`.claude/skills/draft-commit/SKILL.md`：

```yaml
---
description: Draft a Conventional Commits-style message from the staged diff. Use when the user asks for a commit message.
context: fork
agent: Explore
allowed-tools: Bash(git diff *) Bash(git status *) Bash(git log *)
---

## Staged diff
!`git diff --staged`

## Recent commit style
!`git log -10 --pretty=format:'%s'`

## Task

Produce a single Conventional Commits message for the staged changes.

- Type: feat / fix / refactor / docs / test / chore
- Scope: optional, derived from the most-changed directory
- Subject: imperative, under 72 chars
- Body: only if the change spans multiple files or has a non-obvious motivation

Match the casing and style of the recent commits shown above. Output the message and nothing else.
```

動作確認：

```bash
echo "TODO" >> README.md
git add README.md
claude
# プロンプトで:
# /draft-commit
```

✅ 確認ポイント：
- 出力が Conventional Commits 形式（例：`docs: add TODO marker to README`）
- サブエージェント側で実行されるため、メイン会話の文脈に diff の長文が混じらない
- `.claude/skills/` を git に commit すればチーム全員で共有できる

**演習（基礎）：** `/summarize-changes` の `description` から "risky" の単語を消すと自動発火しにくくなる。description のキーワードが trigger に効くことを体感する。

**演習（応用）：** `/review-pr` の `references/` に `output-format.md` を追加し、Markdown テーブルでの出力フォーマットを定義してみる。`SKILL.md` 本体から `[references/output-format.md](references/output-format.md)` で参照する。

**演習（実戦）：** 自分のプロジェクトでよく使う手順（デプロイ、リリースノート生成、テスト生成など）を1つ選んで `.claude/skills/` 配下に作り、PR を出してチーム共有する。`disable-model-invocation: true` を付けるかどうかを判断基準とともに考える。

---

## 4. リファレンス早見表

### frontmatter 主要項目

| 項目 | 必須 | 値の例 | 説明 |
|---|---|---|---|
| `name` | No | `my-skill` | 省略時はディレクトリ名。小文字・数字・ハイフンのみ、64文字以内 |
| `description` | 推奨 | `Reviews a PR with our checklist. Use when...` | Claude が自動発火を判断する材料。最重要 |
| `when_to_use` | No | `Trigger phrases: "review pr", "check this PR"` | description の補強。合算1,536字でカット |
| `argument-hint` | No | `[pr-number]` | `/` 補完で表示されるヒント |
| `arguments` | No | `[issue, branch]` | 名前付き引数。`$issue` `$branch` で参照 |
| `disable-model-invocation` | No | `true` | Claude の自動発火を禁止（手動 `/name` のみ） |
| `user-invocable` | No | `false` | `/` メニューから隠す（背景知識用） |
| `allowed-tools` | No | `Bash(gh *) Read Grep` | このスキル有効中は権限プロンプトをスキップ |
| `model` | No | `claude-opus-4-7` / `inherit` | このスキル実行中だけモデル切替 |
| `effort` | No | `low` / `medium` / `high` / `xhigh` / `max` | 思考努力レベル |
| `context` | No | `fork` | サブエージェントで実行 |
| `agent` | No | `Explore` / `Plan` / `general-purpose` | `context: fork` 時に使うエージェント種別 |
| `paths` | No | `src/**/*.ts` | このパス配下を扱う時だけ自動ロード |
| `hooks` | No | （hooks 設定） | このスキルのライフサイクルに紐づく hooks |

### 配置場所と優先順位

| スコープ | パス | 適用範囲 |
|---|---|---|
| Enterprise | managed settings 経由 | 組織全体（最優先） |
| Personal | `~/.claude/skills/<name>/SKILL.md` | 自分の全プロジェクト |
| Project | `.claude/skills/<name>/SKILL.md` | このプロジェクトのみ |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | plugin が有効な範囲 |

同名スキルの優先順位：**Enterprise > Personal > Project**。Plugin は `plugin-name:skill-name` の namespace を持つので衝突しない。

### 文字列置換

| 変数 | 内容 |
|---|---|
| `$ARGUMENTS` | 引数全体の文字列 |
| `$ARGUMENTS[0]` / `$0` | 0始まりインデックスで個別アクセス |
| `$<name>` | `arguments:` で宣言した名前で参照 |
| `${CLAUDE_SESSION_ID}` | 現在のセッションID |
| `${CLAUDE_EFFORT}` | 現在の effort レベル |
| `${CLAUDE_SKILL_DIR}` | このスキル自身のディレクトリ絶対パス（scripts 参照に必須） |

### 動的コンテキスト注入

```markdown
- インライン:   !`git status --short`
- 複数行ブロック:
    ```!
    node --version
    npm --version
    ```
```

これらは Claude が見る前にシェルで実行され、出力で置き換えられる。`settings.json` の `"disableSkillShellExecution": true` で無効化可能（managed settings での組織制御向け）。

---

## 5. よくあるハマりどころ

| 症状 | 原因 | 解決 |
|---|---|---|
| スキルが発火しない | `description` にユーザーが実際に使う単語が無い | `description` に「レビュー / review / check」など自然な言い回しを並べる。`/doctor` で description budget の溢れも確認 |
| スキルが過剰に発火する | `description` が広すぎる | より具体的に書く。それでも嫌なら `disable-model-invocation: true` で手動専用化 |
| `/` メニューに出てこない | `user-invocable: false` になっている、またはディレクトリ作成後にセッション再起動していない | frontmatter を確認し、トップレベルの skills ディレクトリを新規作成した直後は `claude` を再起動 |
| description が途中で切れる | スキル数が多くて budget を超過 | `/doctor` で確認 → `settings.json` の `skillListingBudgetFraction` を `0.02` に上げる、または使わないスキルを `skillOverrides: { "<name>": "name-only" }` で縮退 |
| `${CLAUDE_SKILL_DIR}` が空になる | bash 変数として参照している（`$CLAUDE_SKILL_DIR`） | `${CLAUDE_SKILL_DIR}` のブレース必須形式で書く（プレースホルダ置換のため） |
| `!`cmd`` が実行されない | settings で `disableSkillShellExecution: true` | プレースホルダが `[shell command execution disabled by policy]` で置換されている。設定を見直す |
| frontmatter が読まれない | `---` の前に空行・BOM・YAMLの構文ミス | ファイル先頭が `---\n` で始まっているか確認 |
| プロジェクトスキルが他人の環境で動かない | `.claude/skills/` を gitignore している、または workspace trust 未承認 | `.gitignore` を確認。承認ダイアログで Trust を選んで `allowed-tools` を有効化 |

### コミュニティの知見（記事から）

- **description にトリガー語を複数並べる**：「レビューして」「コードチェック」「確認して」など、ユーザーが実際に発する言葉を3〜5個入れると発火精度が上がる（[Qiita: dai_chi氏](https://qiita.com/dai_chi/items/a061382e0616fa76fb32) より）
- **`SKILL.md` は 500行以下に保つ**：手順は本体、判断基準は `references/criteria.md`、フォーマットは `references/format.md` と分割する
- **`scripts/` を呼ぶときは構造化JSONを返す**：`{ "status": "ok|not_found|error", "data": ... }` を返すと Claude が分岐しやすい（[Zenn: acntechjp](https://zenn.dev/acntechjp/articles/554765e5a0d0be) より）
- **`feedback.md` 運用**：使った後の気づきを `references/feedback.md` に貯めて継続改善するパターンが定着しつつある

---

## 6. 次に学ぶこと

- [Subagents](https://code.claude.com/docs/en/sub-agents) — `context: fork` の挙動を本質的に理解し、専用エージェントに preload する設計へ
- [Hooks](https://code.claude.com/docs/en/hooks-guide) — スキルのライフサイクルイベントで決定的処理を差し込む
- [Plugins](https://code.claude.com/docs/en/plugins) — 自作スキル群をパッケージ化して配布
- [Permissions](https://code.claude.com/docs/en/permissions) — `allowed-tools` の細粒度制御と deny rules
- [Memory (CLAUDE.md)](https://code.claude.com/docs/en/memory) — Skills と CLAUDE.md の役割分担

---

## 7. 参照

### 公式ドキュメント
- [Extend Claude with skills](https://code.claude.com/docs/en/skills) — 本ハンズオンのメインソース。frontmatter リファレンスと公式チュートリアル全文
- [How to create custom skills (support article)](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills) — スキルパッケージ化と配布の基本
- [anthropics/skills (GitHub)](https://github.com/anthropics/skills) — 公式サンプルスキル集と仕様
- [Subagents](https://code.claude.com/docs/en/sub-agents) — `context: fork` 詳細
- [Settings](https://code.claude.com/docs/en/settings) — `skillOverrides` / `skillListingBudgetFraction` / `disableSkillShellExecution`
- [Permissions](https://code.claude.com/docs/en/permissions) — `Skill(name)` 形式の許可ルール

### 関連記事（コミュニティの知見）
- [Claude CodeのSkillsを作成例から徹底理解する (Zenn / acntechjp)](https://zenn.dev/acntechjp/articles/554765e5a0d0be) — Progressive Disclosure と構造化JSON出力、PowerShell環境のハマりどころ
- [Claude Code Agent Skills 実践ガイド (Qiita / dai_chi)](https://qiita.com/dai_chi/items/a061382e0616fa76fb32) — description最適化、references/ 分割、feedback.md 運用
- [SKILL.mdでClaude Codeのワークフローを自動化する実践ガイド (Qiita / nogataka)](https://qiita.com/nogataka/items/c59defafd0dfb88c4a90) — superpowers リポジトリの設計思想
- [Claude Codeのスキル機能完全ガイド (Zenn / ino_h)](https://zenn.dev/ino_h/articles/2025-10-23-claude-code-skills-guide) — モジュール型機能拡張のオーバービュー
- [Claude Code Skillsの作り方 (Zenn / tmasuyama1114)](https://zenn.dev/tmasuyama1114/books/claude_code_basic/viewer/skills-creation) — references フォルダ実用例
