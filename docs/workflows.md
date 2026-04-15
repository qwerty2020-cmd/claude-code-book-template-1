# GitHub Actions ワークフロー ドキュメント

このドキュメントでは、`.github/workflows/` ディレクトリに含まれる GitHub Actions ワークフローの設定について説明します。

---

## 1. Claude Code ワークフロー (`claude.yml`)

### 概要

`@claude` メンションをトリガーとして Claude AI が自動的に応答・実行するワークフローです。Issue やプルリクエストのコメントで `@claude` と書くことで Claude に作業を依頼できます。

### トリガー条件

以下のイベントで起動します。

| イベント | 条件 |
|---|---|
| `issue_comment` | コメントが作成されたとき |
| `pull_request_review_comment` | PRのレビューコメントが作成されたとき |
| `issues` | Issueが作成またはアサインされたとき |
| `pull_request_review` | PRレビューが送信されたとき |

ただし、コメントまたはIssue本文に `@claude` が含まれている場合のみ実行されます。

### 実行条件 (if 文)

```yaml
if: |
  (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
  (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
  (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
  (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
```

### パーミッション

| パーミッション | 権限 | 用途 |
|---|---|---|
| `contents` | read | リポジトリのコード読み取り |
| `pull-requests` | read | PRの情報読み取り |
| `issues` | read | Issueの情報読み取り |
| `id-token` | write | 認証トークン発行 |
| `actions` | read | CI実行結果の読み取り |

### 主要設定

- **使用アクション**: `anthropics/claude-code-action@v1`
- **認証**: `CLAUDE_CODE_OAUTH_TOKEN` シークレットを使用
- **追加パーミッション**: `actions: read`（PRのCI結果読み取りのため）

### 使い方

Issue やプルリクエストのコメントに `@claude` を含めて書くだけです。

```
@claude このコードをレビューしてください
@claude バグを修正してください
@claude ドキュメントを更新してください
```

### カスタマイズ

`claude_args` オプションで Claude の動作をカスタマイズできます。

```yaml
claude_args: '--allowed-tools Bash(gh pr *)'
```

---

## 2. Claude Code Review ワークフロー (`claude-code-review.yml`)

### 概要

プルリクエストが作成・更新されると自動的に Claude AI がコードレビューを行うワークフローです。

### トリガー条件

以下の PR イベントで起動します。

| イベントタイプ | 説明 |
|---|---|
| `opened` | PRが新規作成されたとき |
| `synchronize` | PRに新しいコミットがプッシュされたとき |
| `ready_for_review` | ドラフトPRがレビュー可能状態になったとき |
| `reopened` | クローズされたPRが再オープンされたとき |

### パーミッション

| パーミッション | 権限 | 用途 |
|---|---|---|
| `contents` | read | リポジトリのコード読み取り |
| `pull-requests` | read | PRの情報読み取り |
| `issues` | read | Issueの情報読み取り |
| `id-token` | write | 認証トークン発行 |

### 主要設定

- **使用アクション**: `anthropics/claude-code-action@v1`
- **認証**: `CLAUDE_CODE_OAUTH_TOKEN` シークレットを使用
- **プラグイン**: `code-review@claude-code-plugins`

### 動作内容

PR が作成・更新されると、Claude が以下のプロンプトでコードレビューを実行します。

```
/code-review:code-review <repository>/pull/<pr-number>
```

### カスタマイズ

#### 特定ファイルのみを対象にする

`paths` フィルターを使用して特定のファイルが変更された場合のみ実行できます。

```yaml
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
    paths:
      - "src/**/*.ts"
      - "src/**/*.tsx"
      - "src/**/*.js"
      - "src/**/*.jsx"
```

#### 特定の作者のPRのみレビューする

`if` 条件を追加することで、特定ユーザーのPRのみレビュー対象にできます。

```yaml
jobs:
  claude-review:
    if: |
      github.event.pull_request.user.login == 'external-contributor' ||
      github.event.pull_request.author_association == 'FIRST_TIME_CONTRIBUTOR'
```

---

## 必要なシークレット設定

両ワークフローで以下のシークレットが必要です。

| シークレット名 | 説明 |
|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code の OAuth トークン |

シークレットはリポジトリの **Settings > Secrets and variables > Actions** から設定してください。

---

## 参考リンク

- [claude-code-action リポジトリ](https://github.com/anthropics/claude-code-action)
- [使用方法ドキュメント](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md)
- [Claude Code CLI リファレンス](https://code.claude.com/docs/en/cli-reference)
