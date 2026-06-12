# ビュー（View）（CodeIgniter 4）

## ひとことで言うと
HTMLを生成するテンプレート（`app/Views/*.php`）。コントローラから `view('名前', $data)` で呼び出し、`$data` の配列キーがそのまま変数として展開される。

## 役割・なぜ必要か
- MVC の「表示」を担当。コントローラはデータを用意するだけ、画面の見た目はビューに閉じ込めることで責務を分離する。
- CI4 のビューは**素の PHP**（独自テンプレ言語ではない）。ただし**レイアウト機構**・**View Cells**・**`esc()`** といった仕組みが乗っており、これらを理解しないと「共通レイアウトの使い回し」「XSS対策」が抜ける。

## 基本の書き方（コード）
```php
<?php
// コントローラ側：配列キー → ビュー内の変数
return view('users/show', [
    'user'  => $user,
    'title' => 'プロフィール',
]);
```
```php
<!-- app/Views/users/show.php -->
<h1><?= esc($title) ?></h1>
<p>名前：<?= esc($user['name']) ?></p>   <!-- ★必ず esc() -->

<!-- ループ -->
<ul>
<?php foreach ($posts as $p): ?>
    <li><?= esc($p['title']) ?></li>
<?php endforeach ?>
</ul>
```

### レイアウト（共通ガワの使い回し）
```php
<!-- app/Views/layouts/main.php（親レイアウト） -->
<!DOCTYPE html>
<html lang="ja">
<head><title><?= esc($title ?? 'My App') ?></title></head>
<body>
    <header>共通ヘッダ</header>
    <main>
        <?= $this->renderSection('content') ?>  <!-- 子の中身がここに入る -->
    </main>
    <footer>共通フッタ</footer>
</body>
</html>
```
```php
<!-- app/Views/users/index.php（子ビュー） -->
<?= $this->extend('layouts/main') ?>   <!-- 親を継承 -->

<?= $this->section('content') ?>       <!-- content 区画の中身を定義 -->
    <h1>ユーザー一覧</h1>
    <?php foreach ($users as $u): ?>
        <p><?= esc($u['name']) ?></p>
    <?php endforeach ?>
<?= $this->endSection() ?>
```
- `extend()` で親レイアウトを指定 → `section()`〜`endSection()` で埋める内容を書く → 親側の `renderSection('content')` に差し込まれる。Rails の `yield` 相当。

### include / View Cells
```php
<!-- 部分テンプレの取り込み（同じ $data を共有） -->
<?= $this->include('partials/nav') ?>

<!-- View Cells：ロジック付きの再利用部品（カート個数など） -->
<?= view_cell('App\Cells\CartCell::badge', ['userId' => 7]) ?>
```
```php
<?php
// app/Cells/CartCell.php
namespace App\Cells;
use CodeIgniter\View\Cells\Cell;

class CartCell extends Cell
{
    public function badge(int $userId): string
    {
        $count = (new \App\Models\CartModel())->countFor($userId);
        return "<span class='badge'>{$count}</span>";
    }
}
```

## 実務での使い方・定番パターン
- **レイアウトは1〜数個に集約**（`layouts/main`、`layouts/admin`）。全ページが `extend` する。
- **共通パーツは `include`**（ナビ・フッタ）。**動的なロジックを伴う部品は View Cells**（DBアクセスや計算が要るもの）。`include` は単純な差し込み、Cells は「小さなコントローラ付き部品」と覚える。
- **エラー表示の定番**：コントローラから `$validation->getErrors()` を渡し、ビューで `esc()` しつつ列挙。→ [validation.md](./validation.md)
- 複数 `section` を切れる（`content` 以外に `styles`・`scripts` を親で `renderSection` しておくと、子から CSS/JS を差し込める）。

## ハマりどころ / アンチパターン
- **`esc()` の付け忘れ＝XSS**。CI4 のビューは**自動エスケープではない**（生PHPだから）。ユーザー由来の値は必ず `esc($x)`。HTML/属性/JS/CSS で文脈が違うので `esc($x, 'attr')`・`esc($x, 'js')` を使い分ける。→ [security.md](./security.md)
- **`extend` した子ビューで `section` の外に書いた HTML は出力されない**。表示したい内容は必ず `section`〜`endSection` の中に入れる。
- **`include` と `extend` の混同**：`extend` は「自分が親に埋め込まれる側」、`include` は「他のビューを自分に取り込む側」。逆に使うと真っ白になる。
- **ビューにビジネスロジックを書く**（DBクエリ・複雑な計算）。重い処理はコントローラ／モデル／View Cell へ。ビューは表示整形に徹する。
- **`view()` の戻り値を `return` し忘れる**：コントローラでは `return view(...)` が基本。`echo` してから別レスポンスを返すと二重出力になる。

## 関連
[controllers.md](./controllers.md) / [security.md](./security.md) / [request_response.md](./request_response.md)
