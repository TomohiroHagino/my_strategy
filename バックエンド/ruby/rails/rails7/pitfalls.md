# 実務でハマる罠まとめ（Pitfalls）（Rails 7）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Railsは「便利な書き方」と「事故る書き方」が紙一重なものが多い。症状から該当箇所へ素早く飛ぶための索引。

## DB / Active Record
- **N+1問題**：一覧で関連を1件ずつ引きSQLが `1+N` 回。`includes` でまとめ読み。検出は bullet。→ [active_record.md](./active_record.md)
- **コールバック地獄**：`after_save` 等に副作用（メール送信・別モデル更新）を詰めると追跡不能。重い処理はService / Active Jobへ。→ [active_record.md](./active_record.md) / [service_form.md](./service_form.md)
- **`save` と `save!`**：`save` は失敗時 `false` を返すだけ（握り潰し注意）、`save!` は例外。調査時は `save!`。→ [model.md](./model.md)
- **`default_scope` の副作用**：全クエリに暗黙の絞り込みが効き、予期せぬ結果に。基本使わない。→ [active_record.md](./active_record.md)
- **`insert_all` / `upsert_all` / `update_all` / `delete_all`**：バリデーション・コールバック非実行。高速な代わりに整合性は自前で担保。→ [active_record.md](./active_record.md)
- **マイグレーション不可逆**：カラム削除等の破壊的変更は段階移行（追加→移行→後で削除）。`change` で戻せない操作は `up`/`down` を書く。→ [active_record.md](./active_record.md)
- **time zone**：`Time.now` でなく `Time.current`（`Time.zone.now`）。`Date.today` も `Time.zone.today` に。→ [active_record.md](./active_record.md)
- **マイグレーションとコードの不整合**：消したカラムをコードが参照したまま、で実行時エラー。

## モデル / 設計
- **Fat Model / Fat Controller**：分岐・計算・複数モデルにまたがる手続きが膨らんだらService / Formオブジェクトへ。→ [service_form.md](./service_form.md)
- **`belongs_to` は既定で必須**：任意にするなら `optional: true`。→ [model.md](./model.md)
- **モデルに表示/HTTPの都合を持ち込む**：整形はヘルパー/プレゼンターへ。→ [helper.md](./helper.md)

## コントローラ / リクエスト
- **Strong Parameters の `permit` 漏れ**：許可し忘れたキーは黙って無視＝「保存したのに入らない」。→ [strong_parameters.md](./strong_parameters.md)
- **二重 render/redirect（`DoubleRenderError`）**：1アクション1応答。分岐後の `return` 忘れに注意。→ [controller.md](./controller.md)
- **認可忘れ**：`Post.find` は他人のも引ける。`current_user.posts.find` でスコープを効かせる。→ [controller.md](./controller.md)

## Turbo / Hotwire
- **status 422（`:unprocessable_entity`）忘れ**：作成/更新失敗時に付けないとTurboがフォームエラーを再描画しない。→ [hotwire.md](./hotwire.md)
- **status 303（`:see_other`）**：Turboのredirect、特にPUT/PATCH/DELETE後のリダイレクトで必要になる場面がある。→ [hotwire.md](./hotwire.md)

## セキュリティ
- **`where` に文字列展開**：`"#{params[...]}"` はSQLインジェクション。`?` / 名前付きプレースホルダに。→ [security.md](./security.md)
- **`raw` / `html_safe` 乱用**：ユーザ入力に付けるとXSS。不安なら `sanitize`。→ [security.md](./security.md)
- **master.key / credentials の鍵をコミット**：`config/master.key` は `.gitignore` 必須。漏れたら再生成。→ [config_credentials.md](./config_credentials.md)
- **オープンリダイレクト**：`redirect_to params[:url]` は許可リストで検証。→ [security.md](./security.md)

## コンソール / 運用
- **本番コンソールでの直接データ変更**：`--sandbox` を付け、対象を確定・件数確認してから実行。→ [console.md](./console.md)

## 関連
[model.md](./model.md) / [active_record.md](./active_record.md) / [controller.md](./controller.md) / [security.md](./security.md) / [console.md](./console.md) / [hotwire.md](./hotwire.md)
