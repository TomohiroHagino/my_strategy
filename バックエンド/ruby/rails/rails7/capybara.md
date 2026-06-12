# Capybara（Rails 7）

## ひとことで言うと
ブラウザ操作を**人間の言葉に近い API で書く**E2E テスト用ライブラリ。「このページを開く」「このボタンを押す」「この文字が見える」を `visit`・`click_on`・`have_content` で記述する。Rails の system spec の操作部分はこれが担い、RSpec と組み合わせて重要フローを画面から検証する。

## 役割・なぜ必要か
- request spec は HTTP の入出力までしか見ない。実際にユーザがフォーム入力・クリックして動くかは、**ブラウザを動かして確認**する必要がある。それを担うのが Capybara。
- DOM 要素を CSS や表示テキストで探し、操作・検証する API を提供する。低レベルなドライバ（Selenium 等）の差を吸収し、テストコードはドライバ非依存に書ける。
- **自動待機**を内蔵し、非同期（Ajax・Turbo）で後から現れる要素も `have_content` 等が出現まで待つ。`sleep` を書かずに安定したテストにできる。

## 基本の書き方（コード）
```ruby
# spec/system/login_spec.rb（system spec）
RSpec.describe "ログイン", type: :system do
  before do
    driven_by(:rack_test)   # 既定ドライバを指定（後述）
  end

  it "正しい情報でログインできる" do
    user = create(:user, password: "secret")   # FactoryBot

    visit login_path                    # ページを開く
    fill_in "Email", with: user.email   # ラベル/name/id でフィールド特定
    fill_in "Password", with: "secret"
    click_button "ログイン"             # ボタン押下（click_on でリンクも可）

    expect(page).to have_content("ようこそ")        # 表示テキストの検証
    expect(page).to have_current_path(dashboard_path)
  end
end
```
```ruby
# 主な操作 API
visit "/posts"                       # 遷移
click_on "新規作成"                  # リンク or ボタン（テキスト/id）
fill_in "title", with: "記事"        # テキスト入力
select "公開", from: "status"        # セレクトボックス
check "規約に同意"                   # チェックボックス
choose "プランA"                     # ラジオボタン
attach_file "画像", "spec/fixtures/a.png"  # ファイル添付

# 主な検証マッチャ（いずれも自動待機する）
expect(page).to have_content("保存しました")
expect(page).to have_selector("h1", text: "一覧")
expect(page).to have_link("編集", href: edit_post_path(post))
expect(page).to have_field("Email", with: "a@example.com")
expect(page).not_to have_content("エラー")
```
```ruby
# スコープを絞って操作（同名要素が複数あるとき）
within "#sidebar" do
  click_on "設定"
end

within(".post", text: "記事A") do
  click_on "削除"
end
```

## 実務での使い方・定番パターン
- **ドライバを使い分ける**：`rack_test` は JS を実行しない高速ドライバ（純粋なフォーム送信向き）。JS（Turbo・Stimulus・Ajax）が絡む画面は `selenium`（ヘッドレス Chrome）や Cuprite を使う。spec ごとに `driven_by(:selenium_chrome_headless)` で切替。
- **JS が要る spec だけ重いドライバ**：全部 Selenium にすると遅い。`js: true` 相当のタグでドライバを出し分け、JS 不要な画面は `rack_test` で速く回す。
- **要素はテキスト・ラベルで探す**：`fill_in "Email"` はラベル/name/id を見る。CSS セレクタ直書きより壊れにくく、画面の意図に沿う。
- **`within` で範囲限定**：一覧の特定行など同名要素が複数ある場合、`within` でスコープを絞ってから操作する。
- **自動待機を信じる**：`have_content` 等は既定で数秒待つ。Ajax 後の要素も `sleep` なしで検証できる。
- **system spec の前提データは FactoryBot**：ログインユーザや一覧データは `create(:user)` 等で用意する（→ [factory_bot.md](./factory_bot.md)）。

## ハマりどころ / アンチパターン
- **`sleep` で待つ（最頻のアンチパターン）**：固定待機は遅く・不安定（flaky）になる。`have_content`・`have_selector` の**自動待機マッチャ**を使う。
- **`have_no_content` を `not_to have_content` で代用しない場面**：「無いこと」を検証するとき `expect(page).to have_no_content(x)` は待機して消えるのを待つが、`not_to have_content` は即時判定で誤判定しうる。消えるのを待ちたいなら `have_no_*` を使う。
- **JS 画面を `rack_test` でテスト**：`rack_test` は JS を実行しないため、Turbo/Ajax で描画される要素が見つからず落ちる。JS が要る画面は Selenium 系に切り替える。
- **flaky の放置**：たまに落ちる system spec を再実行で誤魔化すと信頼性が崩れる。待機マッチャの徹底・データ初期化・ドライバ設定を見直す。
- **CSS セレクタ直書きの多用**：`find(".btn-primary")` のような実装依存の指定はデザイン変更で壊れる。表示テキストやラベルで探す。
- **DB クリーニングと非同期の競合**：Selenium は別スレッドで動くため、トランザクションロールバックが効かないことがある。必要なら `database_cleaner` の truncation 戦略を併用する。

## 関連
[testing.md](./testing.md) / [rspec.md](./rspec.md) / [factory_bot.md](./factory_bot.md)
