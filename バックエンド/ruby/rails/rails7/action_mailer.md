# Action Mailer（メール送信）（Rails 7）

## ひとことで言うと
**Rails 標準のメール送信機能**。コントローラ＋ビューと同じ構造で「Mailer クラス（送信ロジック）」と「メールテンプレート（本文）」を書き、`deliver_later` / `deliver_now` で送る。

## 役割・なぜ必要か
- 会員登録の歓迎メール・パスワードリセット・通知などを、**ビューと同じ書き方**（HTML/テキスト両方）で組み立てるためにある。
- Active Job と統合され、`deliver_later` で**送信処理をバックグラウンドに逃がせる**＝リクエストを待たせない。

## 基本の書き方（コード）
```ruby
# rails g mailer User welcome で生成
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  default from: "no-reply@example.com"

  def welcome(user)
    @user = user                                  # ビューへ渡すインスタンス変数
    @url  = login_url
    mail(to: @user.email, subject: "ようこそ")    # HTML/テキスト両テンプレを自動で使う
  end
end
```
```erb
<%# app/views/user_mailer/welcome.html.erb %>
<h1><%= @user.name %> さん、登録ありがとうございます</h1>
<p><%= link_to "ログイン", @url %></p>
```
```ruby
# 送信
UserMailer.welcome(user).deliver_later   # 非同期（Active Job 経由）★通常はこちら
UserMailer.welcome(user).deliver_now     # 同期（その場で送信）
```

## 実務での使い方・定番パターン
- **基本は `deliver_later`**：SMTP は遅い/失敗しうるので、リクエストから切り離す。裏は Active Job → [active_job.md](./active_job.md)
- **メールプレビュー**：`test/mailers/previews/` に preview クラスを置くと `/rails/mailers` でブラウザ確認できる（送信せず見た目チェック）。
- **開発環境は letter_opener**：実送信せずブラウザでメールを開いて確認（`gem "letter_opener"` ＋ `delivery_method = :letter_opener`）。
  ```ruby
  # config/environments/development.rb
  config.action_mailer.delivery_method = :letter_opener
  config.action_mailer.perform_deliveries = true
  ```
- **本番は SMTP / 外部送信サービス**（SendGrid・SES・Postmark 等）を `smtp_settings` で設定。認証情報は credentials/ENV へ。
- **URL ヘルパーを使うなら `default_url_options[:host]` 必須**（メールにはリクエストの host が無いため）。

## ハマりどころ / アンチパターン
- **`deliver_now` でリクエストをブロック（最頻）**：SMTP 応答待ちでレスポンスが遅延・タイムアウト。原則 `deliver_later`。
- **本番の送信設定漏れ**：`delivery_method`/`smtp_settings`/`default_url_options[:host]` のどれか欠けで「開発では飛ぶのに本番で飛ばない/リンクが localhost」。
- **`deliver_later` をトランザクション内で**：コミット前にジョブが走り「まだ無いレコード」を参照。`after_commit` で送る。→ [active_job.md](./active_job.md)
- **HTML だけ用意してテキスト版なし**：迷惑メール判定されやすい。`*.text.erb` も用意。
- **送信失敗を無視**：本番は外部依存。失敗時のリトライ（Active Job 側）と監視を持つ。
- **個人情報を本文/ログに過剰出力**：メール本文・ログの取り扱いに注意。

## 関連
[active_job.md](./active_job.md) / [controller.md](./controller.md)
