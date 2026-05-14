---
title: "Claude Code ハンズオン：Subagents（サブエージェント）"
topic: subagents
last_updated: 2026-05-14
source_hash: 64557887cf67
source_urls:
  - https://code.claude.com/docs/en/sub-agents
  - https://code.claude.com/docs/en/agents
  - https://code.claude.com/docs/en/agent-teams
  - https://code.claude.com/docs/en/agent-sdk/subagents
revision: 1
---

# Claude Code ハンズオン：Subagents（サブエージェント）

> 生成日：2026-05-14 / 対象：Claude Code 最新版（v2.1.63 以降 / Opus 4.7・Sonnet 4.6・Haiku 4.5 対応）
> 推奨読了時間：20分 / 所要ハンズオン時間：45〜60分

## 0. このハンズオンで身につくこと
- サブエージェントの目的（**コンテキスト隔離・並列化・専門化・ツール制限**）を1分で説明できる
- `.claude/agents/<name>.md` を1ファイル書くだけでカスタムサブエージェントを作れる
- frontmatter（`description` / `tools` / `model` / `permissionMode` / `isolation` / `memory` など）を実用レベルで設定できる
- 自動委譲・@-mention・`--agent` フラグ・`/agents` UI を使い分けて呼び出せる
- 組み込みの **Explore / Plan / general-purpose** と自作サブエージェントの違いを理解する
- Subagents / Agent view / Agent teams / Worktrees のうち、いつどれを使うべきか判断できる

---

## 1. 前提
- Claude Code がインストール済み（`claude --version` が **v2.1.63 以降** を返すこと。Agent tool への改名が入っているため）
- 動作確認用に Git リポジトリが1つあること
- 応用パートで `gh` コマンド（GitHub CLI）を使うので、未導入なら入れておくと進めやすい

セットアップ確認：

```bash
# 1. Claude Code のバージョン確認（v2.1.63 以降を推奨）
claude --version

# 2. ユーザースコープのサブエージェントディレクトリ
mkdir -p ~/.claude/agents

# 3. 動作確認用プロジェクト
mkdir -p ~/cc-subagents-handson && cd ~/cc-subagents-handson
git init -q
echo "# subagents handson" > README.md
git add . && git commit -qm "init"
mkdir -p .claude/agents
```

✅ 確認ポイント：`~/.claude/agents/` と `./cc-subagents-handson/.claude/agents/` の2つが存在し、`claude --version` がバージョン文字列を返す。

---

## 2. 5分でわかる概要

**サブエージェントとは何か。** サブエージェントは「**自分専用のコンテキストウィンドウ**で動く、特化型のAIアシスタント」。メイン会話を汚さずに、検索結果・ログ・大きな diff・テスト出力など**メインで参照しない大量出力**を別コンテキストで処理させ、サマリだけを返してもらう。同じ種類の作業を繰り返すなら、定義ファイルにして再利用する。

**4つのメリット。** 公式が挙げているのは以下：
1. **Preserve context（文脈を守る）** — 探索や実装でメイン会話のトークンを食いつぶさない
2. **Enforce constraints（制約の強制）** — `tools` で使えるツールを制限できる
3. **Reuse configurations（設定の再利用）** — `~/.claude/agents/` に置けば全プロジェクトで使える
4. **Specialize behavior（専門化）** — 集中したシステムプロンプトでドメイン特化させる
5. **Control costs（コスト制御）** — Haiku などの軽量モデルにルーティングできる

**組み込みサブエージェント。** 何も定義しなくても以下が使える：
- **Explore**：Haikuで動く読み取り専用のコードベース探索エージェント
- **Plan**：plan モード中のリサーチ用、読み取り専用
- **general-purpose**：全ツール利用可能な汎用エージェント。複雑な多段タスク向け
- 補助：`statusline-setup`（Sonnet）、`claude-code-guide`（Haiku）

**Subagents / Agent view / Agent teams / Worktrees の使い分け。** 全部「並列に何かをやる」ための仕組みだが住み分けが明確：

| 何が違うか | Subagents | Agent view | Agent teams | Worktrees |
|---|---|---|---|---|
| 誰が指揮 | メイン会話のClaude | あなた（人間） | 専任の lead Claude | あなた（人間） |
| ワーカー間通信 | なし（lead経由のみ） | なし | あり（mailbox + 共有task list） |  なし |
| ファイル分離 | 任意（`isolation: worktree`） | 自動worktree | 手動で領域分担 | 必須 |
| 起動方法 | Agent tool 経由 / `@-mention` | `claude agents` | 自然言語で「チーム作って」と依頼 | `git worktree` |
| 安定性 | GA | Research preview | **Experimental（既定OFF）** | GA |

**重要な制約。** サブエージェントは**サブエージェントを生成できない**（無限ネスト防止）。ネストしたい場合は Skills か、メイン会話からチェーンで呼ぶ。

---

## 3. ハンズオン

### 3-1. 基礎：5分で最小のサブエージェントを作る

PR や差分のレビュー専用「`code-reviewer`」を作る。読み取り専用ツールしか持たせず、Haikuで動かしてコストを抑える。

```bash
cat > ~/.claude/agents/code-reviewer.md <<'EOF'
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: haiku
---

You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run `git diff` to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is clear and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented
- Good test coverage
- Performance considerations addressed

Provide feedback organized by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)

Include specific examples of how to fix issues.
EOF
```

動作確認：

```bash
cd ~/cc-subagents-handson
echo "function divide(a, b) { return a / b }" > calc.js   # ゼロ除算チェック無し＝レビュー対象
claude
```

Claudeのプロンプトでどれか1つを試す（**段階的に強い指定**になる）：

```text
# パターン1: 自然言語（Claudeが description を見て委譲を判断）
Have the code-reviewer subagent look at my recent changes

# パターン2: @-mention（Claudeに必ずこのエージェントを使わせる）
@"code-reviewer (agent)" review the new calc.js

# パターン3: セッション全体をこのエージェントとして起動
# （別シェルで） claude --agent code-reviewer
```

✅ 確認ポイント：
- メイン会話の transcript に `git diff` の長文が出ない（**サブエージェント側で実行されているから**）
- レビュー結果は **Critical / Warnings / Suggestions** に分かれて返ってくる
- `/agents` を実行して **Library** タブに `code-reviewer` が表示される

**今起きていること：** Claude は `description` を読んで「このタスクは code-reviewer の領域だ」と判断 → **Agent tool**（旧 Task tool）でサブエージェントを起動 → サブエージェントは自分の context window で `git diff` を実行・分析 → **最終メッセージだけ**がメイン会話に戻る。中間の tool_use や long output は破棄される。

---

### 3-2. 応用：プロジェクト固有エージェント＋ツール制限＋永続メモリ

自分のリポジトリに、デバッグ専門エージェント `debugger` を置く。**ポイントは4つ：**
1. `tools:` を **allowlist** で絞る（`Edit` は入れる、修正もさせるため）
2. `disallowedTools:` で個別にブロック（例：`Write` を禁止して新規ファイル作成を防ぐ）
3. `memory: project` で `.claude/agent-memory/debugger/` に学びを蓄積する
4. `isolation: worktree` で**作業を別チェックアウト**に閉じ込め、メインのworking treeを汚さない

`~/cc-subagents-handson/.claude/agents/debugger.md`：

```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues. Trigger phrases - "debug", "bug", "error", "test failure", "stack trace".
tools: Read, Edit, Bash, Grep, Glob
disallowedTools: Write
model: sonnet
memory: project
isolation: worktree
color: red
---

You are an expert debugger specializing in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix (edit existing files only — do not create new ones)
5. Verify solution works

Debugging process:
- Analyze error messages and logs
- Check recent code changes (git log, git diff)
- Form and test hypotheses
- Add strategic debug logging
- Inspect variable states

For each issue, provide:
- Root cause explanation
- Evidence supporting the diagnosis
- Specific code fix
- Testing approach
- Prevention recommendations

Memory directive:
Before starting, read `.claude/agent-memory/debugger/MEMORY.md` if it exists.
After fixing, append a one-line summary of the bug pattern and fix to that
file so future invocations learn from history.

Focus on fixing the underlying issue, not the symptoms.
```

実行：

```bash
cd ~/cc-subagents-handson
# わざと壊す
cat > calc.js <<'EOF'
function divide(a, b) { return a / b }
console.log(divide(10, 0))   // Infinity が出る
console.log(divide(10))      // NaN が出る
EOF

claude
# プロンプトで:
# @"debugger (agent)" calc.js でゼロ除算と引数欠落のバグを直して
```

✅ 確認ポイント：
- 修正は**worktree側**で行われる（`git worktree list` で確認できる。変更があれば worktree が残る／無ければ自動削除）
- 新規ファイル作成は試みても拒否される（`disallowedTools: Write` の効果）
- `.claude/agent-memory/debugger/MEMORY.md` が生成され、学習が蓄積される
- 完了後に `git diff` でメインのworking treeに変更が出ていない（worktree隔離が効いている）

**ツール制限の組み合わせルール：**
- `tools` のみ → **allowlist**（書いたものだけ使える）
- `disallowedTools` のみ → **denylist**（書いたものだけ禁止、それ以外は親から継承）
- 両方 → `disallowedTools` を先に適用 → 残った中から `tools` で再度絞る（両方に書かれたツールは消える）

---

### 3-3. 実戦：3つのサブエージェントを並列で動かす（PRレビュー）

公式の use case「並列コードレビュー」を体験する。`security-reviewer` / `perf-reviewer` / `test-coverage-reviewer` を別々の関心軸で並走させ、Claude にサマリを統合させる。

`.claude/agents/security-reviewer.md`：

```markdown
---
name: security-reviewer
description: Security-focused PR reviewer. Use when reviewing PRs for vulnerabilities, secrets, injection risks, and authn/authz issues.
tools: Read, Grep, Glob, Bash(gh pr *), Bash(gh api *), Bash(git *)
model: sonnet
permissionMode: dontAsk
color: red
---

You are a security reviewer. For the given PR, identify:
- Hardcoded secrets, tokens, credentials
- Injection vectors (SQL, shell, XSS, SSRF)
- Authentication / authorization weaknesses
- Unsafe deserialization or eval-like constructs

Output a markdown table: severity | file:line | issue | suggested fix.
Do not modify any files.
```

`.claude/agents/perf-reviewer.md`：

```markdown
---
name: perf-reviewer
description: Performance-focused PR reviewer. Use when reviewing PRs for hot loops, N+1 queries, missing indexes, blocking I/O, and memory leaks.
tools: Read, Grep, Glob, Bash(gh pr *), Bash(git *)
model: sonnet
color: yellow
---

You are a performance reviewer. For the given PR, identify:
- N+1 queries and missing batch operations
- Hot loops and accidental O(n^2) patterns
- Blocking I/O on the request path
- Unbounded memory growth

Output a markdown table: severity | file:line | issue | suggested fix.
```

`.claude/agents/test-coverage-reviewer.md`：

```markdown
---
name: test-coverage-reviewer
description: Test coverage reviewer. Use when reviewing PRs to verify changed code has corresponding tests covering edge cases.
tools: Read, Grep, Glob, Bash(gh pr *), Bash(git *)
model: haiku
color: green
---

You are a test-coverage reviewer. For each non-test file changed in the PR,
verify there is a corresponding test that exercises the new code path,
including edge cases (null, empty, boundary, error path).

Output: a markdown checklist per changed file with covered / not-covered status.
```

実行：

```bash
cd ~/cc-subagents-handson
claude
# プロンプトで:
# Review PR #42 in parallel using the security-reviewer, perf-reviewer,
# and test-coverage-reviewer subagents. Then synthesize their findings into
# a single approve / request-changes / comment verdict.
```

✅ 確認ポイント：
- `/agents` の **Running** タブに3つのサブエージェントが**同時に**表示される
- メイン会話に戻ってくるのは各エージェントの**最終レポート**だけ（`gh pr diff` の生出力は出ない）
- Claudeが3つのレポートを統合して verdict（approve/request-changes/comment）を出す

**演習（基礎）：** `code-reviewer` の `description` から "Proactively" を消して、自動委譲が働きづらくなることを確認する。`description` の言い回しがトリガーに効くことを体感する。

**演習（応用）：** `db-reader` サブエージェントを作り、`PreToolUse` フックで `INSERT/UPDATE/DELETE/DROP` を含む Bash コマンドを exit code 2 でブロックする。許可は `SELECT` のみ。スクリプト例は **第7章「公式ドキュメント」** の database query validator を参照。

**演習（実戦）：** 自プロジェクトで「テストランナー」「リリースノート起草」「依存アップデートPR要約」のいずれかをサブエージェント化し、`.claude/agents/` に配置して PR を出してチーム共有する。`tools` の allowlist 設計と `model` 選定（Haiku/Sonnet/Opus）の理由をPRに書く。

---

## 4. リファレンス早見表

### frontmatter 全フィールド

| 項目 | 必須 | 値の例 | 説明 |
|---|---|---|---|
| `name` | **Yes** | `code-reviewer` | 小文字とハイフンのみ。ファイル名と一致しなくてよい。`SubagentStart` フックには `agent_type` として渡る |
| `description` | **Yes** | `Expert code reviewer. Use proactively after code changes.` | Claude が自動委譲を判断する文。最重要 |
| `tools` | No | `Read, Grep, Glob, Bash` | allowlist。省略時は親から全継承。`Skill` は書かず `skills:` を使う |
| `disallowedTools` | No | `Write, Edit` | denylist。`tools` より先に適用される |
| `model` | No | `sonnet` / `opus` / `haiku` / `claude-opus-4-7` / `inherit` | 省略時は `inherit`（メイン会話と同じモデル） |
| `permissionMode` | No | `default` / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan` | 親が `bypassPermissions` `acceptEdits` `auto` の場合は親が優先 |
| `maxTurns` | No | `10` | 最大エージェントターン数 |
| `skills` | No | `[api-conventions, error-handling]` | 起動時にスキル本文をプリロード |
| `mcpServers` | No | `[github, {playwright: {...}}]` | このサブエージェント限定のMCP接続 |
| `hooks` | No | `PreToolUse: [...]` | このサブエージェントのライフサイクルに紐づくhooks |
| `memory` | No | `user` / `project` / `local` | 永続メモリディレクトリのスコープ |
| `background` | No | `true` | 常にバックグラウンド実行（並列・非ブロッキング） |
| `effort` | No | `low` / `medium` / `high` / `xhigh` / `max` | 思考努力レベル。モデルによって利用可能な値が違う |
| `isolation` | No | `worktree` | 一時的な git worktree で実行。変更が無ければ自動削除 |
| `color` | No | `red` / `blue` / `green` / `yellow` / `purple` / `orange` / `pink` / `cyan` | UI 表示色 |
| `initialPrompt` | No | `"Start by listing all routes."` | `--agent` でメインスレッドとして起動した時の最初のユーザターン |

### 配置場所と優先順位

| 場所 | スコープ | 優先度 |
|---|---|---|
| Managed settings | 組織全体 | **1（最優先）** |
| `--agents` CLI フラグ | 現セッションのみ | 2 |
| `.claude/agents/` | プロジェクト固有 | 3 |
| `~/.claude/agents/` | 全プロジェクト | 4 |
| Plugin の `agents/` | プラグインが有効な範囲 | 5（最低） |

同名なら上位が勝つ。プラグイン由来は `<plugin-name>:<agent-name>` で名前空間が分かれるので衝突しない。

### 組み込みサブエージェント

| 名前 | モデル | ツール | 用途 |
|---|---|---|---|
| **Explore** | Haiku | 読み取り専用 | コードベース探索。`quick`/`medium`/`very thorough` の3段階 |
| **Plan** | Inherit | 読み取り専用 | plan モード中のリサーチ |
| **general-purpose** | Inherit | 全ツール | 複雑な多段タスク。`/fork` 有効時はforkに置き換わる |
| `statusline-setup` | Sonnet | — | `/statusline` で自動起動 |
| `claude-code-guide` | Haiku | — | Claude Code の使い方質問 |

### 呼び出し方の3パターン

| 方法 | 構文 | 強制力 |
|---|---|---|
| 自然言語 | `Use the code-reviewer subagent to ...` | Claudeが判断 |
| @-mention | `@"code-reviewer (agent)" ...` または `@agent-code-reviewer` | **必ず**このエージェント |
| セッション全体 | `claude --agent code-reviewer` または `.claude/settings.json` の `"agent"` | **メインスレッド自体**がそのエージェントとして起動 |

### CLIワンライナー（保存せず使い捨て）

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer. Focus on code quality, security, and best practices.",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```

### Subagents / Agent view / Agent teams / Worktrees の選択フロー

```
タスクは並列で動かしたい？
├─ No → 普通のメイン会話 / Skills でOK
└─ Yes
   ├─ 1セッション内で完結し、結果サマリだけ欲しい → Subagents
   ├─ 自分で複数セッションを監視したい → Agent view (claude agents)
   ├─ ワーカー同士が議論・調整する必要がある → Agent teams（実験機能）
   └─ ファイル衝突を物理的に防ぎたい → Worktrees（他と組合せ可）
```

### Agent SDK での定義（プログラマティック）

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Review the authentication module for security issues",
  options: {
    allowedTools: ["Read", "Grep", "Glob", "Agent"],   // Agent ツール必須
    agents: {
      "code-reviewer": {
        description: "Expert code review specialist. Use for security and maintainability reviews.",
        prompt: "You are a code review specialist...",
        tools: ["Read", "Grep", "Glob"],
        model: "sonnet",
      },
    },
  },
})) {
  if ("result" in message) console.log(message.result);
}
```

ポイント：
- `allowedTools` に **`Agent`** を入れないとサブエージェントが起動できない
- サブエージェントが受け取るのは Agent tool の `prompt` 文字列**のみ**。親会話の履歴は渡らない
- プログラム定義はファイル定義より優先される

---

## 5. よくあるハマりどころ

| 症状 | 原因 | 解決 |
|---|---|---|
| サブエージェントが自動委譲されない | `description` が抽象的、または `Agent` tool が `allowedTools` に無い（SDK） | `description` に「use proactively」「after code changes」「when reviewing PR」など具体的なトリガー語を入れる。SDKでは `allowedTools` に `Agent` を必ず追加 |
| ファイルを編集したのに反映されない | サブエージェントは**セッション開始時にロード**される。実行中の編集は読まれない | セッションを再起動。`/agents` UI 経由なら即時反映される |
| メイン会話の context が膨れる | サブエージェントが詳細レポートを返しすぎ | サブエージェントの prompt に「Return only a 5-bullet summary」など出力制限を書く |
| `Task tool not found` 系のエラー | v2.1.63 で **Task → Agent** にリネーム。古い設定や記事の `Task(...)` をそのまま使っている | エイリアスは生きているはずだが、新規は `Agent(...)` で書く |
| 並列実行されない | `background: true` を付けていない、または依存タスク扱いされた | `background: true` を frontmatter に追加。`/fork` 有効時は自動でバックグラウンド |
| サブエージェントが `Edit` できない | `tools:` 指定で `Edit` を入れ忘れた | allowlist に `Edit` を追加。または `tools` を省略して全継承 |
| 親の `acceptEdits` を子の `default` で上書きしたい | できない | 親が `bypassPermissions` `acceptEdits` `auto` の場合、子の `permissionMode` は無視される |
| サブエージェントから子を呼びたい | サブエージェントは**サブエージェントを生成できない**（無限ネスト防止） | メイン会話に戻ってチェーン呼び出し、または Skills で代替 |
| `~/.claude/agents/` を作ったのに `/agents` に出ない | Claude Code 起動後にディレクトリを作った | セッション再起動。または `/agents` UI から作ると即時反映 |
| `--agent` で起動したセッションが `/agents` で出ない | `--agent` はメインスレッド自体をそのエージェントにする機能。サブエージェントとしては動いていない | 起動時のヘッダに `@<name>` が出ているか確認 |
| Windows で長い prompt がエラーになる | コマンドライン長制限（8191文字） | filesystem-based の `.md` ファイルにする |
| Plugin 由来のエージェントで `hooks` が効かない | セキュリティ理由で plugin agent では `hooks` `mcpServers` `permissionMode` は無視される | `.claude/agents/` または `~/.claude/agents/` にコピーして使う |

---

## 6. 次に学ぶこと

- [Skills](https://code.claude.com/docs/en/skills) — メイン会話のコンテキストで動く再利用可能ワークフロー。ネストしたい時の代替
- [Hooks](https://code.claude.com/docs/en/hooks) — `SubagentStart` / `SubagentStop` / `PreToolUse` で決定的処理を差し込む
- [Agent teams](https://code.claude.com/docs/en/agent-teams) — サブエージェントを超えてワーカー間通信が必要になったら
- [Worktrees](https://code.claude.com/docs/en/worktrees) — `isolation: worktree` の本質を理解する
- [Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview) — プログラム制御でサブエージェントを動かす（CI/CD、自動化）
- [Permissions](https://code.claude.com/docs/en/permissions) — `Agent(name)` 形式の許可/拒否ルール

---

## 7. 参照

### 公式ドキュメント
- [Create custom subagents](https://code.claude.com/docs/en/sub-agents) — 本ハンズオンのメインソース。frontmatter 全リファレンスと公式チュートリアル
- [Run agents in parallel](https://code.claude.com/docs/en/agents) — Subagents / Agent view / Agent teams / Worktrees の比較表
- [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams) — Agent teams（実験機能）との比較。`SendMessage` ツールと共有 task list
- [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents) — `AgentDefinition` 全フィールド、TypeScript/Python サンプル、resume パターン
- [Permissions](https://code.claude.com/docs/en/permissions) — `Agent(subagent-name)` 形式の許可ルール
- [Hooks](https://code.claude.com/docs/en/hooks) — `SubagentStart` / `SubagentStop` 仕様、`PreToolUse` 入力スキーマ
- [CLI reference](https://code.claude.com/docs/en/cli-reference) — `--agent` / `--agents` / `--disallowedTools` フラグ
- [Agent view](https://code.claude.com/docs/en/agent-view) — `claude agents` で開く別世界（混同しがち）
- [Worktrees](https://code.claude.com/docs/en/worktrees) — `isolation: worktree` の挙動詳細

### 関連記事（コミュニティの知見）
- [Zenn Claude Code タグ](https://zenn.dev/topics/claudecode) — 日本語ハンズオン記事の最新一覧
- [Qiita Claude Code タグ](https://qiita.com/tags/claudecode) — 実装例とハマりどころの蓄積

---

## 更新履歴

- 2026-05-14: 初版（revision 1）
