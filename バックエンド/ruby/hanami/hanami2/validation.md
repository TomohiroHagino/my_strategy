# バリデーション（dry-validation Contract）（Hanami 2）

## ひとことで言うと
Hanami 2 の検証は **dry-validation** の **Contract（コントラクト）** を使う。特徴は、**「型・構造（schema）」と「業務ルール（rule）」を分けて書く**こと。schema が「正しい形・型か」を見て、それを通った値に対して rule が「業務的に妥当か」を見る。Rails の `validates ...`（モデルに検証を詰める）とは発想が異なり、**検証は独立した Contract オブジェクト**として切り出す。

## 役割・なぜ必要か
- 入力検証には実は**2種類**ある：「メールは文字列か／必須か」という**型・構造**の話と、「そのメールは既に登録済みでないか」という**業務**の話。
- これを混ぜると検証が読みにくくなる。dry-validation は **schema（型・構造）と rule（業務ルール）を意図的に分離**し、それぞれの責務を明確にする。
- Contract は**ただのオブジェクト**なので、アクションから注入して使う／単体でテストする、が自然にできる（モデルに縛られない）。

---

## Contract の基本形（schema と rule）
```ruby
# 例: ユーザー作成の入力検証
class CreateUserContract < Dry::Validation::Contract
  # ① schema: 型・構造・必須/任意（"形" の検証）
  params do
    required(:email).filled(:string)
    required(:age).filled(:integer)
    optional(:name).maybe(:string)
  end

  # ② rule: 業務ルール（"形" を通った後の検証）
  rule(:age) do
    key.failure("must be 18 or older") if value < 18
  end
end

result = CreateUserContract.new.call(email: "a@example.com", age: 15)
result.success?   # => false
result.errors.to_h
# => { age: ["must be 18 or older"] }
```
- **`params` / `schema`**：入力の**型と構造**を定義する層。`params` は「外部入力（文字列で来るパラメータ）を**型強制（coercion）してから**チェックする」用途、`schema` はすでに型付きの値を厳密に見る用途、という使い分け。
- **`rule`**：schema を**通過した値**に対してだけ走る、**業務ロジックの検証**。`key.failure(...)` でエラーを付ける。
- 順序が大事：**schema が落ちた項目には rule が走らない**（型が壊れた値に業務判定をかけない、という安全設計）。

## params / schema / rule の役割
| 要素 | 見るもの | 例 |
|---|---|---|
| `params` | 外部入力の型強制＋構造 | `"15"`(文字列) → `15`(整数) にしてから検証 |
| `schema` | 型付き済みの構造 | 既に型が確定した内部データの検証 |
| `rule` | 業務ルール | 「18歳以上」「在庫が足りる」「重複していない」 |

## アクションでのパラメータ検証
```ruby
module MyApp
  module Actions
    module Users
      class Create < MyApp::Action
        # Contract をアクションの contract として宣言（記法は版で異なり得る）
        contract CreateUserContract

        def handle(request, response)
          if request.params.valid?
            # 検証OK: request.params[:email] などを使って処理へ
            response.body = "ok"
          else
            response.status = 422
            response.body = request.params.errors.to_h.to_json
          end
        end
      end
    end
  end
end
```
- アクションでは入力（params）を Contract で検証し、`valid?` で分岐するのが定番。
- 失敗時は `errors` を取り出してレスポンス（422 など）に整形する。

## エラーの取り出し
```ruby
result = contract.call(input)

result.success?          # 成功？
result.failure?          # 失敗？
result.errors.to_h       # => { email: ["is missing"], age: [...] }  ネストしたハッシュ
result.errors(full: true).to_h   # フィールド名込みのメッセージ
result.to_h              # 検証後の（型強制済み）値
```
- エラーは **`errors.to_h`** でフィールド→メッセージ配列のハッシュとして取れる。API ならこれをそのまま JSON に。
- `to_h` は**型強制後の値**を返すので、後段の処理にはこの“整形済み”の値を渡すと安全。

## dry-types との関係
- `:string` `:integer` などの型指定は **dry-types** に由来する。`filled` / `maybe` / `optional` と組み合わせて「必須かつ非空」「nil 許容」などを表現する。
- 独自の型・制約（範囲・enum 相当）も dry-types 側で定義して schema から使える。schema は**型システムに支えられた検証**だと捉えるとよい。

## ハマりどころ / アンチパターン
- **schema（型）と rule（業務）を分ける思想**を崩す：型チェックを `rule` でやろうとしたり、業務判定を `params` に混ぜたりすると、本来の分離の利点が消える。**「形は schema、意味は rule」**で切り分ける。
- **`params` と `schema` の取り違え**：外部入力（文字列パラメータ）には**型強制する `params`**、型確定済みの内部データには `schema`。これを逆にすると coercion 周りでハマる。
- **rule は schema 通過後にしか走らない**前提を忘れる：必須項目が欠けた状態を rule 側で守ろうとしても呼ばれない。必須・型は schema 側で担保する。
- **エラー整形を毎回手書き**：`errors.to_h` の形を共通のレスポンス整形に寄せると重複が減る。
- 記法・アクション連携（`contract`/`params` の宣言方法）は **dry-validation / Hanami の版で差**がある。手元の version の公式ガイドで裏取りしてから固めるのが安全。

## 関連
[actions.md](./actions.md) / [settings_config.md](./settings_config.md) / [persistence_rom.md](./persistence_rom.md)
