# Slices（サブアプリ）（Hanami 2）

## ひとことで言うと
**Slice（スライス）** は、1つの Hanami アプリの中に置く**サブアプリケーション**。`slices/admin/` のように切り出すと、その配下が**独立したコンテナ・ルート・コンポーネント群**を持ち、機能や境界づけられたコンテキストごとにアプリを分割できる。Rails の「engine」に近い立ち位置だが、Hanami 2 では**最初から設計に組み込まれた標準機能**だと考えるとよい。

## 役割・なぜ必要か
- アプリが育つと、`app/` 一枚にすべて（公開API・管理画面・社内ツール…）を詰め込むと**境界が曖昧**になり、変更の影響範囲が読めなくなる。
- Slice は **機能/責務の境界**でコードをまとめ、それぞれに専用のコンテナ（DIコンテナ）と名前空間を与える。結果として「ここは admin の世界」「ここは api の世界」と**物理的に分離**できる。
- `app/` は全 Slice の**共有・親**の位置づけ。共通コンポーネントは `app/` に置き、Slice 固有のものは各 Slice 内に置く、という分担になる。

---

## ディレクトリの形（イメージ）
```text
app/                      # 親アプリ（共有コンポーネント）
slices/
  admin/
    actions/              # admin 固有のアクション
    views/
    relations/            # admin 固有の永続化（持たせることも可）
    repositories/
  api/
    actions/
    ...
config/
  routes.rb               # ルートで各 Slice をマウント
```
- 各 Slice は `app/` と**同じ構造**を内側に持てる（actions / views / 永続化 …）。
- Slice 配下のクラスは**その Slice のコンテナ**に自動登録され、名前空間も Slice 名で分かれる。

## ルーティングでのマウント
```ruby
# config/routes.rb
module MyApp
  class Routes < Hanami::Routes
    root to: "home.index"

    slice :admin, at: "/admin" do        # /admin 以下を admin スライスへ
      get "/dashboard", to: "dashboard.show"
    end

    slice :api, at: "/api" do
      get "/users", to: "users.index"
    end
  end
end
```
- `slice :admin, at: "/admin"` で、URL プレフィックスと Slice を結びつける。
- 各 Slice 内のアクションは、その Slice の名前空間（`MyApp::Admin::Actions::...`）で解決される。

## Slice ごとのコンテナ（DI）
```ruby
# slices/admin/actions/dashboard/show.rb
module MyApp
  module Admin
    module Actions
      module Dashboard
        class Show < MyApp::Admin::Action
          # admin スライスのコンテナから注入される
          include Deps["repositories.report_repo"]

          def handle(request, response)
            response[:reports] = report_repo.recent
          end
        end
      end
    end
  end
end
```
- Slice は**独立したコンテナ**を持つので、`Deps[...]` はまずその Slice のコンテナを見る。
- 親（`app/`）の共有コンポーネントも参照できる（共有しつつ、固有は分離、という設計）。

## 実務での使い方・定番パターン
- **公開 / 管理 / 内部**で分ける：`api`（公開API）、`admin`（管理画面）、`internal`（社内）など、**ユーザー層・認可境界**で切るのが分かりやすい。
- **境界づけられたコンテキスト（DDD）**で分ける：請求・在庫・通知など、ドメインの境界を Slice にマッピングする使い方。
- **共有は `app/` に寄せる**：複数 Slice で使う値オブジェクトや共通サービスは親へ。Slice 固有のものだけ各 Slice に置く。
- まずは `app/` だけで始め、**境界が見えてきたら Slice に切り出す**（最初から細かく割らない）。

## ハマりどころ / アンチパターン
- **Slice の境界設計（いつ分けるか）**が最大の悩みどころ。早すぎる分割は移動コストとボイラープレートを増やす。**境界が安定してから**切るのが無難。
- **Slice 間の依存**：admin が api の内部に直接依存し始めると、せっかくの分離が崩れる。共有したいものは `app/`（共有層）に上げるか、明示的なインターフェース越しにする。
- **過剰分割**：機能ごとに何でも Slice にすると、ファイルが散り、`Deps[...]` の解決先（どのコンテナ？）が追いづらくなる。Slice は「**独立したサブアプリ**」級の単位に留める。
- **名前空間ずれ**：Slice 配下は `MyApp::Admin::...` のように名前空間が一段増える。パス・モジュール定義のずれで自動登録に乗らないことがある。
- Rails 経験者は engine と混同しがち。**マウント・コンテナ・名前空間**が Slice 単位で分かれる点を、手元の version の公式ガイドで確認しておくと安全。

## 関連
[project_structure.md](./project_structure.md) / [dependency_injection.md](./dependency_injection.md) / [routing.md](./routing.md)
