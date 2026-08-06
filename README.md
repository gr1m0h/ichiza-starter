# ichiza-starter

[ichiza](https://github.com/gr1m0h/ichiza) を導入するコミュニティ運営リポジトリのテンプレート。
workflows / `ichiza.yaml` / 募集ページ本文のテンプレートが配線済みで、
イベント作成から当日までスマホの GitHub アプリだけでも回せます。

## 必要なもの

- GitHub アカウント（private リポジトリの無料枠でも十分）
- Slack の Incoming Webhook URL — 期限リマインドの通知先
  （[作り方](https://api.slack.com/messaging/webhooks)。後から設定しても OK）
- （任意）connpass API キー — 申込数ウォッチを使う場合のみ。
  [利用申請](https://help.connpass.com/api/)で発行（コミュニティ・個人は無償、審査あり）

## セットアップ

1. このリポジトリ右上の **Use this template → Create a new repository**
2. **Settings → Actions → General → Workflow permissions** →
   **Allow GitHub Actions to create and approve pull requests** にチェックして Save
   （イベント作成 workflow が PR を自動作成するための許可。個人アカウントは既定で不許可）
3. **Settings → Secrets and variables → Actions** で `SLACK_WEBHOOK_URL` を登録

gh CLI:

```console
$ gh repo create <owner>/<repo> --template gr1m0h/ichiza-starter --private --clone
$ gh api -X PUT repos/<owner>/<repo>/actions/permissions/workflow \
    -f default_workflow_permissions=read -F can_approve_pull_request_reviews=true
$ gh secret set SLACK_WEBHOOK_URL --repo <owner>/<repo>
```

> 手順 2 を忘れてもイベント作成は失敗しません。PR の代わりに workflow の
> job summary に手動作成リンク（タイトル・本文入力済み）が表示されます。

## コミュニティ設定（イベント作成の前に）

この 2 ファイルからイベント定義と期限つきタスクが生成されます。
コミュニティに合わせて更新してからコミットしてください。

- **`ichiza.yaml`** — コミュニティの既定値。`defaults.mode`（開催形態）・
  `defaults.venue`（会場）・`defaults.roles`（運営役割）、hybrid / online で
  開催するなら `defaults.streaming` を見直す
- **`templates/lifecycle.yaml`** — タスクの雛形。「何日前に何をやるか」を
  `due: -30d` のようなオフセットで定義。初回はそのままでも動きます

各パラメータは
[設定リファレンス](https://github.com/gr1m0h/ichiza/blob/main/docs/configuration.md)を、
フル構成の実例は
[examples/meetup](https://github.com/gr1m0h/ichiza/tree/main/examples/meetup) を参照。

> `ichiza.yaml` の既定値は**イベント作成時に雛形へコピーされる**ため、作成後に
> 変えても既存イベントには反映されません（個別の修正は `events/<slug>/event.yaml` を直接編集）。

## イベントを作成する

1. **Actions タブ → ichiza new → Run workflow**
2. フォームに入力して実行:
   - `slug` — イベント識別子（例: `tokyo-1`。小文字英数字とハイフン）
   - `title` — イベントタイトル
   - `date` — 開催日 `YYYY-MM-DD`（5 週間以上先を推奨 — 近すぎると生成直後から期限超過）
   - `mode` — `onsite` / `hybrid` / `online`
3. 生成されるもの:
   - **PR** — `events/<slug>/event.yaml`（イベント定義）+ `tasks.yaml`（期限つきタスク）
   - **GitHub Issues** — 開催日から逆算した期限入りタスク一式 + マイルストーン
   - **募集ページ本文** — job summary に connpass にそのまま貼れる本文
     （connpass の「コピーして新規作成」→ 本文にペースト）
4. event.yaml に会場名・タイムテーブルを追記して **PR をマージ** — 以降これが SSoT

## 日々の運用

### タスク管理（GitHub Issues）

1 タスク = 1 Issue（期限入りタイトル、イベントごとのマイルストーン）。
終わったタスクは Issue を閉じてください。

リマインドの判定元は Issue ではなく `events/<slug>/tasks.yaml` です。Issue を閉じても
自動では同期されないため、リマインドを止めるには該当タスクを `done: true` にして
コミットしてください（GitHub の Web エディタからで OK）。

### 期限リマインド（自動）

毎朝 09:00 JST に期限超過 + 7 日以内のタスクを Slack へ通知します。announce ラベルの
タスクには X（Twitter）の投稿画面を開くリンクが付き、通知からワンタップで告知できます。
手動実行は Actions タブ → **ichiza remind**。

### 申込数ウォッチ（自動・要 connpass API キー）

`CONNPASS_API_KEY` secret を設定すると、毎朝 09:00 JST に開催前イベントの
申込数 / 定員充足率 / 補欠 / 受付状態を Slack へ通知します。対象は event.yaml に
`connpass_url` があるイベント（募集ページ公開後に追記）。secret 未設定の間は
自動でスキップされます。

### 登壇者の追加と募集ページの更新

1. `events/<slug>/event.yaml` の `speakers:` に登壇者情報を追記
   （形式は[設定リファレンス](https://github.com/gr1m0h/ichiza/blob/main/docs/configuration.md)を参照）
2. **Actions タブ → ichiza registry → Run workflow**（slug を入力）で募集ページ本文が
   summary に出るので、公開済みの connpass ページの本文に**まるごと貼り直す**

タイムテーブルや会場の変更も同じ流れです。文面は `templates/registry/page.md` で
カスタマイズできます。

### 振り返りを次回に効かせる

イベント後の振り返り（KPT）で出た運営改善は `templates/lifecycle.yaml` に
反映してください。次回のイベント作成から自動で効きます。

## ファイル構成

```text
.github/workflows/ichiza-new.yml      # イベント作成（Run workflow ボタン）
.github/workflows/ichiza-remind.yml   # 毎朝 09:00 JST の期限チェック（cron）
.github/workflows/ichiza-registry.yml # 募集ページ本文の再生成（Run workflow ボタン）
.github/workflows/ichiza-watch.yml    # 毎朝 09:00 JST の申込数ウォッチ（cron・要 CONNPASS_API_KEY）
ichiza.yaml                           # コミュニティの既定値
templates/lifecycle.yaml              # タスク雛形（複数可、ichiza new --lifecycle で切替）
templates/registry/                   # 募集ページ本文のテンプレート（page.md）
events/                               # イベントごとの event.yaml + tasks.yaml
```

## トラブルシューティング

- **イベント作成 workflow は成功したのに PR がない** — job summary に PR 作成リンクが
  出ています。恒久対応はセットアップ手順 2 の権限設定
- **リマインドが来ない** — `SLACK_WEBHOOK_URL` secret を確認。対象タスクがゼロの日は通知なし
- **申込数の通知が来ない** — `CONNPASS_API_KEY` secret と、対象イベントの event.yaml に
  `connpass_url` が入っているかを確認（watch の実行ログに出ます）
- **`invalid slug`** — slug は小文字英数字とハイフンのみ（例: `tokyo-1`）
