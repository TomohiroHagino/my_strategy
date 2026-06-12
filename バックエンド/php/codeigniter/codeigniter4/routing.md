# ルーティング（CodeIgniter 4）

## ひとことで言うと
**URL とコントローラのメソッドを対応づける仕組み**。`app/Config/Routes.php` に「どの URL が来たら、どのコントローラのどのメソッドを呼ぶか」を書く。

## 役割・なぜ必要か
- リクエストの入口。`GET /posts/5` のような URL を `PostController::show(5)` のような処理に振り分ける「交通整理」を担う。
- URL 設計（パーマリンク）を **コントローラのクラス名から独立**させられる。URL を変えても内部のクラス構成を保てる（逆もしかり）。
- どの URL がどこへ繋がるか **1 ファイルに集約**されるので、アプリ全体の入口を一覧できる（明示ルーティング）。CI4 では安全性のためこの明示方式が基本。

## 基本の書き方（コード）
```php
<?php
// app/Config/Routes.php
use CodeIgniter\Router\RouteCollection;

/** @var RouteCollection $routes */

// HTTP メソッド別に明示（推奨）
$routes->get('/',            'Home::index');
$routes->get('posts',        'PostController::index');
$routes->get('posts/(:num)', 'PostController::show/$1');   // 数字をそのまま引数に
$routes->post('posts',       'PostController::create');
$routes->put('posts/(:num)', 'PostController::update/$1');
$routes->delete('posts/(:num)', 'PostController::delete/$1');

// プレースホルダ
//   (:num)     … 数字のみ          /posts/12
//   (:segment) … 1 セグメント(/含まず) /users/taro
//   (:any)     … 残り全部(/含む)    /files/a/b/c
$routes->get('users/(:segment)', 'UserController::profile/$1');

// ルートグループ（共通プレフィックス + 共通フィルタ）
$routes->group('admin', ['filter' => 'auth'], static function ($routes) {
    $routes->get('dashboard', 'Admin\Dashboard::index');   // → /admin/dashboard
    $routes->get('users',     'Admin\UserController::index');
});

// 個別ルートにフィルタ指定
$routes->get('mypage', 'MyPage::index', ['filter' => 'auth']);
```

```bash
# 現在のルート定義を一覧表示（対応の確認に必須）
php spark routes
```

## 実務での使い方・定番パターン
- **HTTP メソッドを明示**（`get`/`post`/`put`/`delete`）して、同じ URL でもメソッドごとに処理を分ける（REST 的な設計）。
- **`(:num)` / `(:segment)`** でプレースホルダを使い、`$1`・`$2` の順でメソッド引数に渡す。型を絞れる `(:num)` を優先すると不正値を弾ける。
- **ルートグループ**でプレフィックス（`admin/`）と共通フィルタ（認証など）をまとめる。→ [filters.md](./filters.md)
- ルートに**名前**を付けて `route_to('routeName', $id)` で URL を生成すると、URL 変更に強くなる（`->get(..., ['as' => 'name'])`）。
- 変更後は必ず **`php spark routes`** で「URL → コントローラ::メソッド」の対応を確認する。

## ハマりどころ / アンチパターン
- **自動ルーティング(Improved) はデフォルト無効**。CI4 では URL から推測でコントローラを呼ぶ「自動ルーティング」が**既定でオフ**。セキュリティ上、明示ルートを書くのが推奨。有効化する場合も `setAutoRoute(true)` を**意識的に**書き、想定外メソッドの露出に注意する。
- **メソッド/コントローラの指定ミス**：`'PostController::show/$1'` の `/$1` 忘れで引数が渡らない、名前空間付きクラス（`Admin\Dashboard`）の指定漏れ。`php spark routes` で照合する。
- **プレースホルダの取り違え**：`(:segment)` は `/` を含まないので、複数階層を受けたい場合は `(:any)`。逆に何でも受けたくないなら `(:num)` で絞る。
- **ルート定義順**：先にマッチしたものが優先。広いパターン（`(:any)`）を上に書くと下の具体ルートに届かない。具体的なルートを先に。
- **`_remap` 任せの乱用**：コントローラ側の `_remap()` でルーティングを握ると、Routes.php を見ても実際の対応が分からなくなる。→ [controllers.md](./controllers.md)

## 関連
[controllers.md](./controllers.md) / [filters.md](./filters.md)
