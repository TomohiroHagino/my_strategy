# ヘルパー（Helper）（Rails 7）

## ひとことで言うと
ビューで使う **「表示まわりの小さなロジック」を切り出すメソッド置き場**。`app/helpers/` に置き、ビューから直接呼べる。

## 役割・なぜ必要か
- 日付の整形、金額表示、状態に応じたCSSクラス、条件付きの文言…といった「見た目のための小さな加工」をビューから追い出して、ERBを読みやすく保つためにある。
- ビジネスロジック（業務ルール）はモデル/Serviceの担当。ヘルパーはあくまで**表示用**。

## 基本の書き方（コード）
```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def status_badge(post)
    klass = post.published? ? "badge badge-green" : "badge badge-gray"
    content_tag(:span, post.status.titleize, class: klass)
  end

  def posted_at(post)
    post.created_at.strftime("%Y/%m/%d")
  end
end
```
```erb
<%= status_badge(@post) %>
<%= posted_at(@post) %>
```

### Rails 標準の組み込みヘルパー（よく使う）
- リンク/フォーム: `link_to` / `button_to` / `form_with`
- 数値/日付: `number_to_currency` / `number_with_delimiter` / `time_ago_in_words` / `l`（I18nローカライズ）
- 出力安全化: `sanitize` / `truncate` / `pluralize`

## 実務での使い方・定番パターン
- 「ビューに2行以上のRubyロジックが出てきたら helper を検討」が目安。
- 表示が複雑なモデル特化のものは **Presenter / Decorator（draper gem 等）** に発展させることも。
- I18n（多言語/文言一元管理）と組み合わせ、文言は `t("...")`、整形は helper、で分担。

## ハマりどころ / アンチパターン
- **ヘルパーに業務ロジックを置く**（料金計算・権限判定など）→ 本来はモデル/Service/Policy。
- **ヘルパーは全ビューでグローバルに混ざる**（既定で `include_all_helpers`）→ メソッド名の衝突に注意。命名を具体的に。
- `html_safe` を安易に返す → XSS。組み立ては `content_tag` / `tag` / `safe_join` を使う。
- ロジックが重い/状態を持つなら helper でなくオブジェクト（Presenter）に。

## 関連
[view.md](./view.md) / [model.md](./model.md)
