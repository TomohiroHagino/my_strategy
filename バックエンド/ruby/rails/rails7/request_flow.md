# リクエストの流れ・各層は何を返すか（Rails 7）

## ひとことで言うと
1リクエストが **routes.rb → Controller#action → Model（Active Record）→ View** と降り、**ARレコードが逆向きに上がってくる**。Rails は既定で Service / Repository 層を持たず、**Model がデータ＋業務ロジック＋DBアクセスを兼ねる（Fat Model）**。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

> **この版 = Rails 7**：失敗時の再描画や削除リンクは **Turbo（Turbo Stream / Turbo Drive）** が前提。フォーム再送には `status: :unprocessable_entity`、削除は `data: { turbo_method: :delete }`。`render turbo_stream:` で画面の一部だけ差し替える。

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
   │ レスポンス（HTML / Turbo Stream / JSON）
   ▼
ブラウザ
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す | 置き場所 |
|---|---|---|---|
| **routes.rb** | URL + HTTPメソッド | 「どの Controller#action か」の振り分け | `config/routes.rb` |
| **Controller** | `params`（フォーム値・クエリ・パス）| **HTML / Turbo Stream / JSON**（`render`/`redirect_to`）→ クライアントへ | `app/controllers/` |
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
    respond_to do |format|
      format.html                    # HTML を返す
      format.json { render json: @post }  # JSON を返す
    end
  end

  def create
    @post = Post.new(post_params)
    if @post.save
      redirect_to @post              # 成功 → リダイレクト
    else
      render :new, status: :unprocessable_entity  # Turbo前提：失敗時はこのstatus
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
  def published? = published_at.present?  # 業務ロジックもここ
end
```
```erb
<%# 4) View（app/views/posts/show.html.erb）：HTMLを生成 %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>
```

## 実務での使い方・定番パターン
- **Controller は薄く**：「params を受けて・Model を呼んで・View/JSON を返す」だけにする。
- **業務ロジックは Model**：バリデーション・関連・スコープ・計算は Active Record に置く（Fat Model が既定）。
- **肥大化したら Service / Form を足す**：1アクションで複数モデルをまたぐ・外部API連携などは `app/services/` に切り出して Model を痩せさせる。→ [service_form.md](./service_form.md)
- **部分更新は Turbo Stream**：一覧の追加・削除など画面の一部だけ更新は `render turbo_stream:` で返す（Rails 7 の標準）。→ [hotwire.md](./hotwire.md)
- **APIなら View を介さず `render json:`**：`--api` 構成や JSON 応答は Controller から直接シリアライズして返す。

## ハマりどころ / アンチパターン
- **Controller に DB クエリや業務ロジックを書き散らす**：層が崩れる。クエリ・ルールは Model（またはスコープ）へ。
- **Fat Model の行き過ぎ**：1モデルに何百行も詰めると保守不能。複数モデルにまたがる処理は Service へ。
- **失敗時に `status:` を付け忘れる**：Rails 7 は Turbo 前提。`render :new` だけだと Turbo がフォームを再描画しないことがある → `status: :unprocessable_entity` を付ける。
- **Strong Parameters を通さず保存**：`params` をそのまま `Post.new` に渡すと mass assignment 脆弱性。必ず `permit` する。→ [strong_parameters.md](./strong_parameters.md)
- **N+1 クエリ**：一覧で関連を都度引くと遅い。`includes` で先読みする。→ [active_record.md](./active_record.md)

## 関連
[routing.md](./routing.md) / [controller.md](./controller.md) / [model.md](./model.md) / [view.md](./view.md) / [active_record.md](./active_record.md) / [service_form.md](./service_form.md) / [hotwire.md](./hotwire.md)
