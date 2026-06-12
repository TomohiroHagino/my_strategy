# アセット（Vite）（Laravel 11）

## ひとことで言うと
**Vite** は Laravel 11 標準のフロントエンド・ビルドツール。JS / CSS / 画像などのアセットを **開発中はHMR（ホットリロード）で即反映**、**本番は1つにバンドル＆ハッシュ付与**して配信する。Blade からは `@vite([...])` ディレクティブで読み込む。旧 Laravel Mix（webpack ラッパー）の後継で、Laravel 9.19 以降の新規プロジェクトは Vite が既定。

## 役割・なぜ必要か
- 素の `<script src>` / `<link>` だと、依存解決・トランスパイル・minify・キャッシュバスティング（ファイル名へのハッシュ付与）を手作業でやることになる。Vite が**まとめて面倒を見る**。
- 開発中は変更を保存した瞬間にブラウザへ反映（HMR）され、フィードバックが速い。
- 本番ではファイル名にハッシュが付く（`app-4f2a.js` 等）ため、**デプロイ後の古いキャッシュ問題**が起きにくい。

## 基本の書き方（コード）
```js
// vite.config.js（プロジェクト直下）
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,   // Blade を保存したらブラウザを自動リロード
        }),
    ],
});
```
```blade
{{-- resources/views/layouts/app.blade.php の <head> 内 --}}
<!DOCTYPE html>
<html>
<head>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    @yield('content')
</body>
</html>
```
```bash
npm install        # 初回：依存を入れる
npm run dev        # 開発サーバ起動（HMR・常駐させたまま開発）
npm run build      # 本番ビルド（public/build/ に成果物を出力）
```

## 実務での使い方・定番パターン
- **開発フロー**：別ターミナルで `php artisan serve` と `npm run dev` を**両方常駐**させる。`@vite` は `npm run dev` 中は Vite 開発サーバ（HMR）を、停止中は `public/build` のビルド済みファイルを自動で参照する。
- **本番デプロイ**：必ず `npm run build` を実行してから（または CI で実行して `public/build` を成果物に含めて）デプロイする。本番サーバで `npm run dev` は使わない。
- **画像・静的アセット**：JS/CSS から `import` した画像は Vite が処理する。`public/` 直下に置いた素材は `asset('images/logo.png')` でそのまま参照（Vite を通さない）。
  ```js
  // resources/js/app.js から参照する場合
  import logo from '../images/logo.png';
  ```
- **Tailwind と組み合わせる**：`resources/css/app.css` に `@tailwind` ディレクティブを書き、`npm run build` でパージ込みの最終CSSが出る。新規 Breeze/Jetstream はこの構成が既定。
- **Laravel Mix からの移行**：`webpack.mix.js` を捨て、`vite.config.js` を追加。Blade の `mix('/js/app.js')` を **`@vite('resources/js/app.js')`** に置換。`package.json` の `dependencies` から `laravel-mix` を外し `vite` / `laravel-vite-plugin` を入れる。`npm run dev/prod` を `npm run dev/build` に読み替える。
- **本番判定**：`@vite` は `public/build/manifest.json` を見て実ファイル名（ハッシュ付き）に解決する。CDN 配信なら `vite.config.js` の `base` を設定。

## ハマりどころ / アンチパターン
- **本番ビルド忘れ**：`npm run build` せずにデプロイすると `public/build/manifest.json` が無く、ページが **"Vite manifest not found"** で500になる。デプロイ手順に `npm ci && npm run build` を必ず入れる。
- **`@vite` ディレクティブ未記載**：レイアウトの `<head>` に `@vite([...])` を書き忘れると、JS/CSS が一切読み込まれず「スタイルが全く効かない」状態に。素の `<link>`/`<script>` で直書きしてもハッシュ解決されず壊れる。
- **manifest 不整合**：古い `public/build` が残ったまま新ビルドが一部だけ差し変わると、Blade が古いハッシュ名を参照して404。デプロイ時は `public/build` を**まるごと作り直す**（古い成果物を消してからビルド）。
- **`input` の指定漏れ**：`vite.config.js` の `input` に列挙していないエントリは `@vite` で呼んでも解決されない。エントリを増やしたら両方を揃える。
- **開発サーバのホスト/ポート問題**：Docker や Sail 越しだと HMR が繋がらないことがある。`server.host` を `0.0.0.0` にする、または `APP_URL` / Vite の `hmr.host` を環境に合わせる。
- **`npm run dev` を本番で常駐**：開発サーバはビルド最適化されておらず遅く不安定。本番は必ず `build` の成果物を静的配信する。

## 関連
[blade.md](./blade.md) / [getting_started.md](./getting_started.md)
