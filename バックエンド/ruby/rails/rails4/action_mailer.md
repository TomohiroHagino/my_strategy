# Action Mailer（Rails 4）

## ひとことで言うと
**メール送信を担当するフレームワーク**。コントローラ＋ビューと同じ構造で、`app/mailers/` にメーラークラス、`app/views/メーラー名/` にメール本文テンプレートを置く。

## 役割・なぜ必要か
- 登録完了・パスワード再設定・通知などのメールを、HTML/テキストのテンプレートと送信処理に分けて整理するためにある。
- ビューと同じ仕組み（partial・layout・I18n）でメール本文を組み立てられる。

## 基本の書き方（コード）
```ruby
# app/mailers/user_mailer.rb
class UserMailer < ActionMailer::Base       # 4は ActionMailer::Base を継承
  default from: "noreply@example.com"

  def welcome(user)
    @user = user
    @url  = login_url
    mail(to: @user.email, subject: "ようこそ")
  end
end
```
```erb
<%# app/views/user_mailer/welcome.html.erb %>
<p><%= @user.name %> さん、登録ありがとうございます。</p>
<p><%= link_to "ログイン", @url %></p>
```
```erb
<%# app/views/user_mailer/welcome.text.erb（テキスト版。両方あればマルチパート） %>
<%= @user.name %> さん、登録ありがとうございます。
ログイン: <%= @url %>
```

### 送信
```ruby
UserMailer.welcome(@user).deliver_now       # 即時送信（4.2〜は deliver_now / deliver_later）
UserMailer.welcome(@user).deliver_later      # 非同期（Active Job 経由。4.2〜）
UserMailer.welcome(@user).deliver            # 4.0/4.1 はこちら（deliver_now は無い）
```
- **`deliver` は 4.2 で非推奨**になり `deliver_now` / `deliver_later` に分かれた。4.0/4.1 は `deliver` を使う。
- `deliver_later` は **Active Job（4.2〜）** が前提。4.0/4.1 では使えないので Sidekiq 等で自前にキューイングする。→ [active_job.md](./active_job.md)

## 実務での使い方・定番パターン
- **送信は非同期に寄せる**：リクエスト内で `deliver_now` するとSMTP待ちで遅くなる。4.2 は `deliver_later`、4.0/4.1 は Sidekiq の worker から `deliver`。→ [../周辺インフラ/sidekiq.md](../周辺インフラ/sidekiq.md)
- **設定**は `config/environments/*.rb` の `config.action_mailer.*`。
  ```ruby
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.smtp_settings = { address: ENV["SMTP_HOST"], port: 587, ... }
  config.action_mailer.default_url_options = { host: "example.com" }
  ```
- **`*_url` を使う**：メール内リンクは絶対URLが必要。`*_path` ではホストが付かない。`default_url_options` の設定が必須。
- 開発は `letter_opener` gem でブラウザ確認、`config.action_mailer.raise_delivery_errors` で失敗を顕在化。
- HTML/テキスト両テンプレートを置くとマルチパートメールになる。

## ハマりどころ / アンチパターン
- **`deliver_later` を 4.0/4.1 で呼ぶ**：Active Job が無いのでエラー。`deliver` を使うか手動でキューイング。
- **`*_path` でリンク**：メールにホスト無しの相対パスが入りリンク切れ。`*_url` ＋ `default_url_options`。
- **リクエスト内同期送信で遅延**：ユーザを待たせる。非同期へ。
- **`default_url_options` 未設定**：`Missing host to link to!` エラー。environment ごとに設定。
- **本番で例外を握り潰す**：`raise_delivery_errors = false` だと送信失敗に気づけない。監視/ログを入れる。

## 関連
[active_job.md](./active_job.md) / [service_form.md](./service_form.md) / [config_secrets.md](./config_secrets.md)
