# コントローラ（CodeIgniter 4）

## ひとことで言うと
**リクエストを受け取り、モデルやサービスを呼んで、レスポンス（HTML や JSON）を返す調整役**。`App\Controllers\BaseController` を継承したクラスで、public メソッド 1 つ = 1 アクションになる。

## 役割・なぜ必要か
- ルーティングで振り分けられた処理の「受け皿」。`$this->request` で入力を受け取り、モデルでデータを扱い、ビューで HTML を組み、`$this->response` で返す、という**流れの司令塔**。
- 業務ロジック（計算・ルール）は**モデルやサービスに寄せ、コントローラは薄く保つ**。Rails の Skinny Controller と同じ思想で、再利用とテストがしやすくなる。
- 共通の前処理（フィルタ）と各アクションの間に立ち、入力検証 → 処理委譲 → 出力整形だけに専念させるのが理想。

## 基本の書き方（コード）
```php
<?php
// app/Controllers/PostController.php
namespace App\Controllers;

use App\Models\PostModel;

class PostController extends BaseController
{
    // initController でヘルパーや共通依存を読み込む（任意）
    protected $helpers = ['form', 'url'];   // BaseController に書くと全体で有効

    // GET /posts  → 一覧
    public function index()
    {
        $posts = (new PostModel())->orderBy('id', 'DESC')->findAll();
        return view('posts/index', ['posts' => $posts]);
    }

    // GET /posts/(:num)  → 詳細（$1 が渡る）
    public function show($id)
    {
        $post = (new PostModel())->find($id);
        if ($post === null) {
            throw \CodeIgniter\Exceptions\PageNotFoundException::forPageNotFound();
        }
        return view('posts/show', ['post' => $post]);
    }

    // POST /posts  → 作成
    public function create()
    {
        // $this->request で入力を取得
        $data = $this->request->getPost(['title', 'body']);
        (new PostModel())->insert($data);

        // $this->response / リダイレクト
        return redirect()->to('/posts')->with('msg', '作成しました');
    }

    // JSON を返す例
    public function api()
    {
        return $this->response->setJSON(['ok' => true]);
    }
}
```

```php
// initController：リクエスト処理前の初期化フック（DI・共通サービス取得など）
public function initController($request, $response, $logger)
{
    parent::initController($request, $response, $logger);
    $this->session = \Config\Services::session();
}
```

## 実務での使い方・定番パターン
- **public メソッド = アクション**。`index()` が既定。`show($id)` のように引数でルートの `$1` を受ける。→ [routing.md](./routing.md)
- 入力は **`$this->request`**（`getPost()` / `getGet()` / `getVar()` / `getJSON()`）、出力は **`$this->response`**（`setJSON()` / `setStatusCode()`）や `view()` / `redirect()`。→ [request_response.md](./request_response.md)
- **ヘルパー**は `$helpers` プロパティ（`['form','url']`）で宣言すると `helper()` 呼び出しを省ける。全体共通なら `BaseController` に書く。
- DB アクセスは**モデル経由**で。コントローラに SQL を直書きしない。→ [models.md](./models.md)
- 大きくなったら **サービスクラス**へ処理を出し、コントローラは「受けて・委譲して・返す」だけに保つ。

## ハマりどころ / アンチパターン
- **Fat Controller（太ったコントローラ）**：バリデーション・業務ルール・DB 操作を全部コントローラに書くと再利用もテストもできない。モデル/サービスへ分割。
- **メソッド名とルートの対応ズレ**：自動ルーティングを使わない CI4 では、Routes.php に書いた `Controller::method` と実メソッド名が一致していないと動かない。`php spark routes` で照合。
- **`_remap()` の乱用**：`_remap($method, ...$params)` で全メソッド呼び出しを横取りできるが、ルーティングが追えなくなる。明示ルートで足りるなら使わない。
- **public メソッドの意図しない公開**：自動ルーティング有効時は public メソッドが URL から呼べてしまう。内部用は `protected`/`private` にするか、明示ルーティングで露出を絞る。
- **コントローラにビュー HTML を直書き**：表示はビューへ。整形ロジックはヘルパー/ビューセルへ出す。

## 関連
[routing.md](./routing.md) / [models.md](./models.md) / [request_response.md](./request_response.md)
