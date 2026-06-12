# リクエストの流れ・各層は何を返すか（CodeIgniter 4）

## ひとことで言うと
1リクエストが **Routes → (Filters) → Controller → Model** と降り、**Model が配列/Entity を逆向きに上げてきて、Controller が `view()`(HTML) or `setJSON`(API) に詰め替えて返す**。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

## 全体の流れ（図）
```
クライアント
   │ リクエスト（POSTなら request body）
   ▼
[app/Config/Routes.php]  URL→Controllerメソッドを対応づけ
   │
   ▼
[Filters(before)]  認証・CSRFなど前処理（通過 or 拒否）
   │
   ▼
[Controller]   $this->request で入力取得・検証／Model を呼ぶ
   │
   ▼
[Model]        DBアクセス（Query Builder内蔵）
   │
   ▼
  DB ──→ 配列 or Entity を返す ─┐
   ▲                            │
[Model] が 配列/Entity を Controller に返す
   ▲
[Controller] が view()(HTML) or $this->response->setJSON()(API) に詰めて返す
   │
   ▼
[Filters(after)]  ヘッダ付与など後処理
   │ レスポンス
   ▼
クライアント
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す |
|---|---|---|
| **Routes** | HTTPメソッド + URL | 対応するController::method へ振り分け |
| **Filters(before)** | `Request` | 次へ通す or レスポンス（拒否） |
| **Controller** | `$this->request`（入力）/ パラメータ | **`view()`(HTML文字列)** or **`setJSON()`(JSON)** |
| **Model** | id / 検索条件 / 保存データ | **配列 or Entity**（DBから） |
| **View** | Controllerが渡すデータ配列 | レンダリング済みHTML文字列 |

- Controllerは入力を受け取り、Modelを呼び、結果をView or JSONに詰めて返す係。
- Modelは既定で配列を返す。`$returnType = 'App\Entities\Post'` を設定するとEntityを返す。

## コードで通して見る
```php
// 1) Routes：URL→Controllerメソッド
$routes->post('posts', 'PostController::create');

// 2) Controller：検証 → Model呼び出し → JSONで返す
class PostController extends BaseController {
    public function create() {
        $rules = ['title' => 'required|max_length[255]'];
        if (! $this->validate($rules)) {
            return $this->response->setStatusCode(422)
                ->setJSON(['errors' => $this->validator->getErrors()]);
        }
        $model = new PostModel();
        $id = $model->insert($this->request->getPost()); // Modelが結果(挿入ID)を返す
        $post = $model->find($id);                        // Modelが配列/Entityを返す
        return $this->response->setJSON($post);           // JSONに詰めて返す
    }
}

// 3) Model：Query Builder内蔵。配列 or Entity を返す
class PostModel extends Model {
    protected $table = 'posts';
    protected $allowedFields = ['title', 'body'];
}
```

## 実務での使い方・定番パターン
- **層をまたいで返す型を決めておく**：Model→Controller は 配列/Entity、Controller→クライアントは `view()` か `setJSON`。
- **検証は Model か Controller の validation ルールに集約**：Modelに `$validationRules` を持たせると保存時に自動検証。→ [validation.md](./validation.md)
- **横断処理は Filters**：認証・CSRF・CORSは before/after フィルタで共通化。→ [filters.md](./filters.md)
- **API主体なら `$returnType` を Entity に**：オブジェクトでアクセサを使える。

## ハマりどころ / アンチパターン
- **Controllerに直接SQL/業務ロジックを書く**：層が崩れる。Modelに寄せる。
- **`$allowedFields` 未設定で mass assignment**：保存したい列が通らない/想定外列が入る。明示する。→ [pitfalls.md](./pitfalls.md)
- **viewにエスケープ漏れ**：`<?= esc($x) ?>` を徹底。`<?= $x ?>` 直書きはXSS。→ [security.md](./security.md)
- **Filtersの登録忘れ**：`app/Config/Filters.php` で有効化しないと before/after は動かない。

## 関連
[routing.md](./routing.md) / [filters.md](./filters.md) / [controllers.md](./controllers.md) / [models.md](./models.md) / [views.md](./views.md) / [request_response.md](./request_response.md)
