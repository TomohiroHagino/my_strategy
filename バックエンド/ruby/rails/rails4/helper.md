# ヘルパー（Helper）（Rails 4）

## ひとことで言うと
ビューから呼ぶ**表示整形用のメソッド置き場**。`app/helpers/` のモジュールに定義し、ビュー内でそのまま呼べる。

## 役割・なぜ必要か
- ビューに散らばりがちな「整形ロジック（日付フォーマット・ステータスのラベル付け・条件付きCSSクラス）」を切り出して、ビューを表示に専念させるためにある。
- モデルに「表示の都合」を持ち込まないための受け皿。判断はモデル、整形はヘルパー、という役割分担。

## 基本の書き方（コード）
```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def post_status_label(post)
    if post.published?
      content_tag(:span, "公開", class: "label label-success")
    else
      content_tag(:span, "下書き", class: "label label-default")
    end
  end

  def formatted_date(time)
    time.present? ? l(time, format: :short) : "-"
  end
end
```
```erb
<%# ビュー側 %>
<%= post_status_label(@post) %>
<%= formatted_date(@post.published_at) %>
```

## 実務での使い方・定番パターン
- **Rails 4 はヘルパーがデフォルトで全ビューから呼べる**（`include_all_helpers` が既定 true）。`PostsHelper` のメソッドが他のビューからも見える。
- **Railsが用意する組み込みヘルパー**を活用：`link_to` / `form_for` / `image_tag` / `number_to_currency` / `truncate` / `pluralize` / `time_ago_in_words` など。
- 表示専用の組み立ては `content_tag` / `tag` / `safe_join` で。文字列結合で `html_safe` を直書きしない。
- 整形ロジックが増えてオブジェクト単位でまとめたくなったら **プレゼンター/デコレータ**（`draper` gem 等）へ移す。
- アプリ全体で使う整形は `ApplicationHelper` に。

## ハマりどころ / アンチパターン
- **ヘルパーにビジネスロジックを書く**：判断（保存可否・権限）はモデル/ポリシーへ。ヘルパーは「見せ方」だけ。
- **`html_safe` の手書き連結**：`"<span>#{x}</span>".html_safe` はユーザ入力でXSS。`content_tag` を使う。→ [security.md](./security.md)
- **全ヘルパーがグローバルに見える副作用**：同名メソッドの衝突に注意（4は `include_all_helpers` 既定 true）。名前を具体的に。
- **複雑な分岐をヘルパーに詰めすぎ**：オブジェクト単位の整形はデコレータへ。

## 関連
[view.md](./view.md) / [model.md](./model.md) / [partial_layout.md](./partial_layout.md)
