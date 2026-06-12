# APIモード（Rails 5 特有）

## ひとことで言うと
ビュー（HTML）やセッション・Cookie前提のミドルウェアを省き、**JSONだけを返す軽量なRailsアプリ**を作るモード。`rails new myapp --api` で生成する。Rails 5 で正式に追加された。

## 役割・なぜ必要か
- フロントが別（React / Vue / モバイルアプリ）で、Railsは**JSON APIサーバとしてだけ使う**構成が増えたため。
- HTMLレンダリングやフォーム関連のミドルウェア（Cookie・flash・asset 等）を外すことで、**ミドルウェアが軽くなり起動・処理が速くなる**。
- 通常モードでも `render json:` でJSONは返せるが、APIモードは「最初から不要なものを外した雛形」を提供する。

## 通常モードとの違い
- **`ApplicationController` が `ActionController::API` を継承**（`ActionController::Base` ではない）。
- ビュー・ヘルパー・asset 関連の生成物が無い。
- **`protect_from_forgery`（CSRF対策）が無い**：トークン認証前提のため。Cookieセッションを使うなら自分で対策が要る。
- Cookie / flash / セッション関連のミドルウェアが既定で外れている（必要なら個別に戻せる）。

## 基本の書き方（コード）
```bash
rails new myapp --api -d postgresql -T
```
```ruby
# config/routes.rb（バージョニングが定番）
namespace :api do
  namespace :v1 do
    resources :posts
  end
end
```
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API   # ★ API 基底
end

# app/controllers/api/v1/posts_controller.rb
class Api::V1::PostsController < ApplicationController
  def index
    render json: Post.all
  end

  def create
    post = Post.new(post_params)
    if post.save
      render json: post, status: :created                            # 201
    else
      render json: { errors: post.errors.full_messages }, status: :unprocessable_entity  # 422
    end
  end

  private

  def post_params
    params.require(:post).permit(:title, :body)
  end
end
```

## 実務での使い方・定番パターン
- **URLバージョニング**（`/api/v1/...`）で後方互換を保ちながらAPIを進化させる。
- **JSON整形**：`jbuilder`（ビューで `index.json.jbuilder`）や **ActiveModelSerializers** / **fast_jsonapi** で構造を制御。`render json: post` の素返しは小規模向き。
- **認証はトークン**：`has_secure_token` / JWT / Devise Token Auth。session に依存しない。→ [auth.md](./auth.md)
- **エラー応答を統一**：`rescue_from ActiveRecord::RecordNotFound` で 404 JSON を返す等、レスポンス形式（成功/エラーの封筒）を揃える。
- **CORS**：別オリジンのフロントから叩くなら `rack-cors` gem で許可設定。
- **Strong Parameters はAPIでも必須**：JSONボディも `params` に入る。→ [strong_parameters.md](./strong_parameters.md)

## ハマりどころ / アンチパターン
- **CSRF対策の認識漏れ**：APIモードに `protect_from_forgery` は無い。**Cookieセッション認証をAPIで使う場合**は CSRF の穴になりうる。トークン認証なら問題なし。→ [security.md](./security.md)
- **後からHTMLを返したくなる**：APIモードはビュー前提が外れている。HTMLも返すなら通常モードで `render json:` を使い分ける方が素直。
- **CORS未設定**：ブラウザから別オリジンで叩くと CORS エラー。`rack-cors` を入れる。
- **エラー形式がバラバラ**：成功とエラーでJSON構造が違うとクライアントが辛い。封筒を統一する。
- **ページネーション無し**：`render json: Post.all` で全件返すとデータ増で破綻。`limit`/`offset` か kaminari/pagy を入れる。

## 関連
[controller.md](./controller.md) / [routing.md](./routing.md) / [auth.md](./auth.md) / [strong_parameters.md](./strong_parameters.md) / [security.md](./security.md)
