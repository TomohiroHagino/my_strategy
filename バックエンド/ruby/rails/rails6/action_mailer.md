# Action Mailer（Rails 6）

## ひとことで言うと
メール送信をコントローラ風に書く仕組み。Mailerクラスのメソッドで件名・宛先を組み立て、ビューで本文を描画し、`deliver_later` でActive Job経由の非同期送信ができる。

## 役割・なぜ必要か
会員登録通知・パスワードリセット・各種お知らせなどのメールを、SMTPの詳細を意識せずRailsの作法（クラス＋ビュー）で書ける。開発では実際に送らず画面で確認、本番ではSMTP/SendGrid等へ、と環境ごとに送信方法を差し替えられる。

## 基本の書き方（コード）
Mailer生成: `rails g mailer User welcome`
```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: "noreply@example.com"
  layout "mailer"
end

# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  def welcome(user)
    @user = user                      # ビューで使えるインスタンス変数
    @url  = login_url                 # default_url_options が必要
    mail(to: @user.email, subject: "登録ありがとうございます")
  end
end
```
ビュー（`app/views/user_mailer/welcome.html.erb` と `.text.erb`）:
```erb
<h1><%= @user.name %> さん、ようこそ</h1>
<p><%= link_to "ログイン", @url %></p>
```
送信:
```ruby
UserMailer.welcome(user).deliver_later  # Active Jobで非同期（推奨）
UserMailer.welcome(user).deliver_now    # その場で同期送信
```

## 実務での使い方・定番パターン
- 送信は基本 `deliver_later`。重いSMTP通信をリクエストから切り離す（[active_job.md](./active_job.md)）。本番Sidekiqなら [`../周辺インフラ/sidekiq.md`](../周辺インフラ/sidekiq.md)。
- 開発確認は letter_opener。実送信せずブラウザでプレビューできる。
```ruby
# Gemfile: gem "letter_opener", group: :development
# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
```
- 本番はSMTP（SendGrid/SES等）:
```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address: "smtp.sendgrid.net", port: 587,
  user_name: "apikey", password: Rails.application.credentials.sendgrid_api_key,
  authentication: :plain, enable_starttls_auto: true
}
config.action_mailer.default_url_options = { host: "example.com", protocol: "https" }
```
- 認証情報は credentials で管理する（[config_credentials.md](./config_credentials.md)）。

## ハマりどころ / アンチパターン
- `deliver_now` のままだとリクエスト内で同期送信され、SMTPが遅いと体感が悪い。原則 `deliver_later`。
- メール内でURLを生成するには `default_url_options`（host）が必須。未設定だと `Missing host to link to!` で失敗する。リクエストコンテキスト外（ジョブ内）では特に注意。
- 本番で `delivery_method` 未設定だと `:smtp` 既定で接続先が無く送信に失敗、または `:test` でメールが配列に溜まるだけ。環境ごとに明示する。
- `deliver_later` は実行時にレコードを再ロードするため、送信前に対象を削除すると失敗する。`discard_on` を検討（[active_job.md](./active_job.md)）。
- HTML版とテキスト版のビューを両方用意しないと、片方しか持たないクライアントで本文が崩れることがある。

## 関連
[active_job.md](./active_job.md) / [config_credentials.md](./config_credentials.md) / [`../周辺インフラ/sidekiq.md`](../周辺インフラ/sidekiq.md) / [view.md](./view.md) / [security.md](./security.md) / [pitfalls.md](./pitfalls.md)
