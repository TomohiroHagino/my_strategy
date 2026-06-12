# リクエストの流れ・各層は何を返すか（Rails 6）

## ひとことで言うと
1リクエストが **routes.rb → Controller#action → Model（Active Record）→ View** と降り、**ARレコードが逆向きに上がってくる**。Rails は既定で Service / Repository 層を持たず、**Model がデータ＋業務ロジック＋DBアクセスを兼ねる（Fat Model）**。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

> **この版 = Rails 6**：フロントは **Webpacker（`yarn add`）+ rails-ujs + Turbolinks 5**。Hotwire は無い。削除リンクは `method: :delete, data: { confirm: ... }`、失敗時の再描画は単に `render :new`（`status: :unprocessable_entity` は不要）。JSイベント初期化は `turbolinks:load`。

## 全体の流れ（図）
```
ブラウザ
   │ リクエスト（GET /posts/1, POST /posts …）
   ▼
[config/routes.rb]  URL+HTTPメソッド → どの Controller#action か決める（ディスパッチ）
   │
   ▼
[Controller#action]  params を受け取る／before_action でフィルタ／Model を呼ぶ
   │
   ▼
[Model（Active Record）]  業務ロジック＋DBアクセスを兼ねる（Repository層は無い）
   │
   ▼
  DB ──→ AR レコード（モデルのインスタンス）を返す ─┐
   ▲                                                │
[Model] が AR レコードを Controller に返す
   ▲
[Controller] が @post に詰めて View へ渡す（render / redirect / render json）
   │
   ▼
[View（app/views/*.html.erb）]  HTML を生成
   │ レスポンス（HTML / JSON）（Turbolinks 5 でページ遷移を高速化）
   ▼
ブラウザ
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す | 置き場所 |
|---|---|---|---|
| **routes.rb** | URL + HTTPメソッド | 「どの Controller#action か」の振り分け | `config/routes.rb` |
| **Controller** | `params`（フォーム値・クエリ・パス）| **HTML / JSON**（`render`/`redirect_to`）→ クライアントへ | `app/controllers/` |
| **Model（Active Record）** | id / 条件 / 属性 | **AR レコード**（DBの1行＝オブジェクト）→ Controllerへ | `app/models/` |
| **View** | Controller が渡したインスタンス変数（`@post`） | **HTML 文字列** → Controllerへ（レスポンスボディになる） | `app/views/` |

- **Model がデータ＋業務ロジック＋DBアクセスを兼ねる**のが Rails の既定（Fat Model）。Spring の Service/Repository に当たる層は最初は無い。
- 複雑化したら `app/services/`（Service オブジェクト）や Form オブジェクトを足して Model を痩せさせる。→ [service_form.md](./service_form.md)

## コードで通して見る
```ruby
# 1) config/routes.rb：URL → Controller#action を決める
Rails.application.routes.draw do
  resources :posts            # GET /posts/:id → PostsController#show など
end

# 2) Controller：params を受け取り → Model を呼び → View / JSON を返す
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])   # Model が AR レコードを返す
    # render "show"（既定で app/views/posts/show.html.erb）
  end

  def create
    @post = Post.new(post_params)
    if @post.save
      redirect_to @post              # 成功 → リダイレクト
    else
      render :new                    # Rails 6：失敗時はこれでOK（status指定は不要）
    end
  end

  private
  def post_params
    params.require(:post).permit(:title, :body)   # Strong Parameters
  end
end

# 3) Model：データ＋業務ロジック＋DB（Active Record）を兼ねる
class Post < ApplicationRecord       # ← Rails 5以降の基底クラス
  belongs_to :user                   # 関連（既定で必須）
  validates :title, presence: true   # 業務ルール（バリデーション）
  scope :published, -> { where.not(published_at: nil) }  # スコープもここ
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
- **肥大化したら Service / Form を足す**：複数モデルをまたぐ・外部API連携は `app/services/` に切り出す。→ [service_form.md](./service_form.md)
- **APIなら View を介さず `render json:`**：JSON 応答は Controller から直接シリアライズして返す。
- **JS追加は Webpacker（`yarn add`）**：importmap ではない。イベント初期化は `turbolinks:load` で行う。→ [webpacker.md](./webpacker.md)

## ハマりどころ / アンチパターン
- **Controller に DB クエリや業務ロジックを書き散らす**：層が崩れる。クエリ・ルールは Model（またはスコープ）へ。
- **Fat Model の行き過ぎ**：1モデルに数百行は保守不能。複数モデルにまたがる処理は Service へ。
- **Strong Parameters を通さず保存**：`params` をそのまま渡すと mass assignment 脆弱性。必ず `permit`。→ [strong_parameters.md](./strong_parameters.md)
- **N+1 クエリ**：一覧で関連を都度引くと遅い。`includes` で先読み（6.1 なら `strict_loading` でガード）。→ [active_record.md](./active_record.md)
- **Zeitwerk 由来の NameError**：ファイル名・ディレクトリ名と定数名が不一致だと読み込めない。→ [zeitwerk.md](./zeitwerk.md)
- **`turbo_method` を使う**：Rails 6 は rails-ujs。削除は `method: :delete` を使う。

## 関連
[routing.md](./routing.md) / [controller.md](./controller.md) / [model.md](./model.md) / [view.md](./view.md) / [active_record.md](./active_record.md) / [service_form.md](./service_form.md) / [webpacker.md](./webpacker.md)
