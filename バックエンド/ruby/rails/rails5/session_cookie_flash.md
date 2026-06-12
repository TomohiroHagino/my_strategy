# session / cookie / flash（Rails 5）

## ひとことで言うと
**session** はリクエストをまたいでユーザごとに保持する一時データ（ログイン状態など）、**cookie** はブラウザに保存する小さなデータ、**flash** は「次の1リクエストだけ」表示するメッセージ（通知）。

## 役割・なぜ必要か
- HTTPはステートレス（リクエスト間で状態を覚えない）。ユーザを識別し続けるために session が要る。
- session の実体はブラウザの cookie に保存される（既定は CookieStore）。
- 「保存しました」のような操作直後の通知をリダイレクト先で1回だけ出すために flash が要る。

## 基本の書き方（コード）
```ruby
# session（既定は暗号化された cookie に保存）
session[:user_id] = user.id          # 保存
session[:user_id]                    # 取得
session.delete(:user_id)             # 削除（ログアウト）
reset_session                        # 全消去（ログイン直後の固定化対策に）

# cookie
cookies[:locale] = "ja"                                   # 通常
cookies.signed[:uid]   = user.id                          # 改ざん検知付き
cookies.encrypted[:token] = secret                        # 暗号化
cookies[:remember] = { value: "1", expires: 2.weeks.from_now }

# flash（次の1リクエストだけ）
redirect_to posts_path, notice: "作成しました"            # flash[:notice] の糖衣構文
redirect_to root_path, alert: "権限がありません"          # flash[:alert]
flash.now[:error] = "失敗しました"                        # render（同一リクエスト）用
```
```erb
<%# レイアウトでフラッシュを表示 %>
<% flash.each do |type, message| %>
  <div class="flash flash-<%= type %>"><%= message %></div>
<% end %>
```

## 実務での使い方・定番パターン
- **ログイン**：`session[:user_id] = user.id`、`current_user` は `User.find_by(id: session[:user_id])` で復元。→ [auth.md](./auth.md)
- **flash の種類**：`notice`（成功）/ `alert`（注意）/ 任意キー（`flash[:error]` 等）。レイアウトで一括表示。
- **`flash.now`** は `render` 時（リダイレクトしないとき）に使う。`redirect_to` には通常の `flash`。
- **`cookies.signed` / `cookies.encrypted`** で改ざん・盗み見に耐える値を保存（「ログイン保持」トークン等）。
- **APIモード**は通常 session を使わず**トークン認証**にする（cookie セッションに依存しない）。→ [api_mode.md](./api_mode.md)

## ハマりどころ / アンチパターン
- **CookieStore に大きなデータ**：cookie は約4KB上限。配列やオブジェクトを詰めすぎると `CookieOverflow`。大きいものはDB/Redisセッションへ。
- **session に機密を平文で**：CookieStore は署名/暗号化されるが、機密はそもそも入れない方針が安全。
- **flash の出しっぱなし**：`flash` は次の1リクエストで消える。`render` で出したいときに `flash`（`flash.now` でない）を使うと2回表示される。
- **ログイン後に `reset_session` を忘れる**：セッション固定化攻撃の余地が残る。ログイン直後に session を作り直す。→ [security.md](./security.md)
- **`flash.keep`** を知らずに、リダイレクト連鎖でメッセージが消える。

## 関連
[auth.md](./auth.md) / [security.md](./security.md) / [api_mode.md](./api_mode.md)
