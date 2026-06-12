# 実務でハマる罠まとめ（Pitfalls）（Rails 5）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Rails は「便利な書き方」と「事故る書き方」が紙一重なものが多い。症状から該当箇所へ素早く飛ぶための索引。Rails 5 特有の罠（`belongs_to` 必須化など）を最初にまとめる。

## Rails 5 特有（バージョン起因の罠）
- **`belongs_to` が既定で必須になった（最頻・4→5移行で既存コードが壊れる）**：親が未設定だと保存時に検証エラー。任意にするなら `optional: true`、または `config.active_record.belongs_to_required_by_default = false` で旧挙動に戻す。→ [model.md](./model.md) / [active_record.md](./active_record.md)
- **APIモードに `protect_from_forgery` が無い**：Cookieセッション認証をAPIで使うと CSRF の穴。トークン認証なら不要。→ [api_mode.md](./api_mode.md) / [security.md](./security.md)
- **`secrets.yml`（5.0/5.1）と credentials（5.2）の取り違え**：`rails credentials:edit` は 5.2 から。それ以前は `secrets.yml`。→ [config_credentials.md](./config_credentials.md)
- **Active Storage は 5.2 から**：5.0/5.1 では使えない（CarrierWave / Paperclip を使う）。→ [active_storage.md](./active_storage.md)
- **system test は 5.1 から**：5.0 には無い。→ [testing.md](./testing.md)
- **`retry_on` / `discard_on` は 5.1 から**：5.0 のジョブはアダプタ側のリトライに頼る。→ [active_job.md](./active_job.md)
- **`deliver`（無印）非推奨**：`deliver_now` / `deliver_later` を使う。→ [action_mailer.md](./action_mailer.md)
- **`before_filter` 非推奨**：`before_action` を使う。→ [filters.md](./filters.md)

## フロント / JS（Turbolinks 5 + rails-ujs）
- **2ページ目以降でJSが動かない**：Turbolinks はフルリロードしないので `$(document).ready` が発火しない。**`turbolinks:load`** に乗せる。→ [javascript.md](./javascript.md)
- **削除リンクが反応しない**：rails-ujs（5.1+）/ jquery_ujs（5.0）がレイアウトで読み込まれていない。`//= require` を確認。→ [javascript.md](./javascript.md)
- **`form_with` が勝手にAjax化（5.1）**：5.1 の `form_with` は既定でリモート。HTML送信なら `local: true`。→ [view.md](./view.md) / [javascript.md](./javascript.md)

## DB / Active Record
- **N+1問題**：一覧で関連を1件ずつ引きSQLが `1+N` 回。`includes` でまとめ読み。検出は bullet。→ [active_record.md](./active_record.md)
- **コールバック地獄**：`after_save` 等に副作用（メール送信・別モデル更新）を詰めると追跡不能。重い処理は Service / Active Job へ。→ [active_record.md](./active_record.md) / [service_form.md](./service_form.md)
- **`save` と `save!`**：`save` は失敗時 `false` を返すだけ（握り潰し注意）、`save!` は例外。調査時は `save!`。→ [model.md](./model.md)
- **`default_scope` の副作用**：全クエリに暗黙の絞り込みが効き、予期せぬ結果に。基本使わない。→ [active_record.md](./active_record.md)
- **マイグレーション不可逆**：カラム削除等の破壊的変更は段階移行（追加→移行→後で削除）。→ [active_record.md](./active_record.md)
- **time zone**：`Time.now` でなく `Time.current`（`Time.zone.now`）。→ [active_record.md](./active_record.md)

## モデル / 設計
- **Fat Model / Fat Controller**：分岐・計算・複数モデルにまたがる手続きが膨らんだら Service / Form オブジェクトへ。→ [service_form.md](./service_form.md)
- **モデルに表示/HTTPの都合を持ち込む**：整形はヘルパー/プレゼンターへ。→ [helper.md](./helper.md)

## コントローラ / リクエスト
- **Strong Parameters の `permit` 漏れ**：許可し忘れたキーは黙って無視＝「保存したのに入らない」。→ [strong_parameters.md](./strong_parameters.md)
- **二重 render/redirect（`DoubleRenderError`）**：1アクション1応答。分岐後の `return` 忘れに注意。→ [controller.md](./controller.md)
- **認可忘れ**：`Post.find` は他人のも引ける。`current_user.posts.find` でスコープを効かせる。→ [controller.md](./controller.md)

## 非同期 / メール
- **トランザクション内で `perform_later` / `deliver_later`**：コミット前にワーカーが走り「まだ無いレコード」で失敗。`after_commit` で積む。→ [active_job.md](./active_job.md) / [action_mailer.md](./action_mailer.md)
- **メールの `default_url_options` 未設定**：本文で `_url` を使うと host が無くて落ちる。→ [action_mailer.md](./action_mailer.md)

## Action Cable
- **本番で `async` アダプタのまま**：プロセスをまたいでブロードキャストが届かない。本番は `redis`。→ [action_cable.md](./action_cable.md)
- **接続認証の漏れ**：Connection で認証しないと誰でも購読できる。→ [action_cable.md](./action_cable.md)

## セキュリティ
- **`where` に文字列展開**：`"#{params[...]}"` は SQLインジェクション。`?` / 名前付きプレースホルダに。→ [security.md](./security.md)
- **`raw` / `html_safe` 乱用**：ユーザ入力に付けると XSS。不安なら `sanitize`。→ [security.md](./security.md)
- **master.key / secrets の鍵をコミット**：`config/master.key`（5.2）は `.gitignore` 必須。漏れたら再生成・ローテーション。→ [config_credentials.md](./config_credentials.md)

## アセット / 運用
- **本番で precompile 忘れ**：資産404や古いまま。デプロイ手順に `assets:precompile`。→ [assets.md](./assets.md)
- **本番コンソールでの直接データ変更**：`--sandbox` を付け、件数確認してから実行。→ [console.md](./console.md)

## 関連
[model.md](./model.md) / [active_record.md](./active_record.md) / [controller.md](./controller.md) / [api_mode.md](./api_mode.md) / [security.md](./security.md) / [javascript.md](./javascript.md) / [config_credentials.md](./config_credentials.md)
