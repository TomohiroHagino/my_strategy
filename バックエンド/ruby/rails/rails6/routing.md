# ルーティング（Rails 6）

## ひとことで言うと
`config/routes.rb` で「URLのHTTP動詞＋パス」を「どのコントローラのどのアクション」に対応させる。`resources` でCRUD7ルートを一括定義し、名前付きルートヘルパー（`posts_path` 等）でリンクやリダイレクト先を生成する。

## 役割・なぜ必要か
リクエストはまずルーティングで振り分けられ、該当アクションが無ければ 404 になる。リンクやフォーム送信先を文字列で書かず `_path` ヘルパーで生成することで、URL変更に強くなる。

## 基本の書き方（コード）
```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "posts#index"                       # GET "/" → PostsController#index

  resources :posts                         # index/show/new/create/edit/update/destroy の7ルート
  resource  :profile                       # 単数形：indexなし（自分1件用）

  resources :posts do
    resources :comments                    # ネスト：/posts/:post_id/comments
    member { post :publish }               # /posts/:id/publish → publish_post_path
    collection { get :search }             # /posts/search   → search_posts_path
  end

  get  "about", to: "pages#about"          # 任意の単発ルート
  post "login", to: "sessions#create"

  namespace :admin do                      # /admin/posts かつ Admin::PostsController
    resources :posts
  end
  scope path: "v1", module: "api" do       # URLは /v1 だがヘルパー名は変えない
    resources :posts
  end

  constraints(subdomain: "api") { resources :tokens }
end
```
```bash
bin/rails routes              # 全ルート一覧（Prefix/Verb/URI/Controller#Action）
bin/rails routes -g post      # "post" を含むルートだけ絞り込み
bin/rails routes -c posts     # PostsController のルートだけ
```

## 実務での使い方・定番パターン
- 基本は `resources :posts` でCRUDを賄い、足りない動作だけ `member`/`collection` で足す。
- ネストは1段までに抑え、深い階層は `shallow: true` でID付きルートだけ親から切り離す。
  ```ruby
  resources :posts, shallow: true do
    resources :comments        # show/edit/update/destroy は /comments/:id になる
  end
  ```
- APIは `namespace :api` でURLとコントローラ名前空間を揃える。
- リンク先・リダイレクトは `redirect_to post_path(@post)` や `link_to "一覧", posts_path`、URL文字列が必要なメール等は `_url`。

## ハマりどころ / アンチパターン
- ルートは上から評価される。`get "posts/new"` を `resources :posts` の後ろに置くと `:id="new"` の `show` に食われる。固定パスは先に書く。
- 動かないときはまず `bin/rails routes -g キーワード` で実際に生成されたルートと動詞を確認する。動詞ミス（GETのつもりがPOST必要等）が多い。
- `link_to ... method: :delete` は rails-ujs が必要（[view.md](./view.md) / [javascript.md](./javascript.md)）。JSが無効だとGETになり destroy されない。

## 関連
[controller.md](./controller.md) / [view.md](./view.md) / [getting_started.md](./getting_started.md) / [strong_parameters.md](./strong_parameters.md) / [pitfalls.md](./pitfalls.md)
