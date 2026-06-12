# セキュリティ（Security）（CodeIgniter 4）

## ひとことで言うと
Webの定番攻撃（**CSRF / XSS / SQLインジェクション**）への、**CI4が用意する防御の使い方**。Railsと違い「自動エスケープが既定で効く」わけではないので、`esc()` を**自分で呼ぶ**前提を理解するのが要。

## 役割・なぜ必要か
- 外部から来る入力（`$this->request` の GET/POST、ヘッダ、Cookie、JSON ボディ）は**信用しない**前提で扱う。
- CI4 はCSRFトークン・暗号化・Query Builderのバインドなど道具を持つが、**XSSエスケープはビューで明示的に呼ばないと効かない**。どこが守られ、どこを自分で守るかを知るのが目的。

## 基本の書き方（コード）
```php
// 1) CSRF: app/Config/Filters.php で 'csrf' フィルタを有効化
public array $globals = [
    'before' => [
        'csrf', // ← これで POST/PUT/DELETE にトークン検証がかかる
    ],
];
// app/Config/Security.php で tokenName / cookieName / regenerate などを調整
```
```php
// CSRF: ビューでトークンを埋め込む（form_open / csrf_field）
<?= form_open('/profile/update') ?>   <!-- 自動でトークン hidden が入る -->
    <input name="name" value="<?= esc($name) ?>">
<?= form_close() ?>

// 手書き <form> なら csrf_field() を必ず入れる
<form method="post" action="/profile/update">
    <?= csrf_field() ?>
</form>
```
```php
// 2) XSS: esc() は「自動でない」。出力時に必ず呼ぶ。コンテキスト指定が肝
<?= esc($user['name']) ?>                 <!-- 既定は 'html' コンテキスト -->
<?= esc($user['name'], 'html') ?>         <!-- 明示。本文・属性向け -->
<a href="<?= esc($url, 'attr') ?>">link</a>   <!-- HTML属性内 -->
<script>var n = <?= esc($name, 'js') ?>;</script>  <!-- JS文字列内 -->
<style>color: <?= esc($c, 'css') ?>;</style>       <!-- CSS値内 -->
<a href="<?= esc($u, 'url') ?>">u</a>     <!-- URLパラメータ -->
```
```php
// 3) SQLインジェクション: Query Builder はバインド済み（安全）
$users = $db->table('users')
            ->where('email', $this->request->getPost('email')) // 自動エスケープ
            ->get()->getResultArray();

// 生クエリ query() は「手動バインド（?）」で渡す。文字列連結は厳禁
$db->query('SELECT * FROM users WHERE email = ?', [$email]);   // OK
// $db->query("SELECT * FROM users WHERE email = '$email'");   // NG: SQLi
```
```php
// 4) 暗号化: services の encrypter を使う（鍵は .env に）
$encrypter = \Config\Services::encrypter();
$cipher = $encrypter->encrypt('secret data');
$plain  = $encrypter->decrypt($cipher);
// encryption.key は .env: encryption.key = hex2bin:... を設定（コードに直書きしない）
```

## 実務での使い方・定番パターン
- **CSRF**：HTMLフォームは `form_open()` を使えばトークンが自動で入る。`csrf` フィルタを `$globals['before']` に入れて初めて検証が効く（入れ忘れると素通り）。AjaxはmetaタグまたはCookieのトークンをヘッダ（`X-CSRF-TOKEN`）に載せる。→ [filters.md](./filters.md)
- **XSS**：ビューの出力は**例外なく `esc()`**。「本文＝html」「属性＝attr」「`<script>`内＝js」「URL＝url」とコンテキストを正しく選ぶ。コンテキストを間違えると属性脱出などで穴が残る。→ [views.md](./views.md)
- **SQLi**：基本は Query Builder（`where()` / `like()` 等は自動バインド）。動的に組む必要があるときも `query($sql, [$bind])` のバインド配列を使う。`$db->escape()` は最終手段。
- **本番設定**：`.env` の `CI_ENVIRONMENT = production` を必ず設定。`development` のままだと**詳細エラー・スタックトレース・DB情報が画面に露出**する。→ [config_env.md](./config_env.md)
- **秘密情報**：DBパスワード・APIキー・`encryption.key` は `.env`（gitignore対象）か環境変数で管理。コードや `app/Config/*.php` に直書きしない。
- **Cookie / セッション**：本番は `Config\Cookie` で `secure = true` / `httponly = true` / `samesite = 'Lax'`。

## ハマりどころ / アンチパターン
- **`esc()` 忘れ**：CI4は自動エスケープ**しない**。素の `<?= $x ?>` はXSS直結。出力は必ず `esc()` を通す癖をつける。→ [views.md](./views.md)
- **`csrf` フィルタ未有効化**：`Config\Security` を設定しても、`Filters.php` の `before` に `'csrf'` を入れないと検証ゼロ。フォームにトークンがあっても無意味。→ [filters.md](./filters.md)
- **コンテキスト指定ミス**：HTML属性内・JS内・URL内で `'html'` のまま出すと脱出される。属性は `'attr'`、JSは `'js'` を選ぶ。
- **生クエリの文字列連結**：`query("... '$x'")` は即SQLi。必ず `?` バインド配列で渡す。Query Builder優先。
- **本番で `CI_ENVIRONMENT=development` のまま**：エラー画面に環境変数・DB接続情報・パスまで出て情報漏洩。デプロイ時の `.env` を確認。→ [config_env.md](./config_env.md)
- **`encryption.key` をコードにハードコード／コミット**：漏れたら鍵を再生成し再暗号化。鍵は `.env` 管理。
- **`getVar()` / `getGet()` の `FILTER_*` 過信**：入力フィルタは万能ではない。出力時の `esc()` を省略しない。

## 関連
[filters.md](./filters.md) / [views.md](./views.md) / [config_env.md](./config_env.md) / [validation.md](./validation.md) / [sessions.md](./sessions.md)
