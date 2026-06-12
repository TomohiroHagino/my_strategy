# Turbo Drive（ページ遷移の高速化）（Rails 7）

## ひとことで言うと
リンククリックやフォーム送信を横取りして、**ページ全体を読み込み直さず `<body>` だけ差し替える**仕組み。CSS/JSを読み直さないので速い。Rails 7 では標準でON（コード不要）。昔の **Turbolinks** の正統進化版。

## なぜ速いのか（フルリロードとの違い）
普通のリンク（フルリロード）はクリックのたびに：
1. サーバからHTML取得
2. **CSS・JSを再ダウンロード → 再パース → 再実行**
3. 画面をゼロから組み立て直す

体感の重さの正体はほぼ「2」。Turbo Drive はここを潰す：
- 新ページを **Ajax（fetch）で取得**
- **`<body>` だけ差し替え**、`<head>`（CSS/JS）は維持
- URL は History API で書き換え（戻る/進むも効く）

→ CSS/JSを読み直さないので速い。SPA風に見える。

## 過去：Turbolinks（Rails 4〜5）
仕組みは同じだが、クセと罠が多かった。

### 罠①：JSが初回しか動かない
`DOMContentLoaded` は最初のフルロードでしか発火しない。Turbolinks遷移はbody差し替えだけなので発火しない。
```js
// ❌ Turbolinks遷移後に動かない
document.addEventListener("DOMContentLoaded", () => initMap());
// ✅ turbolinks:load を使う必要があった
document.addEventListener("turbolinks:load", () => initMap());
```
「2ページ目でJSが死ぬ」「登録が重複して二重発火」がRailsあるあるNo.1だった。

### 罠②：プレビューのチラつき
訪問済みページをキャッシュし、戻る時に一瞬"古い見た目"を表示 → 新しいのに差し替え。「古い内容がチラつく」問題。

### その他
- 対象は**リンクのみ**（フォーム送信は対象外）
- オプトアウト：`<a data-turbolinks="false">`

## 今：Turbo Drive（Rails 7）
基本の仕組みは同じ。進化点：

### ① フォーム送信も自動Ajax化（Turbolinksはリンクだけだった）
```erb
<%= form_with model: @post do |f| %>  <%# 自動でAjax送信される %>
```

### ② イベント名が turbo: に
```js
document.addEventListener("turbo:load", () => initMap());
```
主なイベント：`turbo:load` / `turbo:before-visit` / `turbo:before-render` / `turbo:render`

### ③ 初期化の罠を Stimulus で解決
昔は自分で `turbolinks:load` を書いていた。今は Stimulus コントローラが、DOMが現れたら自動 `connect()`、消えたら `disconnect()`。
```js
export default class extends Controller {
  connect()    { /* 表示されたら自動で呼ばれる（turbo:load を自分で書かなくていい） */ }
  disconnect() { /* 消えたら自動で後始末 */ }
}
```
→ 「2ページ目でJSが動かない問題」を、そもそも気にしなくて済む。

### ④ チラつき対策（キャッシュ無効化）
```erb
<meta name="turbo-cache-control" content="no-cache">
```

### ⑤ オプトアウト
```erb
<a href="/legacy" data-turbo="false">普通の遷移にする</a>
```

### ⑥ 進捗バーが標準で出る（遅い遷移時に上部にバー）

## フォームの作法（Turbo特有・重要）
- 成功してリダイレクト → **303（`status: :see_other`）**
- バリデーションエラーで同じ画面を再表示 → **422（`status: :unprocessable_entity`）**
- これを返さないとTurboが画面を更新しない（Rails 7頻出ハマり）
```ruby
def create
  @post = Post.new(post_params)
  if @post.save
    redirect_to @post, status: :see_other          # 303
  else
    render :new, status: :unprocessable_entity      # 422
  end
end
```

## 過去 → 今 早見表
| | Turbolinks（過去） | Turbo Drive（今） |
|---|---|---|
| 対象 | リンクのみ | リンク＋フォーム |
| JS初期化イベント | `turbolinks:load` | `turbo:load` |
| 初期化の罠 | 自分で書く | Stimulusで自動化 |
| オプトアウト | `data-turbolinks="false"` | `data-turbo="false"` |
| キャッシュ無効化 | （限定的） | `turbo-cache-control: no-cache` |
| 進捗バー | 後期にあり | 標準であり |

## ハマりどころ / アンチパターン
- 自前JSの初期化を `DOMContentLoaded` で書く → 遷移後に動かない。`turbo:load` か Stimulus へ
- フォームのエラーで再描画されない → 422 を返していない
- 戻った時に古い内容がチラつく → `turbo-cache-control: no-cache`
- どうしてもTurboと相性が悪いページ → `data-turbo="false"` で部分的に切る

## 関連
[javascript.md](./javascript.md) / [hotwire.md](./hotwire.md) / [controller.md](./controller.md)
