# ichiza-starter

[ichiza](https://github.com/gr1m0h/ichiza) を導入する技術勉強会の運営リポジトリ用テンプレートです。
イベント定義、運営タスク、GitHub Actions が配線済みで、**1 イベント = 1 Dashboard Issue** を中心に運営できます。
日常操作は GitHub アプリだけでも完結し、必要になったら Cloudflare Workers 上の Web コックピットを追加できます。

## 必要なもの

- GitHub アカウント（private リポジトリでも利用可能）
- （任意）Slack の Incoming Webhook URL — 期限リマインドの通知先
- （任意）connpass API キー — 申込数ウォッチを使う場合のみ
- （任意）Cloudflare アカウント — Web コックピットを使う場合のみ

## 基本セットアップ

1. このリポジトリ右上の **Use this template → Create a new repository**
2. **Settings → Actions → General → Workflow permissions** で
   **Allow GitHub Actions to create and approve pull requests** を有効化
3. Slack 通知を使う場合は **Settings → Secrets and variables → Actions** に
   `SLACK_WEBHOOK_URL` を登録

gh CLI では次のように設定できます。

```console
$ gh repo create <owner>/<repo> --template gr1m0h/ichiza-starter --private --clone
$ gh api -X PUT repos/<owner>/<repo>/actions/permissions/workflow \
    -f default_workflow_permissions=read -F can_approve_pull_request_reviews=true
$ gh secret set SLACK_WEBHOOK_URL --repo <owner>/<repo>
```

> 手順 2 を忘れてもイベント定義は失われません。workflow の job summary に、
> タイトルと本文が入力済みの PR 作成リンクが表示されます。

## コミュニティ設定

イベントを作る前に、次のファイルをコミュニティに合わせて更新します。

- **`ichiza.yaml`** — タイムゾーン、運営メンバー、開催形態・会場・役割などの既定値
- **`templates/lifecycle.yaml`** — 「何日前に何をするか」を定義する運営タスクの雛形

`templates/lifecycle.yaml` の各タスクには、イベントをまたいで安定する一意な `id` が必要です。
`ichiza.yaml` の `members` には GitHub ユーザー名・メールアドレス・Slack User ID を登録します。
これは Dashboard の担当者対応、Web に入れる運営者の許可リスト、Slack メンションに使われます。

```yaml
timezone: Asia/Tokyo

members:
  - github: octocat
    email: octocat@example.com
    slack_user_id: U0123456789
```

設定項目は
[設定リファレンス](https://github.com/gr1m0h/ichiza/blob/main/docs/configuration.md)、
実例は
[examples/meetup](https://github.com/gr1m0h/ichiza/tree/main/examples/meetup) を参照してください。

> `ichiza.yaml` の既定値はイベント作成時にコピーされます。作成後に既定値を変えても、
> 既存イベントには反映されません。

## イベントを作成する

1. **Actions → ichiza new → Run workflow**
2. フォームへ入力して実行
   - `slug` — イベント識別子（例: `tokyo-1`。小文字英数字とハイフン）
   - `title` — イベントタイトル
   - `date` — 開催日 `YYYY-MM-DD`（5 週間以上先を推奨）
   - `mode` — `onsite` / `hybrid` / `online`
3. workflow が次を生成
   - **PR** — `events/<slug>/event.yaml` と `tasks.yaml`
   - **Dashboard Issue** — イベント情報と期限つきタスクをまとめた 1 件の Issue
   - **募集ページ本文** — connpass に貼り付けられる本文を job summary に出力
4. `event.yaml` に会場やタイムテーブルを追記し、PR をマージ

`events/<slug>/event.yaml` と `tasks.yaml` が定義の正本、Dashboard Issue が日々の操作面です。
PR を main へマージすると `ichiza dashboard sync` が task ID をキーに定義を再反映します。
完了済みチェック・Webから変更した現在担当者・担当解除・Notes は保持され、追加タスクは未完了として加わります。

## 日々の運用

### Dashboard Issue

各イベントは 1 件の Dashboard Issue で管理します。完了状態は、Issue 本文にある
**最上位の運営タスクのチェックボックス**が持ちます。

- 完了したら `- [ ]` を `- [x]` にする
- タスク本文や Notes 内の入れ子チェックボックスは完了判定に含めない
- 全タスクが完了すると `ichiza dashboard` が Issue を自動で閉じる
- 完了済みタスクを未完了へ戻すと Issue も再度開く
- タイトル・期限・ラベル・管理用 HTML comment は直接編集せず、`event.yaml` / `tasks.yaml` をPRで変更する
- 作成後の現在担当者はDashboard Issueの実行状態。Webのイベント詳細またはMy Pageから `ichiza.yaml` のメンバーを選択する

GitHub Projects を使う場合は、Dashboard Issue を「イベント 1 件」のカードとして置けます。
細かなタスクはカードを増やさず、Issue 内のチェックボックスで扱います。

### 期限リマインド

毎朝 09:00 JST に、期限超過または 7 日以内の未完了タスクを Slack へ通知します。
完了判定と現在担当者は Dashboard Issue から読み取り、`ichiza.yaml` の
`slack_user_id` がある担当者にはメンションします。`SLACK_WEBHOOK_URL` が未設定なら
workflow は安全にスキップします。手動実行は **Actions → ichiza remind** です。

### 申込数ウォッチ（任意）

`CONNPASS_API_KEY` secret を設定すると、毎朝 09:00 JST に開催前イベントの
申込数、定員充足率、補欠、受付状態を Slack へ通知します。対象は `event.yaml` に
`connpass_url` があるイベントです。API キーは
[connpass API 利用申請](https://help.connpass.com/api/)から取得します。

### 登壇者・募集ページの更新

1. `events/<slug>/event.yaml` の `speakers`、タイムテーブル、会場情報などを更新
2. **Actions → ichiza registry → Run workflow** で本文を再生成
3. job summary の本文を公開済み connpass ページへ貼り直す

文面は `templates/registry/page.md` でカスタマイズできます。

### 振り返りを次回へつなぐ

イベント後の KPT などで得た改善を `templates/lifecycle.yaml` に反映すると、
次回以降のイベント作成時に運営タスクとして自動的に取り込まれます。

## Web コックピット（任意・alpha）

Web は ichiza の任意拡張です。Hono + Cloudflare Workers の DB なし構成で、GitHub の
Dashboard Issue を表示・更新します。イベント一覧、イベント詳細、My Page、期限状態、
担当タスク、チェックボックス操作をブラウザから利用できます。

Web のソースコードは `ichiza` 本体にあり、この starter はデプロイ workflow と設定だけを持ちます。
カスタムドメインは必須ではなく、無料の `workers.dev` URL を Cloudflare Access で保護できます。

### alpha で使う GitHub PAT

`ICHIZA_GITHUB_TOKEN` には、運営リポジトリの所有者または代表運営者が発行した
**fine-grained personal access token** を登録します。個人用 PAT を共有せず、対象を
この運営リポジトリ 1 件に限定してください。

- Repository permissions: **Issues — Read and write**
- Repository permissions: **Metadata — Read-only**
- Contents 権限は不要
- 有効期限を設定し、期限切れ前にローテーションする

PAT は alpha の簡易構成です。正式運用では、個人に依存しない GitHub App への移行を想定しています。

### Secrets と Variables

GitHub Actions の **Secrets** に次を登録します。

- `CLOUDFLARE_API_TOKEN` — Worker をデプロイできる Cloudflare API token
- `ICHIZA_GITHUB_TOKEN` — 上記の fine-grained PAT

**Variables** に次を登録します。

- `CLOUDFLARE_ACCOUNT_ID` — Cloudflare Account ID
- `ICHIZA_WORKER_NAME` — Worker 名（例: `my-community-ichiza`）
- `CF_ACCESS_TEAM_DOMAIN` — Access の team domain（例: `example.cloudflareaccess.com`）
- `CF_ACCESS_AUD` — Access Application の Audience tag

### 初回デプロイと Access 設定

1. `ichiza.yaml` の `members` に、Web を利用する運営者のメールアドレスを登録
2. Cloudflare Account ID、Worker 名、API token、PAT を設定
3. `CF_ACCESS_TEAM_DOMAIN` と `CF_ACCESS_AUD` は空のまま、**Actions → ichiza web → Run workflow** を一度実行
   - Access 設定がない間、Web は fail closed でアクセスを拒否します
4. Cloudflare Dashboard の **Workers & Pages → 対象 Worker → Settings** から
   **Protect this Worker behind Access** を有効化し、許可する運営者のメールアドレスを指定
5. 作成された Access Application の team domain と Audience tag を Variables に設定
6. **ichiza web** workflow を再実行

## ファイル構成

```text
.github/workflows/ichiza-new.yml       # イベント作成
.github/workflows/ichiza-dashboard.yml # Dashboard Issue の完了状態同期
.github/workflows/ichiza-remind.yml    # 毎朝の期限リマインド
.github/workflows/ichiza-registry.yml  # 募集ページ本文の再生成
.github/workflows/ichiza-watch.yml     # connpass 申込数ウォッチ
.github/workflows/ichiza-web.yml       # Web コックピットのデプロイ
ichiza.yaml                            # コミュニティ設定と運営メンバー
templates/lifecycle.yaml               # 運営タスク雛形
templates/registry/                    # 募集ページ本文テンプレート
events/                                # イベントごとの event.yaml + tasks.yaml
```

## トラブルシューティング

- **イベント作成後に PR がない** — job summary の PR 作成リンクを使います。恒久対応は
  Workflow permissions で PR 作成を許可することです
- **Dashboard Issue が閉じない／再度開かない** — `ichiza-dashboard.yml` の
  `issues: write` 権限と、Issue 本文先頭の `ichiza-dashboard` marker が残っているかを確認します
- **リマインドが来ない** — `SLACK_WEBHOOK_URL` と Dashboard の未完了チェックボックスを確認します。
  対象タスクがない日は通知されません
- **Web が 401 または 500** — Cloudflare Access が Worker を保護していること、
  `CF_ACCESS_TEAM_DOMAIN`、`CF_ACCESS_AUD`、`ichiza.yaml` の `members.email` を確認します
- **Web の更新が 409** — GitHub 側で Issue が更新されています。ページを再読み込みしてから操作します
- **Web から GitHub を更新できない** — PAT の期限、対象リポジトリ、Issues 権限を確認します
- **申込数の通知が来ない** — `CONNPASS_API_KEY` と `event.yaml` の `connpass_url` を確認します
- **`invalid slug`** — slug は小文字英数字とハイフンだけを使います（例: `tokyo-1`）
