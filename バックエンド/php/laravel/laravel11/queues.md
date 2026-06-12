# キュー / ジョブ（Queues / Jobs）（Laravel 11）

## ひとことで言うと
**重い/遅い処理をリクエストの外（別プロセス）で非同期に実行する仕組み**。「ジョブ」をキューに積み（dispatch）、常駐ワーカーが取り出して実行する。接続（sync/database/redis/sqs）を設定で差し替えても、ジョブのコードは同じ。

## 役割・なぜ必要か
- メール送信・画像処理・外部API呼び出し・集計などをレスポンスから切り離し、**ユーザーを待たせない**ためにある。
- Laravel 11 は **既定がDB寄り**（`.env` の `QUEUE_CONNECTION=database`）。`jobs` テーブルにジョブを積み、`queue:work` がそれを処理する。Redis にすれば高スループット＋**Horizon** で監視できる。
- Rails の Active Job 相当。アプリ側は `ShouldQueue` を付けて `dispatch()` するだけで、裏側の接続（同期/DB/Redis/SQS）は `config/queue.php` で交換可能。

## 基本の書き方（コード）
```bash
php artisan make:job ProcessPodcast
php artisan queue:table && php artisan migrate   # database 接続なら jobs テーブルを作る
```
```php
// app/Jobs/ProcessPodcast.php
namespace App\Jobs;

use App\Models\Podcast;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;   // ★これを implements するとキュー実行になる
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;        // 最大試行回数
    public int $backoff = 10;     // 失敗後の再試行までの秒数

    public function __construct(public Podcast $podcast) {}  // ★Eloquentは識別子だけ保存される

    public function handle(): void
    {
        // $this->podcast は実行時に最新がDBから取り直される
        $this->podcast->transcode();
    }

    public function failed(\Throwable $e): void
    {
        // 全リトライ失敗後の後始末（通知など）
    }
}
```
```php
// 呼び出し
ProcessPodcast::dispatch($podcast);                       // キューに積む（非同期）
dispatch(new ProcessPodcast($podcast))->onQueue('media') // キュー名で優先度分け
    ->delay(now()->addMinutes(5));                       // 遅延実行
ProcessPodcast::dispatchSync($podcast);                  // その場で同期実行（テスト/即時）
```
```bash
# ワーカー起動（常駐させる）
php artisan queue:work --queue=high,default --tries=3
php artisan queue:listen   # 開発用（コード変更を毎回読み直す。本番は work + restart）
```

## 実務での使い方・定番パターン
- **接続の選択**：`sync`（即時・キューを使わない、開発で手早く確認）/ `database`（11既定・追加インフラ不要）/ `redis`（高速・Horizon前提）/ `sqs`（マネージド）。`.env` の `QUEUE_CONNECTION` で切替。
- **Horizon（Redisキュー）**：`composer require laravel/horizon` → `php artisan horizon`。ダッシュボードでスループット・失敗・実行時間を可視化し、`config/horizon.php` でワーカー数を自動スケールできる。Redis 接続が前提。
- **失敗ジョブの運用**：`php artisan queue:failed-table && migrate` で `failed_jobs` を作る。失敗一覧は `queue:failed`、再実行は `php artisan queue:retry all`（または ID 指定）、削除は `queue:flush`。
- **キュー分割で優先度制御**：`->onQueue('high')` と `--queue=high,default` を組み合わせ、緊急ジョブが低優先に埋もれないようにする。
- **本番はプロセス管理**：`queue:work` を Supervisor / systemd で常駐＆自動再起動。`--max-time=3600` などで定期的にプロセスを入れ替えメモリリークを防ぐ。

## ハマりどころ / アンチパターン
- **デプロイ後に `queue:restart` を忘れる（最頻）**：`queue:work` はコードをメモリに読み込んだまま動き続けるため、デプロイしても**古いコードのまま実行**される。デプロイの最後に `php artisan queue:restart` を必ず叩く（実行中ジョブ完了後に安全に再起動）。
- **冪等性を考えない**：リトライ・重複dispatchで同じジョブが2回走り得る。処理済みフラグやユニーク制約、`ShouldBeUnique` で二重実行を吸収する。
- **Eloquent本体を渡したつもりで「鮮度」を誤解**：`SerializesModels` によりモデルは**識別子だけがシリアライズ**され、実行時にDBから取り直される。なので積んだ瞬間の値ではなく**実行時点の最新**で動く。途中で削除されると復元失敗（`ModelNotFoundException`）になり得る。
- **トランザクション内で dispatch**：コミット前にワーカーが走り「まだDBに無いレコード」を引いて失敗する。`dispatch(...)->afterCommit()` か `config/queue.php` の `after_commit => true` で回避。
- **ワーカー未起動**：`sync` 以外は `queue:work` の常駐が必須。起動し忘れると「ジョブが積まれるだけで永遠に実行されない」。
- **失敗を握り潰す**：`tries` も `failed()` も設けないと「静かに消えるジョブ」になる。失敗監視（Horizon/通知）を必ず持つ。

## 関連
[mail.md](./mail.md) / [artisan.md](./artisan.md) / [pitfalls.md](./pitfalls.md)
