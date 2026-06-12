# session / cookie / flash（Rails 4）

## ひとことで言うと
**session** はリクエストをまたいでユーザ単位の情報を保持する仕組み、**cookie** はブラウザに保存する小さなデータ、**flash** は次の1リクエストだけ表示するメッセージ置き場。

## 役割・なぜ必要か
- HTTPはステートレス（毎回独立）なので、ログイン状態など「リクエストをまたいで覚えておきたい情報」を保つために session/cookie がある。
- 「保存しました」のような**1回だけ出す通知**を redirect 先に渡すために flash がある。

## 基本の書き方（コード）

### session
```ruby
# 保存
session[:user_id] = user.id
# 取得
@current_user = User.find_by(id: session[:user_id])
# 削除（ログアウト）
session.delete(:user_id)        # 個別
reset_session                   # 全部（ログアウト時の定石。固定化攻撃対策）
```
- Rails 4 の既定は **CookieStore**（暗号化＋署名された cookie に格納）。`config/secrets.yml` の `secret_key_base` で署名/暗号化する。→ [config_secrets.md](./config_secrets.md)
- セッションを多用する/サイズが大きいなら Redis 等の外部ストアへ。→ [../周辺インフラ/redis.md](../周辺インフラ/redis.md)

### cookie
```ruby
cookies[:locale] = "ja"
cookies[:token]  = { value: "abc", expires: 1.week.from_now }
cookies.signed[:user_id]    = user.id    # 改ざん検知付き
cookies.encrypted[:secret]  = "..."      # 暗号化（4.0は encrypted 無し→signedを使う。encryptedは4.1〜）
cookies.delete(:locale)
```

### flash
```ruby
redirect_to posts_path, notice: "作成しました"     # flash[:notice] のショートカット
redirect_to posts_path, alert:  "失敗しました"     # flash[:alert]
flash[:info] = "任意キーも使える"
flash.now[:alert] = "render（遷移しない）時はこちら"  # 同一リクエストで表示
```
```erb
<%# layout 側で表示 %>
<% flash.each do |type, msg| %>
  <div class="flash-<%= type %>"><%= msg %></div>
<% end %>
```

## 実務での使い方・定番パターン
- **ログイン状態**は `session[:user_id]` に最小限のID。`current_user` は `helper_method` で公開。→ [auth.md](./auth.md)
- **flash と flash.now の使い分け**：`redirect_to` には `flash`（次リクエスト）、`render` には `flash.now`（同一リクエスト）。混同すると出ない/二重に出る。
- **ログイン直後に `reset_session`**：セッション固定化攻撃を防ぐ定石。→ [security.md](./security.md)
- **`cookies.signed` / `cookies.encrypted`** で改ざん・盗み見を防ぐ（生 `cookies[]` に機微情報を入れない）。

## ハマりどころ / アンチパターン
- **CookieStore の 4KB 制限**：session に大きなオブジェクトを詰めると `CookieOverflow`。IDだけ入れてDBから引く。
- **session にオブジェクトを直接入れる**：シリアライズ事故の元。プリミティブ（ID・文字列）に留める。
- **flash と flash.now の取り違え**：`render` で `flash[...]` を使うと次の画面に出てしまう。
- **`secret_key_base` 変更で全 session 無効化**：鍵を変えると既存ログインが切れる（署名が合わなくなる）。
- **`cookies.encrypted` は 4.1〜**：4.0 では使えない。`cookies.signed` を使う。

## 関連
[auth.md](./auth.md) / [config_secrets.md](./config_secrets.md) / [security.md](./security.md) / [partial_layout.md](./partial_layout.md)
