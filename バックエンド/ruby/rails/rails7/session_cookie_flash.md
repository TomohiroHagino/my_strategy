# Cookie / Session / Flash（Rails 7）

## ひとことで言うと
HTTPは「1リクエストごとに記憶を失う」仕組み。その上でユーザーごとの状態を持ち回るための3つの道具が **cookie**（ブラウザに保存する小さなデータ）、**session**（リクエストをまたいで保持する状態）、**flash**（次の1リクエストだけ生きる一時メッセージ）。

## 役割・なぜ必要か
- **cookie**：ブラウザに保存される小データ（〜4KB程度）。サーバはレスポンスで「これを覚えておいて」と返し、次回リクエストで自動的に送り返してもらう。
- **session**：ログイン状態など「リクエストをまたいで覚えておきたい情報」を入れる箱。Rails 7 の既定は **暗号化された cookie にセッションを載せる**（`cookie_store`）。だからサーバ側にセッションDBを持たなくても動く。
- **flash**：「保存しました」「ログインが必要です」のような、**直後の1画面にだけ出したい一時メッセージ**。リダイレクト先で表示して消える。
- これらが無いと「ログインしたのに次のページで他人扱い」「保存後の通知が出せない」といった、状態を持てないアプリになってしまう。

## 基本の書き方（コード）
```ruby
# --- cookie（ブラウザに保存する小データ） ---
cookies[:locale] = "ja"                         # 平文cookie
cookies.signed[:user_token] = "abc"             # 改ざん検知つき
cookies.encrypted[:secret] = "..."              # 暗号化（中身も隠す）
cookies[:remember] = { value: "1", expires: 2.weeks.from_now, httponly: true }
cookies.delete(:locale)

# --- session（リクエストをまたぐ状態） ---
session[:user_id] = user.id     # ログイン時に保存
current = User.find_by(id: session[:user_id])  # 後続リクエストで取り出す
session.delete(:user_id)        # ログアウト
reset_session                   # セッション総入れ替え（権限昇格時に推奨）

# --- flash（次の1リクエストだけ生きるメッセージ） ---
redirect_to posts_path, notice: "保存しました"   # = flash[:notice]
redirect_to login_path, alert: "ログインが必要です" # = flash[:alert]

# render（リダイレクトせず同一リクエストで表示）するなら flash.now
flash.now[:alert] = "入力に誤りがあります"
render :new, status: :unprocessable_entity
```
```erb
<%# レイアウトでまとめて表示 %>
<% flash.each do |type, message| %>
  <div class="flash flash-<%= type %>"><%= message %></div>
<% end %>
```

## 実務での使い方・定番パターン
- **認証はsessionの定番用途**：ログイン時 `session[:user_id] = user.id`、`current_user` は `session[:user_id]` 起点で引く。→ [auth.md](./auth.md)
- **redirect_to には notice: / alert:** を渡すのが基本（内部的に flash へ入る）。
- **render のときは flash.now**：バリデーション失敗で同じフォームを再表示する場面など。
- **cookies.signed / encrypted を使う**：ユーザーに見せたくない・改ざんされたくない値は平文cookieに置かない。
- **「次の画面に渡すちょっとした値」は flash**、**「ずっと持ち回る状態」は session**、と用途で使い分ける。

## ハマりどころ / アンチパターン
- **sessionに大きいデータを入れる**：既定の `cookie_store` はcookieの4KB上限に縛られる。配列・モデルまるごとはNG。IDだけ入れて都度DBから引く。
- **flash と flash.now の混同**：`redirect_to` なら `flash[:x]`（=notice:/alert:）、`render` なら `flash.now[:x]`。逆にすると「リダイレクトで消える」「2画面ぶん出る」等の不具合に。
- **session を消す時に key削除だけで済ませる**：ログイン後など権限が変わる場面は `reset_session` でセッション固定化攻撃を防ぐ。
- **機密を平文cookieに置く**：`cookies[:token]` は誰でも読める。`signed` / `encrypted` を使う。
- **flashに長文・HTML**：表示時にエスケープを忘れるとXSS。整形はビュー側で。

## 関連
[auth.md](./auth.md) / [controller.md](./controller.md)
