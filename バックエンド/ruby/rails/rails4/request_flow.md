# リクエストの流れ・各層は何を返すか（Rails 4）

## ひとことで言うと
1リクエストが **routes.rb → Controller#action → Model（Active Record）→ View** と降り、**ARレコードが逆向きに上がってくる**。Rails は既定で Service / Repository 層を持たず、**Model がデータ＋業務ロジック＋DBアクセスを兼ねる（Fat Model）**。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

> **この版 = Rails 4**：フロントは **jquery_ujs + Turbolinks Classic（3系）+ `*.js.erb`（SJR）**。Hotwire / Turbo / Stimulus は無い。モデルは **`ActiveRecord::Base` を直接継承**（`ApplicationRecord` は Rails 5から）。削除リンクは `method: :delete, data: { confirm: ... }`、Ajax 部分更新は `format.js` + `*.js.erb` で返す。

## 全体の流れ（図）
```
ブラウザ
   │ リクエスト（GET /posts/1, POST /posts …）
   ▼
[config/routes.rb]  URL+HTTPメソッド → どの Controller#action か決める（ディスパッチ）
   │
   ▼
[Controller#action]  params を受け取る／before_filter/before_action でフィルタ／Model を呼ぶ
   │
   ▼
[Model（ActiveRecord::Base）]  業務ロジック＋DBアクセスを兼ねる（Repository層は無い）
   │
   ▼
  DB ──→ AR レコード（モデルのインスタンス）を返す ─┐
   ▲                                                │
[Model] が AR レコードを Controller に返す
   ▲
[Controller] が @post に詰めて View へ渡す（render / redirect / respond_with）
   │
   ▼
[View（app/views/*.html.erb / *.js.erb）]  HTML / JS を生成
   │ レスポンス（HTML / JS / JSON）（Turbolinks Classic でページ遷移）
   ▼
ブラウザ
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す | 置き場所 |
|---|---|---|---|
| **routes.rb** | URL + HTTPメソッド | 「どの Controller#action か」の振り分け | `config/routes.rb` |
| **Controller** | `params`（フォーム値・クエリ・パス）| **HTML / JS / JSON**（`render`/`redirect_to`/`respond_with`）→ クライアントへ | `app/controllers/` |
| **Model（ActiveRecord::Base）** | id / 条件 / 属性 | **AR レコード**（DBの1行＝オブジェクト）→ Controllerへ | `app/models/` |
| **View** | Controller が渡したインスタンス変数（`@post`） | **HTML / JS 文字列** → Controllerへ（レスポンスボディになる） | `app/views/` |

- **Model がデータ＋業務ロジック＋DBアクセスを兼ねる**のが Rails の既定（Fat Model）。Spring の Service/Repository に当たる層は最初は無い。
- 複雑化したら `app/services/`（Service オブジェクト）や Form オブジェクトを足して Model を痩せさせる。→ [service_form.md](./service_form.md)

## コードで通して見る
```ruby
# 1) config/routes.rb：URL → Controller#action を決める
Blog::Application.routes.draw do
  resources :posts            # GET /posts/:id → PostsController#show など
end

# 2) Controller：params を受け取り → Model を呼び → View / JSON を返す
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])   # Model が AR レコードを返す
    respond_to do |format|
      format.html                    # app/views/posts/show.html.erb
      format.json { render json: @post }
    end
  end

  def create
    @post = Post.new(post_params)
    if @post.save
      redirect_to @post              # 成功 → リダイレクト
    else
      render :new                    # Rails 4：失敗時はこれでOK
    end
  end

  private
  def post_params
    params.require(:post).permit(:title, :body)   # Strong Parameters（Rails 4 で標準）
  end
end

# 3) Model：データ＋業務ロジック＋DB（Active Record）を兼ねる
class Post < ActiveRecord::Base      # ← Rails 4 は Base を直接継承（ApplicationRecord は無い）
  belongs_to :user                   # 既定では必須ではない（presence チェックは付かない）
  validates :title, presence: true   # 業務ルール（バリデーション）
end
```
```erb
<%# 4) View（app/views/posts/show.html.erb）：HTMLを生成 %>
<h1><%= @post.title %></h1>
<%= link_to "削除", @post, method: :delete, data: { confirm: "削除しますか?" } %>
```

## 実務での使い方・定番パターン
- **Controller は薄く**：「params を受けて・Model を呼んで・View/JSON を返す」だけにする。
- **業務ロジックは Model**：バリデーション・関連・スコープ・計算は Active Record に置く（Fat Model が既定）。
- **Ajax 部分更新は `*.js.erb`（SJR）**：`format.js` で対応する `create.js.erb` を返し、画面の一部を JS で差し替える（Rails 4 の定番）。→ [javascript.md](./javascript.md)
- **`respond_to` / `respond_with` を多用**：HTML/JSON/JS を1アクションで出し分ける。
- **肥大化したら Service / Form を足す**：複数モデルをまたぐ処理は `app/services/` へ。→ [service_form.md](./service_form.md)

## ハマりどころ / アンチパターン
- **Controller に DB クエリや業務ロジックを書き散らす**：層が崩れる。クエリ・ルールは Model（またはスコープ）へ。
- **mass assignment 対策の混同**：Rails 4 は `attr_accessible` ではなく **Strong Parameters**（`permit`）で守る。`attr_accessible` は gem 化されて本体から外れた。→ [security.md](./security.md)
- **`ApplicationRecord` を継承する**：Rails 4 には無い。基底は `ActiveRecord::Base` を直接継承する。
- **N+1 クエリ**：一覧で関連を都度引くと遅い。`includes` で先読みする。→ [active_record.md](./active_record.md)
- **Active Job を期待する**：4.2 で導入。4.0/4.1 では無く、Sidekiq/Resque/DelayedJob を直接叩く。

## 関連
[routing.md](./routing.md) / [controller.md](./controller.md) / [model.md](./model.md) / [view.md](./view.md) / [active_record.md](./active_record.md) / [service_form.md](./service_form.md) / [javascript.md](./javascript.md)
