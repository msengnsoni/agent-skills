---
name: create-pull-request
description: GitHub PR を作成する。`.github/PULL_REQUEST_TEMPLATE.md` があればそれに従い、なければ remote の既存 PR 形式を踏襲する。merge 先は未指定時に remote の default branch を使う。Use when the user asks to create a pull request, open a PR, プルリクを作成, PR を出す, or gh pr create.
---

# Create Pull Request

GitHub に PR を作成する。本文フォーマットはリポジトリの慣習に合わせる。PR 本文に Cursor 由来の記載は入れない。

## Hard rules

- `gh` で PR を作成する
- `git config` は変更しない
- force push や破壊的 git 操作はユーザー明示指示がない限り行わない
- 秘密情報を含むファイルは PR に含めない。含まれそうなら警告する
- push はユーザーが明示したとき、または PR 作成に必要なときのみ行う

## Step 1: 現状把握

次を並列で実行する:

```bash
git status
git diff
git diff --cached
git branch -vv
git log --oneline -10
```

あわせて以下を確認する:

- 現在ブランチと upstream の有無
- base ブランチ（ユーザー指定がなければ後述の default を使う）
- PR に含まれるコミット範囲

base が未指定なら default branch を解決する:

```bash
gh repo view --json defaultBranchRef -q '.defaultBranchRef.name'
```

`gh` が使えない、または remote が未設定の場合は次の順で fallback する:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|.*/||'
# 失敗時: main → master → 現在ブランチ
```

差分の把握:

```bash
git diff <base>...HEAD
git log <base>...HEAD --oneline
```

## Step 2: PR 本文テンプレートの決定

**優先順位:**

1. リポジトリ内の PR テンプレート
2. remote の既存 PR の本文形式
3. 最小フォーマット（Summary + Test plan）

### 2a. ローカルテンプレート

次を順に探す。見つかったら内容を読み、見出し・セクション・必須項目をそのまま踏襲する:

```bash
test -f .github/PULL_REQUEST_TEMPLATE.md && echo found
test -f .github/pull_request_template.md && echo found
test -f docs/pull_request_template.md && echo found
ls .github/PULL_REQUEST_TEMPLATE/ 2>/dev/null
```

複数テンプレートがある場合は、変更内容に最も近いものを選ぶ。判断が難しければ default 相当のテンプレートを使う。

### 2b. 既存 PR から形式を推定

ローカルテンプレートがない場合、remote の最近の merged PR を参照する:

```bash
gh pr list --state merged --limit 10 --json number,title,body
gh pr view <number> --json title,body
```

採用ルール:

- 直近の merged PR のうち、本文が空でないものを 2〜3 件読む
- 共通する見出し構造・箇条書きスタイル・セクション名を抽出する
- リポジトリ固有のチェックリストやリンク形式があれば踏襲する
- 1 件だけの特殊フォーマットは真似しない

### 2c. fallback

テンプレートも参考 PR も得られない場合:

```markdown
## Summary
<変更の要約>

## Test plan
- [ ] <確認項目>
```

## Step 3: PR タイトルと本文の起草

- **タイトル**: リポジトリの最近の PR タイトル規約に合わせる（`feat:`, `fix:` などが多ければ踏襲）。1 行で変更の目的が伝わるようにする
- **本文**: Step 2 で決めたテンプレートに沿って書く
- **Summary**: 全コミットを通した「なぜ」を 1〜3 箇条書きで要約する。単なるファイル名列挙にしない
- **Test plan**: 実際に確認できる具体的手順にする。未実施ならその旨を書く

テンプレートに該当セクションがない項目は、無理に追加しない。

## Step 4: ブランチの push

upstream がない、またはリモートより遅れている場合のみ push する:

```bash
git push -u origin HEAD
```

必要なら `network` / `all` 権限を要求する。

## Step 5: PR 作成

```bash
gh pr create \
  --base <base-branch> \
  --head <current-branch> \
  --title "<title>" \
  --body "$(cat <<'EOF'
<body>
EOF
)"
```

`--base` はユーザー指定がなければ Step 1 の default branch。

既に PR が存在する場合:

```bash
gh pr view --json url,number,title
```

- 同一ブランチの PR があれば新規作成せず、既存 PR の URL を返す
- 本文更新が必要なら `gh pr edit` を使う

## Step 6: 結果報告

ユーザーに返す内容:

- PR URL（必須）
- base ← head のブランチ
- 採用したテンプレート源（`PULL_REQUEST_TEMPLATE.md` / 参考にした PR 番号 / fallback）
- 含まれるコミット数の要約（必要なら）

## チェックリスト

```
Task Progress:
- [ ] base ブランチを確定した
- [ ] PR テンプレートまたは参考 PR を特定した
- [ ] 全コミットを反映した title/body を書いた
- [ ] 必要なら push した
- [ ] gh pr create で PR を作成した
- [ ] PR URL を返した
```
