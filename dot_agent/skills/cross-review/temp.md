---
name: dual-review
description: PR を Claude(code-review skill) と Codex の2系統でレビューし、比較・統合した最終レビューを生成する（インラインコメント投稿は承認制）
argument-hint: "<PR番号 or PR URL>"
allowed-tools: Bash, Read, Grep, Glob, Skill, Write, AskUserQuestion
disable-model-invocation: true
---

指定された PR を **Claude（`code-review` skill）と Codex（codex-companion review）の2系統**でレビューし、
両者を比較・統合した最終レビューを生成する。報告は日本語で行う。

レビュー対象: `$ARGUMENTS`
（空の場合は `gh pr list` を表示し、対象 PR の指定を促して中断する）

## 前提となるプラグイン・ツール

| ツール | 確認コマンド | 入手方法 |
|---|---|---|
| `openai-codex` プラグイン | `ls ~/.claude/plugins/marketplaces/openai-codex` | `claude plugin marketplace add openai-codex` → `claude plugin install codex@openai-codex` |
| `code-review` skill | Claude Code 組み込みスキルとして提供 | — |
| `gh` CLI | `gh --version` | https://cli.github.com |

> **Note**: Claude Code のプラグインインストールは「マーケットプレイス登録（カタログの追加）」→「プラグインのインストール」の2ステップ構成。`marketplace add` はカタログを登録するだけで、プラグイン本体はその後 `plugin install` で個別に入れる。

## 厳守する制約

- **レビューのみ**。コードの修正・パッチ適用はしない。
- **インラインコメント／レビューの投稿は、必ず `AskUserQuestion` で明示承認を得てから行う。無断投稿は禁止。**
- 投稿する場合も `event` は **`COMMENT` 固定**（`APPROVE` / `REQUEST_CHANGES` はしない＝マージ可否の最終判断は人間に委ねる）。
- 最終レビューでは **Claude / Codex のどちらの指摘かは明示しない**（総合した1つの結果として提示する）。
- 一時 worktree・一時ファイルは最後に**必ず後始末**する。
- Codex が利用制限・エラーで失敗した場合は、その旨を伝え **Claude（`code-review`）の結果のみ**で統合・報告する。

## 手順

### 0. 前提把握

- `gh pr view <PR> --json number,title,state,isDraft,baseRefName,headRefName,headRefOid,author,additions,deletions,changedFiles` を取得。
- `state` が closed、または `isDraft` が true の場合は、その旨を伝えて中断（必要性をユーザーに確認）。
- `baseRefName` が `main` / `develop` 以外なら **スタック PR**。base 差分の取り違えに注意する。
- リポジトリは `gh repo view --json nameWithOwner -q .nameWithOwner` で解決（URL 指定時は URL から owner/repo/番号 をパース）。

### 1. レビュー対象ブランチの準備（Codex 用）

- `git fetch origin <headRefName> <baseRefName>`
- `git worktree prune`
- `git worktree add --detach /tmp/pr<番号>-review origin/<headRefName>`
  - **このリポジトリは worktree が複数あり、PR ブランチが既に別 worktree でチェックアウト済みのことが多い。**
    現 worktree で checkout すると競合するため、必ず **detached worktree を新規作成**する。
- `git -C /tmp/pr<番号>-review rev-parse HEAD` が `headRefOid` と一致することを確認。

### 2. Codex レビュー（バックグラウンド起動）

次を **`run_in_background: true`** で起動する:

```sh
node "~/.claude/plugins/marketplaces/openai-codex/plugins/codex/scripts/codex-companion.mjs" \
  review --cwd /tmp/pr<番号>-review --base origin/<baseRefName> --scope branch --wait
```

**Codex review の罠（重要）**:
- `codex-companion.mjs review` は **ローカルの git state** を対象とする（PR 番号を直接は受けない）。だから手順1の worktree 化が必須。
- `--cwd` でレビュー対象ディレクトリ、`--base <ref> --scope branch` で **base...head の差分だけ**に限定、`--wait` で**結果を stdout に出力**。
- ブロッカーが無い場合は**要約のみ**を返す（指摘ゼロ＝問題なしの意味）。
- 出力ファイルのパスを控え、完了通知または Read で結果を取得する。

### 3. Claude レビュー（`code-review` skill）

- `Skill` ツールで **`code-review`** を起動し、引数に PR 番号/URL を渡す。
- **重要: `code-review` skill は最終ステップで `gh` による PR コメントを自動投稿する仕様だが、本 command ではそれを実行させない。**
  検出された issues（説明・confidence スコア・根拠）のリストを**統合用に保持するにとどめる**。
  投稿は本 command の承認フロー（手順5）で一括して行う。

### 4. 比較・統合

- Codex（手順2）と `code-review`（手順3）の結果が揃ったら統合する。
- 重複指摘は1つにまとめ、片方だけの指摘も取りこぼさない。confidence の低いもの・false positive は落とす。
- 必要に応じて `/tmp/pr<番号>-review` 内のファイルを Read し、指摘の妥当性を自分で再確認する。
- 出力は「**総評（マージ可否の所見）**」＋「**actionable な指摘（`path:line` 付き）**」に整理する。

### 5. 投稿（承認必須）

- `AskUserQuestion` で「この内容を PR にインラインコメント付きレビューとして投稿してよいか」を確認する（選択肢: 投稿する / 投稿しない）。
- **承認された場合のみ**:
  - payload JSON を作成（`/tmp` に置く）:
    - `commit_id`: `headRefOid`（PR head の full sha）
    - `event`: `"COMMENT"`
    - `body`: 総評
    - `comments`: `[{ "path", "line", "side": "RIGHT", "body" }, ...]`
  - `gh api --method POST /repos/<owner>/<repo>/pulls/<番号>/reviews --input <payload.json>`
  - 投稿後 `gh api /repos/<owner>/<repo>/pulls/<番号>/comments` でインラインが付いたか検証する。
- **否認の場合は投稿せず**、統合レビューをチャットに提示して終了する。

### 6. 後始末

- `git worktree remove /tmp/pr<番号>-review`（必要なら `--force`）。
- 一時 payload JSON を削除。
- `git worktree list` で対象 worktree が消えたことを確認する。

## 補足

- 投稿名義は実行ユーザーの `gh` 認証アカウント。自分自身の PR には `APPROVE` 不可（本 command は `COMMENT` 固定のため通常問題は出ないが留意）。
- インラインコメントの `line` は **新ファイル側の行番号**（追加行なら `side: "RIGHT"`）。行がずれると API が 422 を返すため、`gh pr diff` で行番号を確認してから指定する。