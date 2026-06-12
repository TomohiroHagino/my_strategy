# ルーティング（Routing）（Rails 5）

## ひとことで言うと
**URL（とHTTPメソッド）を、どのコントローラ#アクションに渡すかを決める対応表**。`config/routes.rb` に書く。

## 役割・なぜ必要か
- 受け取ったリクエスト（例：`GET /posts/1`）を「`PostsController#show`、`params[:id]=1`」へ振り分ける入口。
- URL設計を1ファイルに集約することで、リンク生成（`post_path(@post)`）や認可の単位を一元管理できる。

## 基本の書き方（コード）
```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "posts#index"                       # GET /

  resources :posts do                      # CRUD 7アクション分を一括生成
    resources :comments, only: %i[create destroy]   # ネスト（/posts/1/comments）
    member { post :publish }               # /posts/1/publish
    collection { get :search }             # /posts/search
  end

  get  "about", to: "pages#about"          # 個別ルート
  namespace :admin do                      # /admin/... で AdminController 配下
    resources :users
  end
end
```
```bash
rails routes            # 定義済みルート一覧（path / コントローラ#アクション）
rails routes -g post    # post を含む行だけ絞り込み（5系のオプション）
```

## 実務での使い方・定番パターン
- **`resources`** でCRUDを宣言的に。必要なものだけ `only:` / `except:` で絞る。
- **ネスト**は1段まで。深くなると `shallow: true` で浅くする。
- **`namespace`**（URLもモジュールも分ける）/ **`scope`**（URLだけ分ける）/ **`scope module:`**（モジュールだけ分ける）を使い分ける。
- 生成される **path/url ヘルパー**（`posts_path` / `edit_post_path(@post)`）をビューやコントローラで使い、URL文字列を直書きしない。
- **API モード**でも routes.rb は同じ。`namespace :api do namespace :v1 do ... end end` でバージョニングするのが定番。→ [api_mode.md](./api_mode.md)
- **Action Cable** のWebSocketは `mount ActionCable.server => "/cable"` で繋ぐ。→ [action_cable.md](./action_cable.md)

## ハマりどころ / アンチパターン
- **ルート順**：上から評価される。広いルート（`get "*path"`）を上に置くと下が死ぬ。
- **`match` の乱用**：HTTPメソッドが曖昧になる。`get`/`post` を明示する。
- **過剰ネスト**：`/a/1/b/2/c/3` は扱いにくい。`shallow` か分割を検討。
- **path ヘルパー名の取り違え**：`as:` を付けたときの名前を `rails routes` で確認する。
- Rails 5 では `rake routes` でなく **`rails routes`** に統一された（rake も残るが rails 推奨）。

## 関連
[controller.md](./controller.md) / [api_mode.md](./api_mode.md) / [action_cable.md](./action_cable.md)
