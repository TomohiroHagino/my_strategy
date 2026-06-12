# セキュリティ（Security）（Laravel 11）

## ひとことで言うと
Webアプリの定番攻撃（CSRF / mass assignment / XSS / SQLインジェクション / 認可漏れ / 秘密情報漏洩）に対する、**Laravelが標準で用意する防御の使い方**。多くは既定で有効だが、「無効化してしまう書き方」を避けるのがポイント。

## 役割・なぜ必要か
- 外部から来る入力（`$request`・ヘッダ・ボディ・ルートパラメータ）は**信用できない**前提で扱う必要がある。
- Laravelは安全側のデフォルト（Bladeの自動エスケープ・CSRFトークン・バインド済みクエリ・`$fillable`）を持つが、`{!! !!}` や `DB::raw` の文字列連結など**素朴な書き方で穴が開く**。どこが守られ、どこで自分が破りうるかを知るのが目的。

## 基本の書き方（コード）
```blade
{{-- 1) CSRF: HTMLフォームには @csrf 必須（隠しトークンを埋め込む） --}}
<form method="POST" action="/posts">
    @csrf
    <input name="title">
    <button>送信</button>
</form>

{{-- 2) XSS: {{ }} は自動エスケープされ安全。{!! !!} は生HTMLで危険 --}}
{{ $user->name }}                 {{-- HTMLエスケープされる（安全） --}}
{!! $user->bio !!}                {{-- NG: 生HTML注入の穴。サニタイズ必須 --}}
```

```php
// 3) mass assignment: $fillable で許可した属性だけ一括代入を通す
class User extends Model
{
    protected $fillable = ['name', 'email'];   // is_admin は通さない
    // 逆指定なら: protected $guarded = ['id', 'is_admin'];
}
User::create($request->all());   // $fillable 外のキーは黙って無視される

// 4) SQLインジェクション: Eloquent / クエリビルダはバインド済みで安全
User::where('email', $request->email)->first();        // OK（自動バインド）
DB::table('users')->where('name', $request->q)->get(); // OK
// DB::select("... WHERE name = '".$request->q."'");    // NG: 文字列連結
DB::select('... WHERE name = ?', [$request->q]);       // OK（プレースホルダ）
```

## 実務での使い方・定番パターン
- **CSRF**：`web` ミドルウェアグループの `VerifyCsrfToken` が POST/PUT/PATCH/DELETE を検証する。HTMLフォームは `@csrf`、JS の `fetch` は `<meta name="csrf-token">` 由来のトークンを `X-CSRF-TOKEN` ヘッダに載せる。**API は `routes/api.php`（CSRF対象外）** に置き、`Sanctum` のトークン/SPA認証で守るのが定石。→ [auth.md](./auth.md)
- **認可**：`Gate::allows('update-post', $post)` や **Policy**（`$this->authorize('update', $post)`）でリソース単位の権限を判定。一覧取得も `$request->user()->posts()` のように**ログインユーザのスコープ**で引き、他人のリソース操作を構造的に防ぐ。→ [auth.md](./auth.md)
- **入力検証**：すべての入力はフォームリクエスト/`$request->validate()` で**通る形だけに絞る**。検証を通った `$request->validated()` のみを保存に使う。→ [validation.md](./validation.md)
- **秘密情報**：APIキー・DBパスワード・`APP_KEY` はコードに書かず `.env` で管理し、`.env` は**コミットしない**（`.gitignore` 済み）。コミットするのは `.env.example` のみ。新規環境では `php artisan key:generate` で `APP_KEY` を発行する（暗号化・署名Cookie・セッションの基盤）。→ [config_env.md](./config_env.md)
- **静的解析**：`Larastan`（PHPStan拡張）をCIに組み込み、型不整合や未定義プロパティ参照を検出。あわせて `composer audit` で依存の既知脆弱性を確認する。
- **HTTPS / Cookie**：本番は `URL::forceScheme('https')` または前段プロキシでHTTPS強制。セッションCookieは `config/session.php` の `secure` / `http_only` / `same_site` を適切に。

## ハマりどころ / アンチパターン
- **`{!! !!}` の乱用**：ユーザ入力をそのまま生HTML出力するとXSS直結。整形済み安全文字列にだけ使い、ユーザHTMLは `HTMLPurifier` 等で**サニタイズ**してから。→ [blade.md](./blade.md)
- **`DB::raw` / 生SQLへの文字列連結**：`DB::raw("name = '$q'")` や `whereRaw("col = $input")` は即SQLインジェクション。`whereRaw('col = ?', [$input])` のように必ずバインディングを使う。→ [database.md](./database.md)
- **`$guarded = []`（全許可）**：mass assignment 保護を全面解除する書き方。`is_admin` 等を外部から書き込まれる穴になる。`$fillable` で明示許可するのが安全。→ [model.md](./model.md)
- **`@csrf` 付け忘れ**：手書き `<form>` でトークンを入れないと `419 Page Expired`。逆に CSRF を `VerifyCsrfToken::$except` で安易に除外すると、代替認証なしの穴になる。→ [middleware.md](./middleware.md)
- **`.env` / `APP_KEY` をコミット**：`.env` が漏れたら全鍵・全認証情報が露出。漏れたら `key:generate` で再生成し、関連シークレットもローテーションする。
- **APP_KEY 未設定**：暗号化・署名Cookie・セッションが機能せず例外。デプロイ先で必ず設定する。→ [config_env.md](./config_env.md)
- **オープンリダイレクト**：`redirect($request->input('url'))` は外部誘導の穴。許可リストや `url()->to()` で内部URLに限定する。
- **認可の `Gate`/`Policy` 未適用**：コントローラで `authorize` を呼ばないと、ルートに到達した全員が操作可能。コントローラ冒頭で必ず権限チェック。→ [auth.md](./auth.md)

## 関連
[validation.md](./validation.md) / [config_env.md](./config_env.md) / [blade.md](./blade.md) / [auth.md](./auth.md) / [model.md](./model.md) / [database.md](./database.md) / [middleware.md](./middleware.md)
