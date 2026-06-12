# Action Mailer（Rails 5）

## ひとことで言うと
**アプリからメールを送るための仕組み**。メールを「コントローラ＋ビュー」と同じ構造（Mailer クラス＋ビューテンプレート）で組み立てて送信する。Rails 5 からは `ApplicationMailer` を継承する。

## 役割・なぜ必要か
- 会員登録の確認・パスワードリセット・通知などをアプリから送るためにある。
- 件名・宛先・本文（HTML/テキスト）をRubyとERBで組み立て、送信処理（SMTP等）を抽象化する。

## 基本の書き方（コード）
```ruby
# app/mailers/application_mailer.rb（Rails 5 で導入された共通基底）
class ApplicationMailer < ActionMailer::Base
  default from: "no-reply@example.com"
  layout "mailer"
end

# rails g mailer UserMailer welcome で生成
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  def welcome(user)
    @user = user                       # ビューで使える
    mail(to: @user.email, subject: "ようこそ")
  end
end
```
```erb
<%# app/views/user_mailer/welcome.html.erb %>
<p><%= @user.name %> さん、登録ありがとうございます。</p>
```
```ruby
# 送信
UserMailer.welcome(user).deliver_later   # 非同期（Active Job 経由・推奨）
UserMailer.welcome(user).deliver_now     # 同期（その場で送る）
```

## 実務での使い方・定番パターン
- **`deliver_later` を既定にする**：送信をリクエストから切り離す（Active Job 経由）。→ [active_job.md](./active_job.md)
- **HTML版とテキスト版の両方**（`welcome.html.erb` と `welcome.text.erb`）を用意するとマルチパートで届く。
- **開発確認**：`letter_opener` gem で送らずにブラウザでプレビュー、または **Mailer Preview**（`test/mailers/previews/`）で確認。
- **本番のSMTP設定**は `config/environments/production.rb` の `config.action_mailer.smtp_settings` に。認証情報は credentials/ENV から読む。→ [config_credentials.md](./config_credentials.md)
- **`config.action_mailer.default_url_options`** にホストを設定しないと、メール内の `*_url` が生成できず例外になる。

## ハマりどころ / アンチパターン
- **`default_url_options` 未設定**：メール本文で `_url` ヘルパーを使うと「host が無い」で落ちる。環境ごとに設定する。
- **`deliver` の古い書き方**：Rails 5 では `deliver`（無印）は非推奨。`deliver_now` / `deliver_later` を使う。
- **トランザクション内で `deliver_later`**：コミット前にジョブが走ると「まだ無いレコード」で失敗。`after_commit` で送る。→ [active_job.md](./active_job.md)
- **本文に重いロジック**：メールビューにクエリや業務判定を書かない。データは Mailer メソッドで用意。
- **送信失敗の握り潰し**：SMTPエラーを無視すると「届かないのに気づけない」。監視・リトライを用意。

## 関連
[active_job.md](./active_job.md) / [config_credentials.md](./config_credentials.md) / [../周辺インフラ/sidekiq.md](../周辺インフラ/sidekiq.md)
