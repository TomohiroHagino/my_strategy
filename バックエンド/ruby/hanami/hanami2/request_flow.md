# リクエストの流れ・各層は何を返すか（Hanami 2）

## ひとことで言うと
1リクエストが **config/routes.rb → Action（1アクション=1クラス）→ Repository → View → Template** と流れる。Rails の Fat Model と違い、**永続化は ROM の Repository が担い、返すのは不変の struct**。依存は DI コンテナから `Deps[...]` で注入する。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

> **この版 = Hanami 2.x（dry-rb / ROM）**：アクションは `#handle(request, response)` を持つ1クラス。ビューは独立した View クラス＋テンプレート。Active Record 的な「モデルがDBも業務も兼ねる」設計ではなく、責務を分離する。

## 全体の流れ（図）
```
ブラウザ
   │ リクエスト（GET /users/1, POST /users …）
   ▼
[config/routes.rb]  URL+HTTPメソッド → どの Action クラスか決める（ディスパッチ）
   │
   ▼
[Action（app/actions/users/show.rb）]  request を受け取り handle で処理
   │   ├ Deps[...] で Repository / Operation を注入して呼ぶ
   │   └ response にデータ・ステータスを詰める
   ▼
[Repository（ROM）]  DBアクセス。条件を受け取り struct を返す
   │
   ▼
  DB ──→ struct（不変のデータオブジェクト）を返す ─┐
   ▲                                              │
[Repository] が struct を Action に返す
   ▲
[Action] が response にデータを渡す → View へ
   ▲
[View（独立クラス）]  exposures でデータを整え → Template を描画
   │
   ▼ HTML（Template .html.erb が生成）
[response]  ステータス・ヘッダ・ボディを組み立てて返す
   │ レスポンス（HTML / JSON）
   ▼
ブラウザ
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す | 置き場所 |
|---|---|---|---|
| **routes.rb** | URL + HTTPメソッド | 「どの Action クラスか」の振り分け | `config/routes.rb` |
| **Action** | `request`（params 含む）/ `response` | **response**（データ・ステータスを詰めて返す）→ クライアントへ | `app/actions/` |
| **Repository（ROM）** | id / 条件 | **struct（不変のデータオブジェクト）**→ Actionへ | `app/relations`・`app/repositories` |
| **View** | Action が渡した値（exposures） | **HTML**（Template を描画）→ response のボディ | `app/views/` |
| **Template** | View が整えた値 | **HTML 文字列** | `app/templates/*.html.erb` |

- **Repository が返すのは ROM の struct（不変）**：Active Record レコードのように「保存メソッドを持つ生きたオブジェクト」ではない。読み出しと書き込みは Repository のメソッド経由で行う。
- **業務処理は Operation（任意）に切り出す**：複数ステップの処理は Action から `Deps["operations.create_user"]` を注入して呼ぶ。

## コードで通して見る
```ruby
# 1) config/routes.rb：URL → Action クラスを決める
module MyApp
  class Routes < Hanami::Routes
    get  "/users/:id", to: "users.show"     # → app/actions/users/show.rb
    post "/users",     to: "users.create"
  end
end

# 2) Action：1アクション=1クラス。Repository を注入し response を返す
module MyApp
  module Actions
    module Users
      class Show < MyApp::Action
        include Deps["repositories.user_repo"]   # DIコンテナから注入

        def handle(request, response)
          user = user_repo.find(request.params[:id])  # Repository が struct を返す
          response[:user] = user                       # View へ渡す（exposure 経由）
          # response.status = 200（既定）
        end
      end
    end
  end
end

# 3) Repository（ROM）：DBアクセス。struct を返す
module MyApp
  module Repositories
    class UserRepo < MyApp::DB::Repo
      def find(id) = users.by_pk(id).one    # 不変の struct を返す
    end
  end
end
```
```erb
<%# 4) Template（app/templates/users/show.html.erb）：HTMLを生成 %>
<h1><%= user.name %></h1>
```

## 実務での使い方・定番パターン
- **1アクション=1クラスを徹底**：`show` / `create` などをそれぞれ別ファイル・別クラスにする（Rails の「1コントローラに多アクション」ではない）。
- **依存は `Deps[...]` で注入**：Repository / Operation / 外部クライアントをコンテナ経由で受け取る。手動 `new` しない。→ [dependency_injection.md](./dependency_injection.md)
- **読み出しは Repository、業務は Operation**：DB アクセスは Repository、複数ステップの業務処理は Operation に分けて Action を薄く保つ。
- **View は独立させる**：表示整形は View クラスの exposures に置き、Action と Template を疎結合にする。→ [views_templates.md](./views_templates.md)
- **パラメータ検証は Contract**：dry-validation の Contract で入力を検証してから処理へ進む。→ [validation.md](./validation.md)

## ハマりどころ / アンチパターン
- **struct を可変だと思い込む**：Repository が返す struct は不変。更新は `user_repo.update(id, attrs)` のように Repository 経由で行う。
- **Action に DB クエリを直書き**：Action から ROM を直接叩かず、Repository に閉じ込める。Action は「注入されたものを呼ぶ」だけ。
- **DI を使わず手動 new**：`UserRepo.new` するとコンテナの自動配線・差し替え（テスト）が効かない。`Deps[...]` を使う。→ [dependency_injection.md](./dependency_injection.md)
- **コンテナ未登録での参照**：`app/` 配下の規約に沿わない置き方だと自動登録されず `Deps[...]` が解決できない。→ [project_structure.md](./project_structure.md)
- **Rails の Fat Model 流儀を持ち込む**：モデルにDBも業務も詰めない。Hanami は Action / Repository / Operation / View に責務を分ける。

## 関連
[routing.md](./routing.md) / [actions.md](./actions.md) / [persistence_rom.md](./persistence_rom.md) / [views_templates.md](./views_templates.md) / [dependency_injection.md](./dependency_injection.md) / [validation.md](./validation.md)
