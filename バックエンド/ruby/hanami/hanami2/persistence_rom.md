# 永続化（ROM / Repository）（Hanami 2）

## ひとことで言うと
Hanami 2 の永続化は **ROM（Ruby Object Mapper）** が担う。Rails の Active Record とは**別思想**で、データアクセスを担う **Relations**（テーブル/クエリ）と、ドメイン向けの永続化APIである **Repositories** を**明確に分離**する。読み出した結果は、メソッドで自分を保存したりしない**イミュータブルな struct（ただの値）**として返ってくる、と理解しておくとよい。

## 役割・なぜ必要か
- Active Record は「1つのモデルがテーブル行・クエリ・検証・保存・関連まで全部持つ」設計。便利な反面、**モデルが肥大化**し、ドメインロジックとDBの都合が密結合になりやすい。
- ROM は **データゲートウェイ（Relation）** と **永続化の窓口（Repository）** を分け、ドメインオブジェクト（struct）を「DBを知らないただの値」に保つ。これにより**責務が分かれ**、テストや差し替えがしやすくなる、という狙い。
- Hanami 2.2 で **DB 層（ROM 統合・自動登録・マイグレーション）が固まった**とされ、`app/` 配下の relations / repositories がコンテナに自動登録され、`Deps[...]` で注入して使える流れになっている。

---

## ROM の登場人物
| 役割 | 置き場所（目安） | 担当 |
|---|---|---|
| Relation | `app/relations/` | テーブルそのもの・低レベルのクエリ |
| Repository | `app/repositories/` | ドメイン向けの永続化API（高レベル） |
| Struct（結果） | （自動生成 / 明示も可） | 読み出し結果＝**不変の値オブジェクト** |
| Migration | `config/db/migrate/`（目安） | スキーマ変更 |

「**Relation = 低レベル（テーブル/クエリ）**、**Repository = 高レベル（ドメインの言葉で操作）**」という二段構えが核。

## Relation（テーブル/クエリ）
```ruby
# app/relations/users.rb
module MyApp
  module Relations
    class Users < MyApp::DB::Relation
      schema :users, infer: true do      # DBスキーマから型を推論
        associations do
          has_many :posts                # 関連を宣言
        end
      end

      # 低レベルのクエリをメソッド化（ROMの記法）
      def active
        where(active: true)
      end
    end
  end
end
```
- Relation は **テーブル＝関係（relation）** を表す。`where` / `order` などは**新しい Relation を返す**（破壊的でなく合成していくイメージ）。
- ここでは「ドメインの都合」ではなく「DB クエリの都合」を書く。直接アクションから触るより、Repository 越しに使うのが定番。

## Repository（永続化API・ドメイン向け）
```ruby
# app/repositories/user_repo.rb
module MyApp
  module Repositories
    class UserRepo < MyApp::DB::Repo
      def create(attrs)
        users.changeset(:create, attrs).commit   # 書き込みは changeset 経由
      end

      def find(id)
        users.by_pk(id).one                       # struct を1件返す（無ければ nil）
      end

      def active_with_posts
        users.active.combine(:posts).to_a         # Relation を組み立てて取得
      end
    end
  end
end
```
- Repository は**ドメインの言葉**（`find`, `active_with_posts` …）で永続化を提供する窓口。内部で Relation を組み立てて呼ぶ。
- アクションやサービスは Repository を **`Deps["repositories.user_repo"]` で注入**して使う（→ dependency_injection.md）。モデルを直接 `new.save` する発想は出てこない。

## 結果は「イミュータブルな struct」
```ruby
user = user_repo.find(1)
user.name          # => "Alice"  読み出しはできる
user.posts         # => [#<...>, ...] combine していれば関連も持てる

# user.name = "Bob"; user.save  ← こういう発想は ROM にはない
# 変更は Repository（changeset）経由で「新しい状態」を書く
```
- 返ってくる struct は**ただの値**。Active Record の「インスタンスが自分を `save` する」挙動は**ない**。
- 値を変えたい＝**Repository を通して更新を投げる**。これが Active Record との一番の発想差。

## DTO（データ運搬オブジェクト）はどこ？

Hanami に「DTO」という名前の専用層は無いが、役割は分担されている。

| DTOの役割 | Hanami での担当 |
|---|---|
| DBから取り出したデータの運搬（出力） | **struct**（`Hanami::DB::Struct`・不変）＝実質これが read DTO |
| 入力（params/フォーム）の検証・整形 | **dry-validation の contract**（action内） |
| 画面表示用の整形 | **View の Part / exposures** |

- Spring の「Entity↔DTO 変換（mapper）」は、**Repo が struct を返す時点で「DBの形」と「使う形」が分かれている**ため、別途 mapper を書かないことが多い。
- 明示的な DTO クラスが欲しければ、`dry-struct` の値オブジェクトを自分で作って `app/` や `lib/` に置けばよい。

## 実務での使い方・定番パターン
- **読み出しは Relation で組み立て → Repository が公開**。一覧・検索・関連ロードは Relation のクエリメソッドに名前を付けて整理。
- **書き込みは changeset 経由**（`create` / `update`）。型変換や前処理を changeset にまとめられる。
- **関連は `combine`**（`includes` 相当の eager load 思想）で N+1 を避ける。
- アクションは Repository だけに依存させ、DBの詳細（Relation/SQL）はアクションから隠す。
- マイグレーションでスキーマ、Relation の `infer: true` でスキーマ追従、という分担。

## ハマりどころ / アンチパターン
- **Active Record と別物**：モデルが自分で `save` しない。`user.save` や `User.find` のような“モデル中心”の操作は無い。**Repository が窓口**だと割り切る。
- **Relation と Repository の役割混同**：低レベル（Relation＝クエリ）と高レベル（Repository＝ドメインAPI）を分ける。アクションから Relation を直接叩き始めると分離が崩れがち。
- **struct を破壊的に変更しようとする**：結果は不変。変更は必ず Repository（changeset）から。
- **関連の取り忘れ**：`combine` していない関連にアクセスして取れない／追加クエリになる。読み出し時に何を持たせるか意識する。
- バージョン差に注意：DB 層は **2.2 で固まった**部分で、記法やパス（`app/db`・自動登録の詳細）は版で変わり得る。**手元の version の公式ガイドで裏取り**してから書くのが安全。

## N+1 と ROM 特有の現象と対策
ROM/Hanami は ActiveRecord と思想が逆で、**「遅延ロードを持たない＝必要な関連は取得時に明示する」**設計。ここを掴むとN+1も他の罠も理解できる。
| 現象 / 罠 | なぜ起きる | 対策 |
|---|---|---|
| **N+1** | `combine` していない関連にアクセスし、その都度クエリ（または取れない） | 取得時に **`combine(:posts)`**（`includes` 相当・別クエリでまとめ取り）。ネストは `combine(posts: :comments)` |
| 「遅延ロードで後から取れる」誤解 | ROMの struct は**不変・遅延ロード無し**。AR感覚で `user.posts` を期待すると取れない/別クエリ | **読み出し時に何を持たせるか先に決める**。Relation/Repository のメソッドに「この一覧は posts 込み」を名前で固定 |
| JOINとまとめ取りの混同 | `combine` は別クエリでまとめ取り。JOINで絞りたい場面と違う | 関連で**絞り込みたい**なら Relation で `join` を使う。**関連を持たせたい**なら `combine` |
| 集計をRubyでやる | struct を全件ロードして Ruby で数える | Relation 側で集計（`select`/`group`/カウント）してから返す |
| 大量データのメモリ | `.to_a` で全件配列化 | Relation の `each`（ストリーム的取得）やページング（`limit`/`offset`）で分割 |
| Repository を通さない更新 | struct は不変。直接変更しようとして失敗 | 書き込みは **changeset**（`create`/`update`）経由のみ |
- **検出**：ROMのロガー（`ROM` の `:sql` ロガー）で発行SQLを目視し、一覧で件数に比例してクエリが増えていないか確認。

## 関連
[dependency_injection.md](./dependency_injection.md) / [validation.md](./validation.md) / [project_structure.md](./project_structure.md)
