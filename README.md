# ichiza-starter

[ichiza](https://github.com/gr1m0h/ichiza) を導入するコミュニティ運営リポジトリのテンプレート。
workflows / `ichiza.yaml` / 原稿テンプレが配線済みで、この README の手順だけで
イベント運営を始められます。旗揚げから当日までスマホの GitHub アプリだけでも回せます。

## 必要なもの

- GitHub アカウント（Actions が動けば OK。private リポジトリの無料枠でも十分）
- Slack の Incoming Webhook URL — 期限リマインドの通知先
  （[作り方](https://api.slack.com/messaging/webhooks)。後から設定しても OK）
- （任意）`gh` CLI — ブラウザだけでもすべて設定できます

## セットアップ

### ブラウザで

1. このリポジトリ右上の **Use this template → Create a new repository**
2. 作成したリポジトリの **Settings → Actions → General → Workflow permissions** →
   **Allow GitHub Actions to create and approve pull requests** にチェックして Save。
   旗揚げ workflow が PR を自動作成するための許可です（個人アカウントの
   リポジトリは既定で不許可。テンプレートに設定は引き継がれません）
3. **Settings → Secrets and variables → Actions → New repository secret** で
   `SLACK_WEBHOOK_URL` を登録

### gh CLI で

```console
$ gh repo create <owner>/<repo> --template gr1m0h/ichiza-starter --private --clone
$ gh api -X PUT repos/<owner>/<repo>/actions/permissions/workflow \
    -f default_workflow_permissions=read -F can_approve_pull_request_reviews=true
$ gh secret set SLACK_WEBHOOK_URL --repo <owner>/<repo>
```

> 手順 2 の権限設定を忘れても旗揚げは失敗しません。PR の代わりに workflow の
> job summary に手動作成リンク（タイトル・本文入力済み）と設定手順が表示されます。
> organization 配下なら org の Actions 設定で一括許可でき、リポジトリごとの設定は不要です。

## コミュニティ設定（旗揚げの前に）

旗揚げすると、この 2 ファイルの内容からイベント定義と期限つきタスクが生成されます。
自分のコミュニティに合わせて更新してからコミットしてください。

- **`ichiza.yaml`** — コミュニティの既定値。最低限見直すのは:
  - `defaults.mode` — いつもの開催形態（`onsite` / `hybrid` / `online`）
  - `defaults.venue` — 会場の定員・設備・受付方法
  - `defaults.roles` — 運営役割。ワンオペなら `[organizer]` のまま、
    分担するなら `[mc, reception, photographer, timekeeper]` のように
  - hybrid/online で開催するなら `defaults.streaming`（配信サービス・機材）
- **`templates/lifecycle.yaml`** — タスクの雛形。「何日前に何をやるか」を
  `due: -30d` のようなオフセットで定義します。初回はそのままでも動くので、
  1 度旗揚げして生成された Issues を見てから削る・足すのがおすすめです

各パラメータの意味と記法は
**[設定リファレンス](https://github.com/gr1m0h/ichiza/blob/main/docs/configuration.md)** を、
フル構成の実例（ハイブリッド配信・6役体制・チェックリスト付き Issue）は
[examples/meetup](https://github.com/gr1m0h/ichiza/tree/main/examples/meetup) を参照してください。

> どちらも既定値のままで動作はします。ただし `ichiza.yaml` の既定値は
> **旗揚げ時に雛形へコピーされる**ものなので、旗揚げ後に変えても既存イベントには
> 反映されません（個別イベントの修正は `events/<slug>/event.yaml` を直接編集）。

## イベントを旗揚げする

1. **Actions タブ → ichiza new → Run workflow**
2. フォームに入力して実行:
   - `slug` — イベント識別子（例: `tokyo-1`。小文字英数字とハイフン）
   - `title` — イベントタイトル（例: `Your Meetup #1`）
   - `date` — 開催日 `YYYY-MM-DD`。**実行日から 5 週間以上先を推奨**
     （既定タスクの最長オフセットが 35 日前のため、近すぎると生成直後から期限超過になります）
   - `mode` — `onsite`（オフライン）/ `hybrid` / `online`
3. 1〜2 分で生成されるもの:
   - **PR** — `events/<slug>/event.yaml`（イベント定義の雛形）+ `tasks.yaml`（期限つきタスク）
   - **GitHub Issues** — 開催日から逆算した期限入りタスク一式 + マイルストーン
   - **募集ページ原稿** — workflow 実行結果ページ下部の summary に、connpass に
     そのまま貼れる原稿が出ます（connpass の「コピーして新規作成」→ 本文にペースト。
     文面は `templates/registry/page.md` で自由にカスタマイズ可能）
4. event.yaml に会場名・タイムテーブルなどを追記して **PR をマージ**

あとは Issues を上から消化していくだけです。運営で気づいたことも同じリポジトリの
Issue に残すと、振り返りの一次ソースになります。

## 日々の運用

### 期限リマインド（自動）

`ichiza remind` workflow が毎朝 09:00 JST に動き、期限超過 + 7 日以内のタスクを
Slack に通知します。announce ラベルのタスクには X（Twitter）の投稿画面を開く
リンクが付くので、通知からワンタップで告知できます。

手動で試すには: Actions タブ → **ichiza remind** → Run workflow。
（リマインド対象があるのに `SLACK_WEBHOOK_URL` 未設定だと workflow が失敗します）

### 登壇者の追加と募集ページの更新

1. DM 等で受け取った登壇者情報を `events/<slug>/event.yaml` の `speakers:` に追記します
   （GitHub の Web エディタからで OK）:

   ```yaml
   speakers:
     - handle: alice
       sns: https://x.com/alice
       bio: SRE。〇〇社で△△をやっています。
       session_title: SLO入門
       remote: false
   ```

2. **Actions タブ → ichiza registry → Run workflow**（slug を入力）で
   募集ページ原稿の全文が summary に出ます。公開済みの connpass ページの
   本文に**まるごと貼り直して**ください（差分追記より簡単で崩れません）

タイムテーブルや会場の変更も同じ流れです — event.yaml を直して再生成するだけ。
文面は `templates/registry/page.md` でカスタマイズできます（登壇者 1 名分の形式も
変えたい場合は speaker テンプレートを追加 — [設定リファレンス](https://github.com/gr1m0h/ichiza/blob/main/docs/configuration.md)）。

### 振り返りを次回に効かせる

イベント後の振り返り（KPT）で出た運営改善は `templates/lifecycle.yaml` に
反映してください。次回の旗揚げから自動で効きます。

## ファイル構成

```text
.github/workflows/ichiza-new.yml      # 旗揚げ（Run workflow ボタン）
.github/workflows/ichiza-remind.yml   # 毎朝 09:00 JST の期限チェック（cron）
.github/workflows/ichiza-registry.yml # 募集ページ原稿の再生成（Run workflow ボタン）
ichiza.yaml                           # コミュニティの既定値
templates/lifecycle.yaml              # タスク雛形（ライフサイクル定義）
templates/registry/                   # 募集ページ原稿の文面テンプレ（page.md）
events/                               # 旗揚げごとに events/<slug>/ が生成される（event.yaml + tasks.yaml）
```

`templates/` 配下は設定ではなく生成のたびに展開される**テンプレート**です。
`lifecycle.yaml` は用途別に複数置けます（例: 通常回と LT 大会。`ichiza new --lifecycle` で
切替）。`registry/` はコミュニティの文面そのものなので、自由に書き換えてください。

## トラブルシューティング

- **旗揚げ workflow は成功したのに PR がない** — Actions の実行結果ページ下部の
  summary に PR 作成リンクが出ています。恒久対応はセットアップ手順 2 の権限設定
- **リマインドが来ない** — `SLACK_WEBHOOK_URL` secret の設定を確認。
  期限超過 + 7 日以内のタスクがゼロの日は通知自体がありません
- **`invalid slug`** — slug は小文字英数字とハイフンのみ（例: `tokyo-1`）
