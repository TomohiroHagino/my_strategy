# Action Mailbox（Rails 6）

## ひとことで言うと
Rails 6 標準の受信メール処理機能。外部メールサービスから受け取ったメールを `InboundEmail` として保存し、ルーティングで Mailbox クラスへ振り分けて `process` で処理する。

## 役割・なぜ必要か
- 「メールを受け取って Rails 側で処理する」（問い合わせのチケット化、返信メールのスレッド紐付け、bounce 処理）を標準化する。
- 受信経路（ingress）は relay / mailgun / sendgrid / postmark / amazon(SES) に対応。Webhook で受け取る。
- Action Mailer が「送信」なのに対し、Action Mailbox は「受信」を担当する。

## 基本の書き方（コード）
```bash
bin/rails action_mailbox:install   # ルーティングファイルとマイグレーション生成
bin/rails db:migrate
```
```ruby
# app/mailboxes/application_mailbox.rb
class ApplicationMailbox < ActionMailbox::Base
  routing /^support@/i  => :support
  routing /^reply\+/i   => :reply
end
```
```ruby
# app/mailboxes/support_mailbox.rb
class SupportMailbox < ApplicationMailbox
  def process
    mail = inbound_email.mail   # Mail::Message
    Ticket.create!(
      from:    mail.from.first,
      subject: mail.subject,
      body:    mail.decoded
    )
  end
end
```
```ruby
# config/environments/production.rb
config.action_mailbox.ingress = :sendgrid   # :mailgun / :postmark / :relay など
```
```bash
# ingress 認証用パスワードは credentials に保存
bin/rails credentials:edit
# action_mailbox:
#   ingress_password: ...
```
- 開発環境では `/rails/conductor/action_mailbox/inbound_emails` から疑似的にメールを投入してテストできる。

## 実務での使い方・定番パターン
- 問い合わせ用アドレス宛のメールを受信して `Ticket` レコード化し、サポート画面に表示する。
- 通知メールへの返信（`reply+<id>@`）を該当スレッドに紐付ける。
- 送れなかったメール（bounce）を受信して配信停止フラグを立てる。

## ハマりどころ / アンチパターン
- ingress の Webhook URL とプロバイダ側の設定が一致していないとメールが届かない。プロバイダ管理画面で受信フックを設定する。
- 認証用の `INGRESS_PASSWORD` / credentials を設定しないと Webhook が 401 で弾かれる。
- メール本文のパースは難しい（HTML/プレーン、引用、署名、添付）。`mail.decoded` だけに頼らず multipart を考慮する。
- 本番ではメールプロバイダ（SendGrid 等）との連携設定が必須。ローカルでだけ動いて本番で受信できないことが多い。

## 関連
[action_mailer.md](./action_mailer.md) / [active_job.md](./active_job.md) / [active_storage.md](./active_storage.md) / [config_credentials.md](./config_credentials.md) / [model.md](./model.md) / [pitfalls.md](./pitfalls.md)
