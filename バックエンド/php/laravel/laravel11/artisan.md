# Artisan / Tinker（Laravel 11）

## ひとことで言うと
`artisan` は Laravel に付属する **CLI コマンドランナー**（`php artisan ...`）。コード生成・マイグレーション・キャッシュ操作・自作コマンド実行など、開発と運用の作業をコマンド一発で回す。`php artisan tinker` はその中の **アプリ本体を読み込んだ REPL（対話環境）**で、Eloquent やサービスをそのまま叩いて確認できる（Rails の `rails console` 相当）。

## 役割・なぜ必要か
- 「モデルを1個作る」「テーブルを追加する」「設定をキャッシュする」といった定型作業を、手作業ではなく**再現可能なコマンド**にするためにある。
- Tinker は「このクエリで何件返る？」「この値で `save()` は通る？」を、画面やコードを書かずに**その場で試す**ための日常ツール。
- スケジュール実行（cron 相当）も artisan の `schedule:run` を軸に回すため、バッチ運用の中心でもある。

## 基本の書き方（コード）
```bash
# プロジェクト直下で実行（php artisan が基本形）
php artisan list                 # 使えるコマンド一覧
php artisan make:model Post -mfc  # モデル+マイグレーション(m)+ファクトリ(f)+コントローラ(c)を一括生成
php artisan make:controller PostController --resource
php artisan migrate              # マイグレーション実行
php artisan migrate:rollback     # 直近バッチを取り消し
php artisan migrate:fresh --seed # 全テーブル作り直し+シード（開発用・破壊的）
php artisan route:list           # ルート一覧（--path=posts で絞り込み）
php artisan tinker               # REPL 起動
```
```php
// tinker 内：アプリのコードをそのまま呼べる
> App\Models\User::count();                 // 件数
> $u = App\Models\User::firstWhere('email', 'a@b.c');
> $u->posts()->latest()->limit(3)->get();   // 関連もそのまま
> User::factory()->count(3)->create();      // ファクトリでデータ生成
```

## 実務での使い方・定番パターン
- **生成系 `make:*`** で雛形を作る。`make:model Post -mfc` のようにオプションを合わせると関連ファイルを一括生成できて速い。`make:request`（フォームリクエスト）/ `make:job` / `make:mail` / `make:test` も頻出。
- **キャッシュ系（本番デプロイの定番）**：
  ```bash
  php artisan config:cache    # config をまとめてキャッシュ（高速化）
  php artisan route:cache     # ルートをキャッシュ
  php artisan view:cache      # Blade をコンパイル
  php artisan optimize        # 上記をまとめて実行
  php artisan optimize:clear  # 全キャッシュ破棄（設定変更が効かない時はこれ）
  ```
- **カスタムコマンド**：定型バッチは自作する。
  ```bash
  php artisan make:command SendReminders
  ```
  ```php
  // app/Console/Commands/SendReminders.php
  class SendReminders extends Command
  {
      protected $signature = 'app:send-reminders {--limit=100}';
      protected $description = 'リマインドメールを送る';

      public function handle(): int
      {
          $limit = (int) $this->option('limit');
          $this->info("送信開始: 最大 {$limit} 件");
          // 処理本体
          return self::SUCCESS;   // 0 を返すと成功扱い
      }
  }
  ```
- **スケジューラ（Laravel 11 で場所が変わった）**：旧 `app/Console/Kernel.php` の `schedule()` は**廃止**。**`routes/console.php` に `Schedule` ファサードで登録**する。
  ```php
  // routes/console.php
  use Illuminate\Support\Facades\Schedule;

  Schedule::command('app:send-reminders --limit=50')->dailyAt('09:00');
  Schedule::command('queue:prune-batches')->daily();
  Schedule::call(fn () => cache()->flush())->weekly();  // クロージャも可
  ```
  サーバ側の cron は **この1行だけ**を毎分回せばよい（個別ジョブの cron は書かない）：
  ```cron
  * * * * * cd /path/to/app && php artisan schedule:run >> /dev/null 2>&1
  ```
- **ワンライナー実行**：コンソールを開かず1行流すなら `php artisan tinker --execute="echo User::count();"`。

## ハマりどころ / アンチパターン
- **本番 tinker での直接データ変更**：`$u->delete()` / `Model::truncate()` / 一括 `update()` は取り返しがつかない。本番では原則「読むだけ」。変更が要るなら対象を `find` で確定し件数を確認してから、できればトランザクションで囲む（`DB::transaction(...)`）。
- **スケジューラの cron が1行で済むのを知らない**：個別ジョブごとに crontab を書いてしまう事故。Laravel では `schedule:run` の1行が**毎分**回り、定義は `routes/console.php` 側で完結する。逆にこの1行を cron に入れ忘れると、`Schedule::command` を書いても**永久に動かない**。
- **`routes/console.php` でなく旧 Kernel を探す**：Laravel 10 以前の記事をなぞると `app/Console/Kernel.php` を探して見つからず混乱する。11 では存在しない。
- **本番で `config:cache` 後に `.env` を変えても効かない**：config キャッシュ中は `.env` が読まれない。設定変更後は必ず `config:clear`（または `optimize:clear`）。逆に開発中に `config:cache` したまま忘れて「設定が反映されない」も定番。
- **`migrate:fresh` を本番で叩く**：全テーブル DROP → 再作成で**データ全消失**。本番は `migrate` のみ。`fresh` / `--seed` は開発・テスト専用と割り切る。
- **カスタムコマンドの戻り値**：`handle()` は終了コードを返す前提。失敗時に `self::FAILURE`（非ゼロ）を返さないと、cron やCIが「成功」と誤判定する。

## 関連
[getting_started.md](./getting_started.md) / [database.md](./database.md) / [queues.md](./queues.md)
