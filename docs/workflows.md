# GitHub Actions ワークフロー設定ドキュメント

このドキュメントでは、`.github/workflows/` ディレクトリに含まれる GitHub Actions ワークフローの設定について説明します。

---

## 1. Claude Code ワークフロー (`claude.yml`)

### 概要

イシューやプルリクエストのコメントで `@claude` とメンションすることで、Claude AI を呼び出してコードの実装・レビュー・質問への回答などを行うワークフローです。

### トリガー条件

以下のイベントが発生し、かつ `@claude` というキーワードが含まれる場合に実行されます。

| イベント | 条件 |
|---|---|
| `issue_comment` | イシューコメントが作成されたとき |
| `pull_request_review_comment` | プルリクエストのレビューコメントが作成されたとき |
| `pull_request_review` | プルリクエストのレビューが送信されたとき |
| `issues` | イシューが作成（`opened`）または担当者が割り当て（`assigned`）されたとき |

### パーミッション

| パーミッション | レベル | 用途 |
|---|---|---|
| `contents` | `read` | リポジトリのコードを読み取る |
| `pull-requests` | `read` | プルリクエスト情報を読み取る |
| `issues` | `read` | イシュー情報を読み取る |
| `id-token` | `write` | OIDC認証トークンの取得 |
| `actions` | `read` | CIの実行結果を読み取る（PR上でのCI結果確認に必要） |

### ステップ

1. **Checkout repository** - リポジトリを `fetch-depth: 1` でチェックアウトします（最新の1コミットのみ取得）。
2. **Run Claude Code** - `anthropics/claude-code-action@v1` アクションを使用して Claude Code を実行します。

### 設定パラメータ

| パラメータ | 必須 | 説明 |
|---|---|---|
| `claude_code_oauth_token` | 必須 | Claude Code の認証トークン（シークレット `CLAUDE_CODE_OAUTH_TOKEN` に設定） |
| `additional_permissions` | 任意 | 追加パーミッションの設定（デフォルトで `actions: read` を付与） |
| `prompt` | 任意 | Claudeへのカスタムプロンプト（未設定時はコメント内容に従って動作） |
| `claude_args` | 任意 | Claude のオプション引数（例：`--allowed-tools Bash(gh pr *)`） |

### 必要なシークレット

| シークレット名 | 説明 |
|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code の OAuth トークン。リポジトリまたは Organization のシークレットに設定する必要があります。 |

### 使用例

イシューやプルリクエストのコメント欄に以下のように書き込むと Claude が応答します。

```
@claude このコードのバグを修正してください
@claude このプルリクエストをレビューしてください
@claude この関数の説明をしてください
```

---

## 2. Claude Code レビューワークフロー (`claude-code-review.yml`)

### 概要

プルリクエストが作成・更新されると自動的に Claude AI がコードレビューを実施するワークフローです。

### トリガー条件

`pull_request` イベントで以下のアクションが発生したときに実行されます。

| アクション | 説明 |
|---|---|
| `opened` | プルリクエストが新規作成されたとき |
| `synchronize` | プルリクエストのブランチに新しいコミットが追加されたとき |
| `ready_for_review` | ドラフトPRがレビュー準備完了になったとき |
| `reopened` | クローズされたPRが再オープンされたとき |

### パーミッション

| パーミッション | レベル | 用途 |
|---|---|---|
| `contents` | `read` | リポジトリのコードを読み取る |
| `pull-requests` | `read` | プルリクエスト情報を読み取る |
| `issues` | `read` | イシュー情報を読み取る |
| `id-token` | `write` | OIDC認証トークンの取得 |

### ステップ

1. **Checkout repository** - リポジトリを `fetch-depth: 1` でチェックアウトします。
2. **Run Claude Code Review** - `anthropics/claude-code-action@v1` アクションを使用してコードレビューを実行します。

### 設定パラメータ

| パラメータ | 必須 | 説明 |
|---|---|---|
| `claude_code_oauth_token` | 必須 | Claude Code の認証トークン |
| `plugin_marketplaces` | 必須 | プラグインのマーケットプレイスURL |
| `plugins` | 必須 | 使用するプラグイン（`code-review@claude-code-plugins`） |
| `prompt` | 必須 | コードレビュー実行のプロンプト（PRナンバーを自動挿入） |

### 必要なシークレット

| シークレット名 | 説明 |
|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code の OAuth トークン |

### カスタマイズオプション

#### 特定ファイルのみをレビュー対象にする

`paths` フィルターを有効にすることで、変更されたファイルの種類によってワークフローを実行するか制御できます。

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

#### レビュー対象のPR作者を絞り込む

`if` 条件を有効にすることで、特定の作者のPRのみをレビュー対象にできます。

```yaml
jobs:
  claude-review:
    if: |
      github.event.pull_request.user.login == 'external-contributor' ||
      github.event.pull_request.author_association == 'FIRST_TIME_CONTRIBUTOR'
```

---

## セットアップ手順

1. [Claude Code](https://claude.ai/code) にサインインし、OAuth トークンを取得します。
2. GitHub リポジトリの **Settings > Secrets and variables > Actions** に移動します。
3. `CLAUDE_CODE_OAUTH_TOKEN` という名前でシークレットを追加し、取得したトークンを設定します。
4. ワークフローファイルがリポジトリの `.github/workflows/` ディレクトリに存在することを確認します。

---

## 参考リンク

- [Claude Code Action ドキュメント](https://github.com/anthropics/claude-code-action)
- [使用方法と設定オプション](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md)
- [Claude Code CLI リファレンス](https://code.claude.com/docs/en/cli-reference)
- [よくある質問 (FAQ)](https://github.com/anthropics/claude-code-action/blob/main/docs/faq.md)
