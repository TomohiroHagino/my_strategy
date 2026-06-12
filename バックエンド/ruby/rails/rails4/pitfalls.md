# 実務でハマる罠まとめ（Pitfalls）（Rails 4）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Railsは「便利な書き方」と「事故る書き方」が紙一重なものが多い。さらに Rails 4 は古い記事（Rails 3 / 5+）の情報が混ざりやすいので、バージョン差で詰まる罠を集約する。

## バージョン差で詰まる（Rails 4 特有）
- **`attr_accessible` を書いて動かない**：Rails 4 は Strong Parameters が標準。許可はコントローラで `permit`。`attr_accessible` は `protected_attributes` gem が要る。→ [strong_parameters.md](./strong_parameters.md)
- **`ApplicationRecord` が無い**：モデルは `ActiveRecord::Base` を直接継承（`ApplicationRecord` は5〜）。→ [model.md](./model.md)
- **`belongs_to` が既定で任意**：関連先 nil でも保存できる（7では必須）。必須化は `presence: true` を明示。→ [model.md](./model.md)
- **`enum` が無い（4.0）**：`enum` は 4.1〜。4.0 では使えない。→ [model.md](./model.md)
- **`foreign_key: true` が無い（4.0/4.1）**：マイグレーションの外部キー制約は 4.2〜。→ [active_record.md](./active_record.md)
- **`form_with` が無い**：`form_for` / `form_tag` を使う（`form_with` は5.1〜）。→ [view.md](./view.md)
- **`rails routes` が無い**：`rake routes` を使う（`rails routes` は5系）。→ [routing.md](./routing.md)
- **credentials が無い**：秘密情報は `secrets.yml`（4.1〜）＋ ENV（credentials は5.2〜）。→ [config_secrets.md](./config_secrets.md)
- **Active Job が無い（4.0/4.1）**：4.2 で導入。それ以前は Sidekiq 等を直接。→ [active_job.md](./active_job.md)
- **system test が無い**：ブラウザE2Eは Capybara を自前導入（system test は5.1〜）。→ [testing.md](./testing.md)
- **`scope` は lambda 必須**：`scope :x, -> { ... }`。Rails 3 の `scope :x, where(...)` は不可。→ [active_record.md](./active_record.md)
- **`match` に動詞が必要**：`get`/`post` を明示しないとルートエラー。→ [routing.md](./routing.md)

## DB / Active Record
- **N+1問題**：一覧で関連を1件ずつ引きSQLが `1+N` 回。`includes` でまとめ読み。検出は bullet。→ [active_record.md](./active_record.md)
- **コールバック地獄**：`after_save` 等に副作用（メール送信・別モデル更新）を詰めると追跡不能。重い処理はService / (4.2〜)Active Jobへ。→ [active_record.md](./active_record.md) / [service_form.md](./service_form.md)
- **`save` と `save!`**：`save` は失敗時 `false` を返すだけ（握り潰し注意）、`save!` は例外。調査時は `save!`。→ [model.md](./model.md)
- **`default_scope` の副作用**：全クエリに暗黙の絞り込みが効き、予期せぬ結果に。基本使わない。→ [active_record.md](./active_record.md)
- **`update_all` / `delete_all`**：バリデーション・コールバック非実行。高速な代わりに整合性は自前で担保。→ [active_record.md](./active_record.md)
- **マイグレーション不可逆**：カラム削除等の破壊的変更は段階移行（追加→移行→後で削除）。`change` で戻せない操作は `up`/`down` を書く。→ [active_record.md](./active_record.md)
- **time zone**：`Time.now` でなく `Time.current`（`Time.zone.now`）。→ [active_record.md](./active_record.md)

## モデル / 設計
- **Fat Model / Fat Controller**：分岐・計算・複数モデルにまたがる手続きが膨らんだらService / Formオブジェクトへ。→ [service_form.md](./service_form.md)
- **早すぎる Concern 化**：似ているだけで共通化すると差分が出て破綻。重複が本物になってから。→ [concern.md](./concern.md)
- **モデルに表示/HTTPの都合を持ち込む**：整形はヘルパー/プレゼンターへ。→ [helper.md](./helper.md)

## コントローラ / リクエスト
- **Strong Parameters の `permit` 漏れ**：許可し忘れたキーは黙って無視＝「保存したのに入らない」。配列/ネストは `permit(tag_ids: [])`。→ [strong_parameters.md](./strong_parameters.md)
- **二重 render/redirect（`DoubleRenderError`）**：1アクション1応答。分岐後の `return` 忘れに注意。→ [controller.md](./controller.md)
- **認可忘れ**：`Post.find` は他人のも引ける。`current_user.posts.find` でスコープを効かせる。→ [controller.md](./controller.md)
- **`before_filter` を新規で書く**：4では非推奨。`before_action` に統一。→ [filters.md](./filters.md)

## フロント / JS / アセット
- **Turbolinks Classic で自前JSが初回しか動かない**：`$(document).ready` だけでなく `page:load` も仕掛ける（または `--skip-turbolinks`）。→ [javascript.md](./javascript.md)
- **遷移後にクリックが効かない**：直バインドでなく `$(document).on("click", セレクタ, fn)` のイベント委譲で。→ [javascript.md](./javascript.md)
- **`*.js.erb` が実行されない**：`respond_to { |f| f.js }` ＋ `remote: true` ＋ `.js` 形式リクエストを確認。`j`（escape_javascript）忘れにも注意。→ [javascript.md](./javascript.md)
- **本番でアセット404**：`rake assets:precompile` 忘れ／`precompile` リスト漏れ。`config.assets.compile=true` の本番有効化は激重。→ [assets.md](./assets.md)
- **JSランタイム不在**：`ExecJS::RuntimeUnavailable`。`therubyracer` か Node を入れる。→ [assets.md](./assets.md)

## セッション / メール
- **CookieStore 4KB 超過**：`CookieOverflow`。session にはIDだけ。→ [session_cookie_flash.md](./session_cookie_flash.md)
- **flash と flash.now の取り違え**：`render` 時は `flash.now`、`redirect_to` 時は `flash`。→ [session_cookie_flash.md](./session_cookie_flash.md)
- **`deliver_later` を 4.0/4.1 で使う**：Active Job が無くエラー。`deliver` か Sidekiq 直接。→ [action_mailer.md](./action_mailer.md)
- **メールリンクに `*_path`**：ホスト無しでリンク切れ。`*_url` ＋ `default_url_options`。→ [action_mailer.md](./action_mailer.md)

## セキュリティ
- **`where` に文字列展開**：`"#{params[...]}"` はSQLインジェクション。`?` / 名前付きプレースホルダに。→ [security.md](./security.md)
- **`raw` / `html_safe` 乱用**：ユーザ入力に付けるとXSS。不安なら `sanitize`。→ [security.md](./security.md)
- **`params.permit!`（全許可）**：mass assignment 防御を無効化。使わない。→ [strong_parameters.md](./strong_parameters.md)
- **`reset_session` 忘れ**：ログイン前後で再生成しないと固定化攻撃に弱い。→ [auth.md](./auth.md)
- **`secrets.yml` の production に秘密を直書きしてコミット**：ENV 参照に。漏れたらローテート。→ [config_secrets.md](./config_secrets.md)
- **Rails 4 が EOL**：本体に新規セキュリティパッチが来ない。`bundle-audit`/`brakeman` で監視を厚く、可能なら移行。→ [security.md](./security.md)

## コンソール / 運用
- **本番コンソールでの直接データ変更**：`--sandbox` を付け、対象を確定・件数確認してから実行。→ [console.md](./console.md)
- **rake タスクで `:environment` 忘れ**：モデルが読めず `uninitialized constant`。→ [console.md](./console.md)

## 関連
[model.md](./model.md) / [active_record.md](./active_record.md) / [controller.md](./controller.md) / [strong_parameters.md](./strong_parameters.md) / [javascript.md](./javascript.md) / [security.md](./security.md) / [config_secrets.md](./config_secrets.md) / [active_job.md](./active_job.md)
