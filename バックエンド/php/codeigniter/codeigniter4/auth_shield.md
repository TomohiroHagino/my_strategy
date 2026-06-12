# 認証（CodeIgniter Shield）（CodeIgniter 4）

## ひとことで言うと
- **Shield = CodeIgniter 公式の認証・認可パッケージ**（`codeigniter4/shield`）。
- **認証（Authentication）= 誰か** を確かめ、**認可（Authorization）= やってよいか** をグループ／権限で判断する。
- セッション認証（Web画面）と トークン認証（API）の両方を提供する。

## 役割・なぜ必要か
- 認証は「ログイン・登録・パスワード管理・記憶（Remember me）」など定番機能の塊。自前実装はバグと脆弱性の温床なので、**実績ある公式パッケージに任せる**のが堅い。
- 認可（グループ・権限）まで内蔵しているため、「ログイン済みなら何でもできる」状態を避け、**操作ごとの可否**を一元管理できる。
- Filters と統合され、ルート単位で「未ログインを弾く」「APIトークンを要求する」を宣言的に書ける。

## 基本の書き方（コード）
### 導入（composer + spark セットアップ）
```bash
# 1. パッケージ導入
composer require codeigniter4/shield

# 2. セットアップ（Config と Routes、必要ファイルを生成）
php spark shield:setup

# 3. マイグレーション（users / auth_* テーブルを作成）
php spark migrate
```
セットアップで `app/Config/Auth.php` と `app/Config/AuthGroups.php`、認証ルートが追加される。

### ルートとフィルタ（session / tokens）
```php
// app/Config/Routes.php
// Web 画面：session フィルタでログイン必須にする
$routes->group('', ['filter' => 'session'], static function ($routes) {
    $routes->get('dashboard', 'Dashboard::index'); // 未ログインは /login へ
});

// API：tokens フィルタでアクセストークンを要求する
$routes->group('api', ['filter' => 'tokens'], static function ($routes) {
    $routes->get('me', 'Api\User::me');
});

// グループ／権限で絞る（認可）：group:admin / permission:users.manage
$routes->group('admin', ['filter' => 'group:admin'], static function ($routes) {
    $routes->get('users', 'Admin\Users::index');
});
```

### ログイン状態の取得・ログイン／ログアウト
```php
// 現在のユーザー（auth() ヘルパ経由）
if (auth()->loggedIn()) {
    $user = auth()->user();   // CodeIgniter\Shield\Entities\User
    $id   = auth()->id();
}

// 手動ログイン（資格情報チェック）
$credentials = ['email' => $email, 'password' => $password];
$result = auth()->attempt($credentials);
if (! $result->isOK()) {
    // 失敗（理由は $result->reason()）
    return redirect()->back()->with('error', 'ログインに失敗しました');
}
// ログアウト
auth()->logout();
```

### ユーザー・グループ・権限（認可）
```php
$user = auth()->user();

// グループ（ロール相当）
$user->addGroup('admin');         // グループ付与
$user->removeGroup('admin');
$inAdmin = $user->inGroup('admin');

// 権限（細かい操作単位。app/Config/AuthGroups.php で定義）
$user->addPermission('users.manage');
$can = $user->can('users.manage'); // 認可チェック

// コントローラ内での認可ガード
if (! auth()->user()->can('users.manage')) {
    throw new \CodeIgniter\Exceptions\PageNotFoundException();
}
```

### AuthGroups.php（グループと権限の定義）
```php
<?php
namespace Config;
use CodeIgniter\Shield\Config\AuthGroups as ShieldAuthGroups;

class AuthGroups extends ShieldAuthGroups
{
    public array $groups = [
        'admin' => ['title' => '管理者', 'description' => '全権限'],
        'user'  => ['title' => '一般ユーザー'],
    ];
    public array $permissions = [
        'users.manage' => 'ユーザー管理',
        'posts.edit'   => '投稿編集',
    ];
    // グループに権限を割り当て（admin に users.manage を付与）
    public array $matrix = [
        'admin' => ['users.*', 'posts.*'],
        'user'  => ['posts.edit'],
    ];
}
```

## 実務での使い方・定番パターン
- **Web は `session` フィルタ、API は `tokens` フィルタ**を使い分ける。両対応なら `chain`（複数フィルタ）も可。
- **認可はルートの `filter` で宣言**（`group:admin` / `permission:posts.edit`）し、画面の出し分けは `auth()->user()->can()` で行う。
- 新規登録・パスワードリセットは Shield 付属のコントローラ／ビューをそのまま使い、必要なら `vendor` の View を `app/` に publish して上書きカスタムする。
- 管理者の初期作成は `php spark shield:user create` などのコマンドで行う。
- 重要操作は**毎回 DB の権限を確認**（セッション内の古い権限を信用しない）。

## ハマりどころ / アンチパターン
- **セットアップ手順の抜け（最頻）**：`composer require` だけで動かそうとして失敗する。**`php spark shield:setup` → `php spark migrate`** まで実行しないと、Config もテーブルも無い。順番を守る。
- **認可を忘れて認証だけにする**：`session` フィルタで「ログイン必須」にしただけでは、ログイン済みなら誰でも管理画面に入れる。**`group:` / `permission:` フィルタ**や `can()` で必ず操作可否を絞る（IDOR 防止）。
- **フィルタの適用範囲ミス**：`Routes.php` のグループ外に置いたルートにフィルタが効かず素通りする。保護対象は必ずフィルタ付きグループ内に入れる。`app/Config/Filters.php` の登録漏れにも注意。
- **API に session フィルタを付ける**：トークン認証したいのにセッション前提のフィルタを使い、API クライアントが弾かれる。API は `tokens` を使う。
- **ビューで非表示にしただけで認可した気になる**：リンクを隠してもエンドポイントは叩ける。サーバ側（フィルタ or `can()`）で必ずチェック。
- **権限変更が即時反映されない**：グループ降格してもセッションが残ると旧権限のまま。重要操作前に再取得・再確認する。

## 関連
[filters.md](./filters.md) / [sessions.md](./sessions.md)
