# リクエストの流れ・各層は何を返すか（Laravel 11）

## ひとことで言うと
1リクエストが **routes → Middleware →（FormRequest検証）→ Controller → Service/Model(Eloquent)** と降り、**Model(Eloquent) が逆向きに上がってきて、境界で Resource/JSON or Blade に詰め替えて返す**。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

## 全体の流れ（図）
```
クライアント
   │ リクエスト（POSTなら request body = JSON or form）
   ▼
[routes/api.php・web.php]  URL→アクションを対応づけ／Middlewareを指定
   │
   ▼
[Middleware]   認証・CSRF・throttleなど横断処理（通過 or 拒否）
   │
   ▼
[FormRequest]  rules() で入力検証（NGなら 422 で即返す）→ validated() を渡す
   │
   ▼
[Controller]   検証済みデータを受け取る／Service or Model を呼ぶ
   │
   ▼
[Service / Model(Eloquent)]  業務ロジック・DBアクセス
   │
   ▼
  DB ──→ Model(Eloquent) を返す ─┐
   ▲                              │
[Model] が Eloquent インスタンス/Collection を Controller に返す
   ▲
[Controller] が API Resource or JSON（API）／Blade(HTML)（Web）に詰め替えて返す
   │ レスポンス（response body = JSON or HTML）
   ▼
クライアント
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す |
|---|---|---|
| **routes** | HTTPメソッド + URL | 対応するController/クロージャへ振り分け |
| **Middleware** | `Request` | `Request` を次へ通す or レスポンス（拒否） |
| **FormRequest** | 生の入力 | `validated()`（検証済み配列）／422レスポンス |
| **Controller** | FormRequest / パス変数 | **API Resource / JSON**（API）or **Blade View**（Web） |
| **Service / Model** | DTO/値 / 検索条件 | **Eloquent インスタンス / Collection** |

- Controllerは「受けて・呼んで・詰め替えて返す」だけ。業務とDBアクセスはService/Modelへ。
- Eloquentインスタンスを直接JSONで返さず、API Resourceで出力形を制御する。

## コードで通して見る
```php
// 1) FormRequest：入力検証（NGなら自動で422）
class StorePostRequest extends FormRequest {
    public function rules(): array {
        return ['title' => 'required|string|max:255', 'body' => 'required|string'];
    }
}

// 2) Controller：検証済みデータ → Eloquentで作成 → Resourceで返す
class PostController extends Controller {
    public function store(StorePostRequest $request) {
        $post = Post::create($request->validated()); // Eloquentが Post を返す
        return new PostResource($post);              // Resourceに詰め替えて返す（JSON化される）
    }
}

// 3) API Resource：出力形（クライアントへ返すJSONの形）を定義
class PostResource extends JsonResource {
    public function toArray($request): array {
        return ['id' => $this->id, 'title' => $this->title]; // 必要な項目だけ
    }
}
```

## 実務での使い方・定番パターン
- **層をまたいで返す型を決めておく**：Model→Controller は Eloquent、Controller→クライアントは Resource/JSON（API）か Blade（Web）。
- **検証は FormRequest に切り出す**：Controllerをスリムに保つ。複雑な業務は Service クラスへ。→ [validation.md](./validation.md)
- **Eloquentをそのまま返さない**：API Resourceで出力形を固定すると、列追加やリレーションの露出を制御できる。
- **Middlewareで横断処理を共通化**：auth / throttle / CSRF はルート単位で付与する。→ [middleware.md](./middleware.md)

## ハマりどころ / アンチパターン
- **Controllerに業務やクエリを直書き**：層が崩れる。Service/Modelに寄せる。
- **Eloquentを直接JSONレスポンスにする**：`$hidden` の漏れ・リレーションのN+1・不要列の露出が起きる。Resourceに詰め替える。→ [pitfalls.md](./pitfalls.md)
- **FormRequestを使わずControllerで`$request->validate()`乱用**：検証ルールが散る。FormRequestに集約。
- **mass assignment脆弱性**：`Post::create($request->all())` は危険。`validated()` か `$fillable` を使う。→ [security.md](./security.md)

## 関連
[routing.md](./routing.md) / [middleware.md](./middleware.md) / [controller.md](./controller.md) / [model.md](./model.md) / [validation.md](./validation.md) / [blade.md](./blade.md)
