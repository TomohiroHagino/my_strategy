# ヘルパー（Helper）（Rails 5）

## ひとことで言うと
ビューで使う**表示用の小さなメソッド置き場**。`app/helpers/` に置き、ビュー内からそのまま呼べる。

## 役割・なぜ必要か
- 日付フォーマット・金額表示・条件付きクラス付与など「見た目を整える処理」をビューのERBから切り離し、再利用とテストをしやすくするためにある。
- ビューに長いロジックを書くと読めなくなる。ヘルパーに名前を付けて追い出すことで、ビューは「何を出すか」だけに集中できる。

## 基本の書き方（コード）
```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def published_label(post)
    post.published? ? "公開中" : "下書き"
  end

  def formatted_date(time)
    time.strftime("%Y/%m/%d")
  end
end
```
```erb
<%# ビューからそのまま呼べる %>
<span><%= published_label(@post) %></span>
<time><%= formatted_date(@post.created_at) %></time>
```
- Rails 5 では既定で**全ヘルパーが全ビューから呼べる**（`config.action_controller.include_all_helpers = true` が既定）。

## 実務での使い方・定番パターン
- **Rails 組み込みヘルパー**を活用：`link_to` / `form_with` / `number_to_currency` / `time_ago_in_words` / `pluralize` など。
- **`content_tag` / `tag`** でHTMLを安全に組み立てる（手で文字列連結しない）。
- **`safe_join`** で配列を結合（エスケープを保ったまま）。
- 表示ロジックが**特定モデルに密着**して複雑化したら、ヘルパーより **Presenter / Decorator**（draper gem など）に寄せると整理しやすい。

## ハマりどころ / アンチパターン
- **ヘルパーに業務ロジック**：判定や計算を詰めるとモデルの責務を奪う。ヘルパーは「表示整形」に限定する。
- **`html_safe` の安易な使用**：ユーザ入力を `html_safe` で出すと XSS。`content_tag` や自動エスケープに任せる。→ [security.md](./security.md)
- **巨大ヘルパー**：1モジュールに何十メソッドも詰めると見通しが悪い。機能ごとに分割。
- **ヘルパーから直接DBアクセス**：N+1や責務違反の温床。データはコントローラ/モデルで用意して渡す。

## 関連
[view.md](./view.md) / [partial_layout.md](./partial_layout.md) / [security.md](./security.md)
