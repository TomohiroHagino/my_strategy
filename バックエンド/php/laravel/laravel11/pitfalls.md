# 実務でハマる罠まとめ（Pitfalls）（Laravel 11）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Laravelは「便利な書き方」と「事故る書き方」が紙一重なものが多い。症状から該当箇所へ素早く飛ぶための索引。
- **Laravel 11 は骨格が変わった**（`Kernel.php` 廃止・`bootstrap/app.php` 集約）ため、旧バージョンの記憶でハマりやすい。

## DB / Eloquent
- **N+1問題**：一覧で関連を1件ずつ引きSQLが `1+N` 回。`Model::with('relation')` でまとめ読み（Eager Loading）。`Model::preventLazyLoading()` で開発中に検出。→ [database.md](./database.md)
- **`migrate:fresh` はデータ全消し**：全テーブルをdropして再構築する。本番では絶対に実行しない。差分適用は `migrate`、巻き戻しは `migrate:rollback`。→ [database.md](./database.md)
- **`insert` / `update`（クエリビルダ）はイベント非実行**：Eloquentの `created`/`saving` 等のモデルイベント・キャストが走らない。整合性は自前で担保。→ [database.md](./database.md)
- **マイグレーションとコードの不整合**：消したカラムをコードが参照したまま、で実行時エラー。破壊的変更は段階移行（追加→移行→後で削除）。→ [database.md](./database.md)

## モデル / 設計
- **mass assignment 例外**：`$fillable` 未設定で `create($request->all())` すると `MassAssignmentException`。許可属性を `$fillable` に明示する（`$guarded=[]` の全許可は危険）。→ [model.md](./model.md) / [security.md](./security.md)
- **Fat Model / Fat Controller**：分岐・計算・複数モデルにまたがる手続きが膨らんだら Service クラス / Action へ切り出す。→ [model.md](./model.md) / [controller.md](./controller.md)
- **アクセサ/キャストでHTTP都合を持ち込む**：表示整形はビュー（Blade）やリソースへ。モデルは永続化の責務に集中。→ [blade.md](./blade.md)

## コントローラ / リクエスト
- **バリデーション漏れ**：`$request->all()` をそのまま保存すると未検証データが入る。`$request->validate()` / フォームリクエストの `validated()` を使う。→ [validation.md](./validation.md)
- **認可忘れ**：ルートに到達した全員が操作可能になる。`$this->authorize()` や Policy で権限チェック。→ [auth.md](./auth.md)

## 設定 / 環境（Laravel 11 で特に注意）
- **`config:cache` 後にコード内 `env()` が効かない**：キャッシュ後は `.env` が読まれず `env()` は `null` を返す。設定値は**必ず `config()` 経由**で参照し、`env()` は `config/*.php` の中だけで使う。→ [config_env.md](./config_env.md)
- **APP_KEY 未設定**：暗号化・署名Cookie・セッションが動かず例外。デプロイ先で `php artisan key:generate`。→ [config_env.md](./config_env.md) / [security.md](./security.md)
- **`.env` をコミット**：シークレット全露出。コミットは `.env.example` のみ、`.env` は `.gitignore`。→ [security.md](./security.md)

## ルーティング / ミドルウェア（Laravel 11 の構成変更）
- **ミドルウェア/例外/スケジューラの登録場所が変わった**：`app/Http/Kernel.php`・`app/Console/Kernel.php` は廃止。ミドルウェアと例外は **`bootstrap/app.php`** の `withMiddleware` / `withExceptions`、スケジュール定義は **`routes/console.php`** に書く。旧記事のとおり `Kernel.php` を探しても無い。→ [middleware.md](./middleware.md)
- **`route:cache` とクロージャルート非対応**：`Route::get('/x', function(){...})` はキャッシュできず `route:cache` が失敗する。本番でキャッシュするなら**コントローラクラス参照**にする。→ [routing.md](./routing.md)

## 非同期 / 運用
- **デプロイ時の `queue:restart` 忘れ**：ワーカーは古いコードをメモリに抱えたまま動き続ける。コード更新後は **`php artisan queue:restart`** で再起動させる（Supervisor等が拾い直す）。→ [queues.md](./queues.md)
- **同期ドライバのまま本番**：`QUEUE_CONNECTION=sync` だとジョブが即時同期実行されレスポンスが遅延。`database` / `redis` に切り替える。→ [queues.md](./queues.md)

## ファサード / DI
- **ファサード乱用で依存が隠れる**：`Cache::`/`DB::` を多用するとクラスの依存が引数に現れずテスト/把握しづらい。コアロジックはコンストラクタ注入（DI）に寄せる。→ [facades.md](./facades.md)

## セキュリティ
- **`{!! !!}` 乱用**：ユーザ入力の生HTML出力はXSS。`{{ }}` で自動エスケープ、必要時のみサニタイズ。→ [security.md](./security.md) / [blade.md](./blade.md)
- **`DB::raw` / `whereRaw` への文字列連結**：SQLインジェクション。バインディング（`?`）を使う。→ [security.md](./security.md) / [database.md](./database.md)
- **`@csrf` 付け忘れ**：手書きフォームで `419 Page Expired`。フォームには必ず `@csrf`。→ [security.md](./security.md)

## 関連
[database.md](./database.md) / [model.md](./model.md) / [controller.md](./controller.md) / [config_env.md](./config_env.md) / [middleware.md](./middleware.md) / [routing.md](./routing.md) / [queues.md](./queues.md) / [facades.md](./facades.md) / [security.md](./security.md) / [validation.md](./validation.md) / [auth.md](./auth.md)
