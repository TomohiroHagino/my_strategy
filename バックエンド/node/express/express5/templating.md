# テンプレート（Templating）（Express 5）

## ひとことで言うと
サーバ側で **HTML を組み立てて返す層**。`res.render` ＋ テンプレートエンジン（**EJS** / Pug / Handlebars）で使う。ただし **API 主体（JSON を返すだけ）のアプリでは不要**。

## 役割・なぜ必要か
- **まず前提：API 主体なら View は要らない**。SPA（React/Vue 等）や外部クライアント向けは `res.json()` で JSON を返すだけでよく、テンプレートエンジンを入れない構成が今は普通。→ [request_response.md](./request_response.md)
- サーバが直接 HTML を返したいとき（管理画面・メール本文・SEO 重視の素朴なページ・小規模ツール）にテンプレートが効く。
- コントローラが用意したデータをテンプレートに渡し、サーバ側で HTML 文字列を生成して `res.render` で返す。

## 基本の書き方（コード）
```js
// app.js（EJS を使う最小例）
const express = require("express");
const path = require("path");
const app = express();

// テンプレートエンジンの設定（views ディレクトリと拡張子）
app.set("view engine", "ejs");
app.set("views", path.join(__dirname, "views"));

// 静的ファイル（CSS/JS/画像）は express.static で配信
app.use(express.static(path.join(__dirname, "public")));

app.get("/users/:id", (req, res) => {
  const user = { name: "太郎", bio: "<b>こんにちは</b>" };
  // views/user.ejs を描画してHTMLを返す
  res.render("user", { user });
});

app.listen(3000);
```
```html
<!-- views/user.ejs -->
<!DOCTYPE html>
<html lang="ja">
  <head><meta charset="utf-8"><link rel="stylesheet" href="/style.css"></head>
  <body>
    <!-- <%= %> は自動でHTMLエスケープ（XSS対策） -->
    <h1>こんにちは、<%= user.name %> さん</h1>

    <!-- <%- %> はエスケープしない＝生HTML注入。ユーザ入力には絶対使わない -->
    <p><%- user.bio %></p>

    <!-- 制御構文 -->
    <% if (user.name) { %>
      <p>ログイン中</p>
    <% } %>
  </body>
</html>
```
```js
// JSON だけ返す API 主体の構成（テンプレート不要・これが今の定番の一方）
app.get("/api/users/:id", (req, res) => {
  res.json({ id: req.params.id, name: "太郎" });
});
```

## 実務での使い方・定番パターン
- **エンジンの選択**：`EJS`（HTMLそのまま書ける・学習コスト低・最も無難）／`Pug`（インデント記法で簡潔だがHTMLと別物）／`Handlebars`（ロジックレス志向）。インストールするだけで `app.set("view engine", ...)` が効く（`npm i ejs` 等）。
- **layout / partial**：共通の枠（ヘッダ・フッタ）は partial に切り出す。EJS なら `<%- include('partials/header') %>`。レイアウト機能が要るなら `express-ejs-layouts` を足す。
- **静的アセットは `express.static`** に任せる（テンプレートと別管理）。`/public/style.css` を `app.use(express.static("public"))` で配信し、テンプレートからは `/style.css` で参照。
- **データ受け渡し**：`res.render("view", { ...locals })` の第2引数（locals）に渡したものだけテンプレートで使える。渡し忘れ＝undefinedで描画が崩れる。
- **API と HTML の混在を避ける**：基本はどちらか主軸を決める。HTML を返すルート群と JSON を返す `/api/*` を分け、責務を混ぜない。

## ハマりどころ / アンチパターン
- **そもそも要らないのに導入**：API 主体（SPA バックエンド・モバイル向け）なら View は不要。`res.json()` で足りる。まず構成を疑う。→ [request_response.md](./request_response.md)
- **エンジン未設定 / 未インストール**：`view engine` を `set` し忘れる、`npm i ejs` を忘れると `res.render` が `No default engine`／`Cannot find module` で落ちる。
- **`views` パスのズレ**：`app.set("views", ...)` を絶対パス（`path.join(__dirname, "views")`）にしないと、起動ディレクトリ次第で `Failed to lookup view` になる。
- **XSS（エスケープ忘れ）**：EJS の `<%- %>` や生の文字列連結はエスケープされず、ユーザ入力を入れると XSS になる。表示は `<%= %>`（自動エスケープ）を既定にし、生HTMLが必要なときだけサニタイズ後に限定。
- **テンプレートにロジック詰め込み**：分岐・整形を書きすぎると読めない。整形はコントローラ／ヘルパー関数側で済ませて、テンプレートは表示に専念。
- **静的ファイルを `res.render` で返そうとする**：CSS/画像は `express.static` の担当。テンプレートで配ろうとしない。

## 関連
[request_response.md](./request_response.md) / [project_structure.md](./project_structure.md) / [security.md](./security.md)
