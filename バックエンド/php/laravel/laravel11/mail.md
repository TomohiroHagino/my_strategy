# メール / 通知（Mail / Notification）（Laravel 11）

## ひとことで言うと
- **Mailable** = 1通のメールを表すクラス。件名・宛先・本文（Blade/Markdown）を組み立て、`Mail::to()->send()`（または `->queue()`）で送る。
- **Notification** = 「ユーザーに知らせる」という出来事を、**mail / database / slack など複数チャネルへ同時に**届ける仕組み。

## 役割・なぜ必要か
- 「登録完了メール」「パスワードリセット」などの**メール定義をクラスに集約**し、テンプレートとロジックを分けるのが Mailable。
- 「注文確定」を *メールでも* *アプリ内通知（DB）でも* *Slackでも* 送りたい——という**届け先が複数あるイベント**を1クラスで扱うのが Notification。届け方（チャネル）を増やしても呼び出し側は変わらない。
- 送信は遅いので、`->queue()` で**キュー経由の非同期送信**にしてユーザーを待たせないのが定番（→ queues.md）。

## 基本の書き方（コード）
### Mailable
```bash
php artisan make:mail OrderShipped --markdown=emails.orders.shipped
```
```php
// app/Mail/OrderShipped.php
namespace App\Mail;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\{Content, Envelope};
use Illuminate\Queue\SerializesModels;

class OrderShipped extends Mailable
{
    use Queueable, SerializesModels;

    public function __construct(public Order $order) {}

    public function envelope(): Envelope
    {
        return new Envelope(subject: '注文を発送しました');   // 件名・差出人など
    }

    public function content(): Content
    {
        // Markdownテンプレートに $order を渡す
        return new Content(markdown: 'emails.orders.shipped', with: ['order' => $this->order]);
    }
}
```
```blade
{{-- resources/views/emails/orders/shipped.blade.php（Markdownメール） --}}
<x-mail::message>
# 発送のお知らせ

ご注文 #{{ $order->id }} を発送しました。

<x-mail::button :url="$url">
注文を確認する
</x-mail::button>

ありがとうございます。<br>{{ config('app.name') }}
</x-mail::message>
```
```php
use Illuminate\Support\Facades\Mail;

Mail::to($user)->send(new OrderShipped($order));    // 同期送信
Mail::to($user)->queue(new OrderShipped($order));   // キュー経由で非同期送信（推奨）
```

### Notification（複数チャネル）
```bash
php artisan make:notification InvoicePaid
```
```php
// app/Notifications/InvoicePaid.php
class InvoicePaid extends Notification
{
    public function __construct(public Invoice $invoice) {}

    public function via(object $notifiable): array
    {
        return ['mail', 'database'];   // メール＋アプリ内通知（DB）に同時配信
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('お支払いを受け付けました')
            ->line("請求 #{$this->invoice->id} の支払いが完了しました。")
            ->action('明細を見る', url('/invoices/'.$this->invoice->id));
    }

    public function toArray(object $notifiable): array
    {
        return ['invoice_id' => $this->invoice->id];   // database チャネルが保存する中身
    }
}

// 送る側（User に Notifiable トレイトが必要）
$user->notify(new InvoicePaid($invoice));
```
database チャネルは `php artisan make:notifications-table && migrate` で `notifications` テーブルを用意し、`$user->notifications` で読める。

## 実務での使い方・定番パターン
- **送信は基本 `->queue()`**：Mailable に `implements ShouldQueue` を付ける（または Notification を `ShouldQueue` に）と、`send()/notify()` でも自動でキューに乗る。重い送信でレスポンスを止めない。
- **開発は Mailpit / log ドライバ**：`.env` を `MAIL_MAILER=log`（`storage/logs/laravel.log` に出力）か **Mailpit**（`MAIL_MAILER=smtp`, `MAIL_HOST=mailpit`）にして、**本番に実送信しない**。Laravel Sail には Mailpit が同梱。
- **テンプレートは Markdown メール**を第一候補に。`<x-mail::message>` / `<x-mail::button>` でレスポンシブなHTML＋テキスト版が自動生成される。
- **オンデマンド通知**：DBにユーザーが無い宛先へは `Notification::route('mail', 'a@example.com')->notify(...)` で送れる。
- **Slack 等の追加チャネル**：`via()` に `'slack'` を足し `toSlack()` を実装（`laravel/slack-notification-channel` が必要）。呼び出し側は変えずに届け先を増やせる。

## ハマりどころ / アンチパターン
- **`->queue()` は非同期なので即時に届かない**：`->queue()` にした瞬間、**ワーカー（`queue:work`）が動いていないとメールは送られない**。「送ったはずなのに来ない」の典型。開発で確実に送るなら `->send()`、本番はワーカー常駐を確認（→ queues.md）。
- **MAIL設定漏れ**：`MAIL_MAILER` / `MAIL_HOST` / `MAIL_FROM_ADDRESS` 未設定で送信失敗、または From 不正でスパム判定。`config:cache` 後に `.env` を変えても反映されないので `config:clear` を忘れない。
- **本番に実メールを飛ばす事故**：ステージング/開発で本番SMTPを指したまま実顧客へ送信。環境ごとに `MAIL_MAILER=log`/Mailpit を徹底し、`Mail::fake()` でテスト時の実送信を止める。
- **大量宛先を1リクエストで同期送信**：ループ内 `->send()` でタイムアウト。**1通=1ジョブ**でキューに分散させる。
- **Notification で `via()` の戻りを間違える**：チャネル名を配列で返さないと配信されない。`mail` を入れ忘れてメールだけ来ない、なども多い。
- **Mailable に巨大データを渡す**：`SerializesModels` で識別子化されるとはいえ、キュー化メールには Eloquent モデルを渡し、コレクションや重い配列の直渡しは避ける。

## 関連
[queues.md](./queues.md) / [blade.md](./blade.md) / [config_env.md](./config_env.md)
