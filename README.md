# ichiza-starter

[ichiza](https://github.com/gr1m0h/ichiza) を導入するコミュニティ運営リポジトリのテンプレート。
workflows / Issue Forms / `ichiza.yaml` が配線済みで、複製するだけで運用を始められます。

## セットアップ

```console
$ gh repo create <owner>/<repo> --template gr1m0h/ichiza-starter
$ gh api -X PUT repos/<owner>/<repo>/actions/permissions/workflow \
    -f default_workflow_permissions=read -F can_approve_pull_request_reviews=true
```

2 行目は GitHub Actions に旗揚げ PR の作成を許可する設定です。個人アカウントの
リポジトリは既定で不許可、かつテンプレートに設定は引き継がれないため、リポジトリ
作成のたびに必要です（organization 配下なら org 設定を継承するので不要にできます）。

ブラウザで設定する場合: Settings → Actions → General → Workflow permissions →
**Allow GitHub Actions to create and approve pull requests** を有効化。

> 未設定でも `ichiza new` は失敗しません。PR の代わりに、workflow の job summary に
> 手動作成リンク（タイトル・本文入力済み）と設定手順が表示されます。

## 使い方

### イベントの旗揚げ

Actions タブ → **ichiza new** → Run workflow（slug / title / date / mode を入力）。
スマホの GitHub アプリからも実行できます。実行すると以下が生成されます:

- `events/<slug>/event.yaml` + `tasks.yaml` の PR
- lifecycle テンプレから逆算した期限つき GitHub Issues + マイルストーン

生成された event.yaml に会場・タイムテーブル等を追記してマージしてください。

### 期限リマインド

Slack Incoming Webhook を secret に登録すると、毎朝 09:00 JST に期限超過 +
7日以内のタスクが通知されます（announce タスクには X 投稿リンク付き）。

```console
$ gh secret set SLACK_WEBHOOK_URL --repo <owner>/<repo>
```

### カスタマイズ

- `ichiza.yaml` — 開催形態・役割・会場・配信などコミュニティの既定値
- `templates/lifecycle.yaml` — 開催日からのオフセットで定義するタスク雛形
- `.github/ISSUE_TEMPLATE/speaker.yml` — 登壇者情報の Issue Form（`ichiza speakers` が収集）

フル構成の実例は [examples/meetup](https://github.com/gr1m0h/ichiza/tree/main/examples/meetup) を参照。
