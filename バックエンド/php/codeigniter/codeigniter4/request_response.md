# リクエスト / レスポンス（Request / Response）（CodeIgniter 4）

## ひとことで言うと
- **Request**（`$this->request`＝`IncomingRequest`）：ブラウザから来た入力（GET/POST/JSON・ヘッダ・ファイル）を読むオブジェクト。
- **Response**（`$this->response`）：返すHTTP（本文・ステータス・ヘッダ・JSON）を組み立てるオブジェクト。

## 役割・なぜ必要か
- `$_GET` / `$_POST` を直接触らず、**フレームワーク経由で安全に入力を取得**するため（フィルタリングや型の一貫性が効く）。
- レスポンスも `echo` で垂れ流さず、**ステータスコードやヘッダを明示して返す**ため。API（JSON返却）でもHTML画面でも同じ作法で扱える。
- コントローラ基底クラスが `$this->request` / `$this->response` を最初から持っている。

## 基本の書き方（コード）
```php
<?php
namespace App\Controllers;

use CodeIgniter\Controller;

class UserController extends Controller
{
    public function search()
    {
        // ── 入力の取得（メソッドの使い分けが肝）──
        $q     = $this->request->getGet('q');        // クエリ文字列 ?q=...
        $name  = $this->request->getPost('name');    // POSTボディ（フォーム）
        $any   = $this->request->getVar('id');       // GET/POST/どちらでも
        $all   = $this->request->getPost();          // 引数なし＝POST全部を配列で

        // JSON ボディ（API でフロントから送られる）
        $json  = $this->request->getJSON();          // オブジェクト
        $arr   = $this->request->getJSON(true);      // 連想配列で欲しいとき
        $email = $this->request->getJSON()->email ?? null;

        // その他
        $ip     = $this->request->getIPAddress();
        $isAjax = $this->request->isAJAX();
        $method = $this->request->getMethod();       // 'get' / 'post' ...
        $file   = $this->request->getFile('avatar'); // アップロードファイル

        return view('users/search', ['results' => []]);
    }
}
```

### レスポンスの返し方
```php
// JSON を返す（API の定番。Content-Type も自動で application/json）
return $this->response->setJSON([
    'status' => 'ok',
    'data'   => $user,
]);

// ステータスコードを明示（作成・エラーなど）
return $this->response
            ->setStatusCode(201)            // 201 Created
            ->setJSON(['id' => $id]);

return $this->response->setStatusCode(404)->setJSON(['error' => 'not found']);

// 本文を直接セット（プレーンテキスト/独自フォーマット）
return $this->response->setBody('OK')->setHeader('X-Foo', 'bar');
```

### リダイレクト
```php
// 別URLへ
return redirect()->to('/users');
return redirect()->route('users.show', [$id]); // 名前付きルートへ

// 直前のページへ戻す（バリデーション失敗時の定番）
return redirect()->back()
                 ->withInput()                       // ★入力値を引き継ぐ
                 ->with('error', '保存に失敗しました'); // フラッシュメッセージ
```
```php
<!-- 戻った先のビューで old() / フラッシュを読む -->
<input name="email" value="<?= esc(old('email')) ?>">
<?php if (session('error')): ?>
    <p class="err"><?= esc(session('error')) ?></p>
<?php endif ?>
```

## 実務での使い方・定番パターン
- **入力取得メソッドの選択基準**：フォームPOSTは `getPost()`、URLのクエリは `getGet()`、両対応にしたいルートは `getVar()`、API は `getJSON()`。「とりあえず `getVar`」ではなく**送られてくる経路に合わせる**と意図が明確になりバグりにくい。
- **API の戻りは `setJSON()` 一択**。手で `json_encode` + ヘッダ設定をするより安全・簡潔。`ResponseTrait`（`$this->respond()` / `$this->failValidationErrors()`）を使うと REST がさらに楽。
- **PRG パターン**：フォーム送信は処理後に必ず `redirect()` で別URLへ（リロード二重送信防止）。失敗時は `back()->withInput()` で値を残す。
- 取得した入力は**そのまま信用せず必ず検証**してからモデルへ渡す。→ [validation.md](./validation.md)

## ハマりどころ / アンチパターン
- **`getJSON()` のデフォルトはオブジェクト**。配列前提で `$data['email']` と書くと壊れる。配列が欲しければ `getJSON(true)`。
- **`getPost()` で取れない**：実は JSON で送られている／メソッドが PUT・PATCH のケース。`getJSON()` や `getRawInput()`（PUTフォーム）を使う。経路を取り違えると常に `null`。
- **`withInput()` 忘れ**：`back()` だけだと入力値が消え、ユーザーが全部打ち直しになる。失敗リダイレクトには `->withInput()` をセットにする。
- **`return` し忘れ**：`redirect()->to()` や `setJSON()` は `return` して初めて効く。`return` を落とすと処理が続行して二重レスポンスになる。
- **`$_POST` / `$_GET` 直アクセス**：フレームワークの恩恵（一貫した取得・将来のフィルタ）を捨てることになる。必ず `$this->request` 経由で。

## 関連
[controllers.md](./controllers.md) / [validation.md](./validation.md) / [views.md](./views.md)
