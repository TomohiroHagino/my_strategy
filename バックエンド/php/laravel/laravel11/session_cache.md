# セッション / Cookie / キャッシュ（Laravel 11）

## ひとことで言うと
**セッション** = リクエストをまたいでユーザーごとに値を保持する箱。**Cookie** = ブラウザに持たせる小さな値。**キャッシュ** = 計算/取得結果を一時保存して高速化する仕組み。

## 役割・なぜ必要か
- HTTPはステートレス（リクエスト間で状態を覚えない）。「ログイン状態」「カート」「フラッシュメッセージ」をまたいで保持するために**セッション**が要る。
- **キャッシュ**は「毎回DBや外部APIを叩くと遅い」処理結果を一時保存して、レスポンスを速くするためにある。セッションが「ユーザーごとの状態」なのに対し、キャッシュは「みんなで共有する一時データ」。
- **Laravel 11 の既定セッションドライバは `database`**（旧来の `file` から変更）。設定値は `SESSION_DRIVER`。

## 基本の書き方（コード）
**セッション**（`session()` ヘルパ）
```php
session(['cart_id' => 42]);          // 書き込み
$id = session('cart_id');            // 読み取り
$id = session('cart_id', 0);         // 既定値つき
session()->forget('cart_id');        // 削除
session()->flush();                  // 全消去（ログアウト時など）

// フラッシュ（次の1リクエストだけ生きる）
return redirect('/posts')->with('status', '保存しました');
```
```blade
{{-- 受け取り側（ビュー） --}}
@if (session('status'))
  <div class="alert">{{ session('status') }}</div>
@endif
```

**Cookie**
```php
use Illuminate\Support\Facades\Cookie;

return response('ok')->cookie('theme', 'dark', 60); // 60分
$theme = $request->cookie('theme');
// Laravelの暗号化Cookieはデフォルトで自動暗号化される
```

**キャッシュ**（`Cache` ファサード）
```php
use Illuminate\Support\Facades\Cache;

// remember: あればキャッシュ、なければクロージャ実行→保存して返す（定番）
$users = Cache::remember('active_users', now()->addMinutes(10), function () {
    return User::where('active', true)->get();
});

Cache::put('key', 'value', 600);     // 秒で保存
$v = Cache::get('key', 'default');   // 取得
Cache::forget('key');                // 無効化（明示削除）
Cache::flush();                      // 全消去（注意）
```

## 実務での使い方・定番パターン
- **フラッシュ＋`old()`の流れ**：フォーム送信失敗 → エラーと入力値をフラッシュ → リダイレクト先で `$errors` / `old()` で復元、成功時は `->with('status', ...)` で通知。バリデーションと一体で使う。→ [validation.md](./validation.md)
- **ドライバの選び方**（`SESSION_DRIVER` / `CACHE_STORE`）：
  - `database`（11既定）… 追加サーバ不要、複数アプリサーバでも共有可。`php artisan make:session-table`／`make:cache-table` ＋ migrate が要る。
  - `redis` … 高速・大規模向け。本番でアクセス多いならこれ。
  - `file` … 単一サーバの開発用。複数サーバだとセッションが分散して破綻。
  - `array` … テスト用（永続化しない）。
- **`Cache::remember` を基本形に**：重いクエリ・外部API結果はまずこれで包む。キーは衝突しないよう `posts:popular:{$page}` のように名前空間化する。
- **キャッシュの無効化はイベント駆動で**：データ更新時に該当キーを `Cache::forget()`。モデルの `saved`/`deleted` イベントや Observer で呼ぶと消し忘れにくい。
- **タグ付きキャッシュ**（redis/memcachedのみ）：`Cache::tags(['posts'])->put(...)` → まとめて `Cache::tags(['posts'])->flush()`。

## ハマりどころ / アンチパターン
- **セッションドライバの設定不足**：11既定の `database` なのに**セッションテーブルが無い**と「保存されない／ログインが維持されない」。`php artisan make:session-table && php artisan migrate` が要る。`file` に戻すなら `.env` で `SESSION_DRIVER=file`。
- **キャッシュの無効化漏れ**：データを更新したのに `Cache::forget()` を呼ばず、**古い値が表示され続ける**。「DBは新しいのに画面が古い」はキャッシュ削除忘れの典型。TTLを短くするか、更新箇所で必ず消す。
- **キャッシュにモデルを丸ごと突っ込む**：`User`オブジェクトをシリアライズして保存→スキーマ変更でデシリアライズ失敗。**ID・配列・スカラー**を入れて、取り出してから再取得する方が安全な場面が多い。
- **セッションに巨大データ**：カート全件・検索結果を丸ごと入れると肥大化。IDだけ入れてDBから引く。
- **`Cache::flush()` を安易に**：全環境・全アプリのキャッシュを消す。本番で実行すると一斉にキャッシュミス（サンダリングハード）。原則 `forget` かタグで局所的に。

## 関連
[config_env.md](./config_env.md) / [validation.md](./validation.md) / [middleware.md](./middleware.md)
