# ルーティング（Routing）（Rails 4）

## ひとことで言うと
**「どのURL＋HTTPメソッドを、どのコントローラ#アクションにつなぐか」を定義する対応表**。`config/routes.rb` に書く。

## 役割・なぜ必要か
- リクエストが最初に通る「受付」。URL設計＝アプリのインターフェース設計そのもの。
- 規約に沿った `resources` を使えば、CRUDに必要な7本のルートとURLヘルパー（`posts_path` 等）が一気に揃う。

## 基本の書き方（コード）
```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "home#index"                       # トップページ（4では root "home#index" 記法が使える）

  resources :posts do                     # 7アクション（index/show/new/create/edit/update/destroy）
    member     { post :like }             # /posts/:id/like
    collection { get  :search }           # /posts/search
    resources :comments, only: %i[create destroy]  # ネスト
  end

  namespace :admin do                     # /admin/... ＋ Admin:: モジュール
    resources :users
  end

  get "/about", to: "pages#about"         # 個別ルート（4では get "/about" => "pages#about" も可）
end
```

### URLヘルパー（規約で生える）
- `posts_path` → `/posts`、`post_path(@post)` → `/posts/1`、`edit_post_path(@post)`、`new_post_path`
- `link_to "詳細", @post` のようにモデルを渡すだけでも解決される。

## 実務での使い方・定番パターン
- **基本は `resources`**。RESTに乗せると考えることが減る。
- 一覧/個別に対する追加アクションは **collection / member** で。
- 管理画面は **namespace**、APIは `namespace :api do namespace :v1 ...` でバージョニング。
- `constraints` でサブドメインや書式制限。
- 確認は **`rake routes`**（Rails 4 は `rake routes` が主流。`rails routes` は 5系から）。

## ハマりどころ / アンチパターン
- **`match` の素の使用が非推奨**：`match "/x" => "y#z"` は `via:` 無しだとエラー。`get`/`post` など動詞を明示する（Rails 4 で厳格化）。
- **ルートの順序**：先に書いたものが優先。ワイルドカード（`get "*path"`）を上に置くと他が死ぬ。
- **ネストの深さ**：3階層以上はURLが長く脆くなる。2階層まで＋`shallow: true` を検討。
- **名前付きルートの取り違え**：`post_path` と `posts_path`（単数/複数）の混同。
- HTTPメソッド違い（`get` で破壊的操作を作らない。削除は `delete`）。`jquery_ujs` 経由のDELETEは `data: { method: :delete }`。→ [javascript.md](./javascript.md)

## 関連
[controller.md](./controller.md) / [view.md](./view.md)
