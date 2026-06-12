# リクエストの流れ・各層は何を返すか（Sinatra 4）

## ひとことで言うと
Sinatra は **層が無い**。リクエストは `get "/x" do … end` の**ルートブロックに直行**し、そのブロックが**受付＋処理＋応答を1か所で兼ねる**。レスポンスは**ブロックの戻り値**（文字列、または `erb :view` が生成した HTML）。Rails のような Controller / Model / Repository の分離は最初から無い。「何を受け取り、何を返すか」を1枚で俯瞰する。

> **この版 = Sinatra 4 系**：Rack の上の薄い層。ORM は内蔵せず、DB が要れば ActiveRecord / Sequel を自分で足す。1ファイルの **classic** と、`Sinatra::Base` を継承する **modular** の2スタイル。

## 全体の流れ（図）
```
ブラウザ
   │ リクエスト（GET /posts/1, POST /posts …）
   ▼
[Rack ミドルウェア]  （任意：ログ・セッション・認証など）
   │
   ▼
[ルートマッチング]  HTTPメソッド + URL パターン → どのブロックか決める
   │              （上から順に最初に一致したルートを実行）
   ▼
[ルートブロック do … end]  params を受け取り → 処理 → 戻り値がレスポンスになる
   │   ├ DBが要れば自分で AR/Sequel を呼ぶ（層は無い）
   │   └ erb :view でテンプレートを描画 / 文字列をそのまま返す
   │
   ▼ ブロックの戻り値 = レスポンスボディ
[Rack]  ステータス・ヘッダ・ボディを組み立てて返す
   │ レスポンス（HTML / JSON / 文字列）
   ▼
ブラウザ
```

## 各層は「何を受け取り・何を返す」か

| 「層」 | 受け取る | 返す | 備考 |
|---|---|---|---|
| **Rack ミドルウェア** | Rack env（リクエスト） | env を加工して次へ / レスポンス | 任意。`use` で挿す |
| **ルートマッチング** | HTTPメソッド + URL | 「どのブロックを実行するか」 | 上から最初に一致したもの |
| **ルートブロック** | `params` / `request` / `session` | **ブロックの戻り値 = レスポンスボディ**（文字列 or `erb :view` の HTML） | 受付＋処理＋応答を兼ねる |
| **ビュー（ERB）** | ブロックが渡したインスタンス変数（`@post`） | **HTML 文字列** → ブロックの戻り値になる | `views/*.erb`。任意 |

- **ブロックの最後の式が戻り値＝レスポンス**。明示的に `return` してもよいが、ふつうは最後の式（文字列 or `erb`）がそのまま本文になる。
- **DB アクセスは Sinatra の責務外**：ブロック内で自分が呼ぶ。モデル層を作るかどうかも自由。

## コードで通して見る
```ruby
# classic スタイル（1ファイル app.rb）
require "sinatra"

# 1) GET：params を受け取り → 処理 → erb の HTML を返す
get "/posts/:id" do
  @post = Post.find(params[:id])   # DBは自分で呼ぶ（ORMは内蔵なし）
  erb :show                        # views/show.erb を描画 → その HTML が戻り値＝レスポンス
end

# 2) POST：フォーム値を受け取り → 保存 → リダイレクト
post "/posts" do
  post = Post.create(title: params[:title], body: params[:body])
  redirect "/posts/#{post.id}"     # redirect ヘルパーで応答
end

# 3) JSON を返す：戻り値の文字列がそのまま本文になる
get "/api/posts/:id" do
  content_type :json
  Post.find(params[:id]).to_json   # この文字列が戻り値＝レスポンスボディ
end
```
```erb
<%# views/show.erb：HTMLを生成（戻り値としてブロックに返る） %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>
```

modular スタイルなら同じことを `class App < Sinatra::Base; get "/" do … end; end` の中に書く。

## 実務での使い方・定番パターン
- **小さく保つ**：ルート数が少ないうちは classic（1ファイル）で十分。増えたら modular（`Sinatra::Base`）にしてクラス分割。
- **層は必要になってから足す**：肥大化したらモデル（AR/Sequel）・サービスクラス・ヘルパーを自分で切り出す。最初から作らない。
- **戻り値の型を意識する**：文字列ならそのまま本文、配列 `[status, headers, body]` を返せばステータス・ヘッダも制御できる。
- **JSON API は `content_type :json` + `.to_json`**：シリアライズして戻り値の文字列にする。
- **共通処理は `before`/`after` フィルタ**：認証・ヘッダ付与などをルートの外に出す。→ [rack_and_filters.md](./rack_and_filters.md)

## ハマりどころ / アンチパターン
- **ルートの順序ミス**：上から最初に一致したブロックが実行される。汎用パターン（`/*`）を先に書くと個別ルートに届かない。
- **戻り値を返し忘れる**：ブロックの最後が文字列／`erb`でないと空レスポンスになる。`puts` はログに出るだけで本文にならない。
- **巨大な1ブロック**：処理を全部ブロックに詰めると保守不能。ヘルパーやモデルに切り出す。
- **DB を内蔵だと思い込む**：ActiveRecord/Sequel を自分で require・接続しないと動かない。→ [data_access.md](./data_access.md)
- **`halt` の使い所**：途中で打ち切って即レスポンスを返したいときは `halt 404, "not found"`。例外で止めようとしない。

## 関連
[routing.md](./routing.md) / [request_response.md](./request_response.md) / [views.md](./views.md) / [data_access.md](./data_access.md) / [rack_and_filters.md](./rack_and_filters.md)
