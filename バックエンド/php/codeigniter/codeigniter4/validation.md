# バリデーション（CodeIgniter 4）

## ひとことで言うと
**バリデーション** は、リクエストやモデルに入ってくる値が想定どおりか（必須・形式・長さ等）を**処理前にチェック**する仕組み。コントローラでは `$this->validate($rules)`、モデルでは `$validationRules` で定義する。

## 役割・なぜ必要か
- 不正・欠損データがDBやビジネスロジックに入る前に**入口で弾く**ためにある。SQLや表示の前段で守る最初の関門。
- **サーバ側検証は必須**。クライアント（JS/HTML）の検証はUX用で、簡単に回避できるため信用してはいけない。

---

## 基本の書き方：コントローラで validate()
```php
// app/Controllers/Users.php
public function create()
{
    $rules = [
        'name'  => 'required|min_length[3]|max_length[100]',
        'email' => 'required|valid_email|is_unique[users.email]',
        'age'   => 'permit_empty|integer|greater_than[0]',
        'pass'  => 'required|min_length[8]',
        'pass_confirm' => 'required|matches[pass]',
    ];

    if (! $this->validate($rules)) {
        // 失敗：エラーを取り出してビューへ
        return view('users/new', [
            'errors' => $this->validator->getErrors(),
        ]);
    }

    // 成功：検証済みデータで処理
    $data = $this->request->getPost(['name', 'email', 'age']);
    // ... 保存処理 ...
    return redirect()->to('/users')->with('message', '登録しました');
}
```
- `$this->validate()` は `Controller` 基底クラスのメソッド。成功で `true`、失敗で `false`。
- 失敗時は `$this->validator->getErrors()`（連想配列：フィールド名 => メッセージ）でまとめて、`$this->validator->getError('email')` で個別取得。

## ルール記法
```php
'email' => 'required|valid_email|min_length[3]'
```
- **パイプ `|` 区切り**で複数ルールを連結。`[...]` でパラメータを渡す（`min_length[3]`、`matches[pass]`）。
- よく使う組込ルール：`required` / `permit_empty`（空ならスキップ）/ `valid_email` / `integer` / `numeric` / `min_length[n]` / `max_length[n]` / `greater_than[n]` / `in_list[a,b,c]` / `is_unique[table.col]` / `is_not_unique[table.col]` / `matches[field]` / `regex_match[/.../]`。
- 配列記法でカスタムメッセージも指定できる：
```php
$rules = [
    'email' => [
        'rules'  => 'required|valid_email',
        'label'  => 'メールアドレス',
        'errors' => [
            'required'    => '{field}は必須です',
            'valid_email' => '{field}の形式が不正です',
        ],
    ],
];
```

## ルールセットを Config に置く（再利用）
```php
// app/Config/Validation.php
public array $signup = [
    'name'  => 'required|min_length[3]',
    'email' => 'required|valid_email|is_unique[users.email]',
];
public array $signup_errors = [
    'email' => ['is_unique' => 'そのメールは既に使われています'],
];
```
```php
// コントローラから名前で呼ぶ
if (! $this->validate('signup')) { /* ... */ }
```
- 同じルールを複数箇所で使うなら **Config\Validation にまとめてDRYに**。

## Model の $validationRules との関係
```php
// app/Models/UserModel.php
class UserModel extends \CodeIgniter\Model
{
    protected $validationRules = [
        'email' => 'required|valid_email|is_unique[users.email]',
        'name'  => 'required|min_length[3]',
    ];
    protected $validationMessages = [
        'email' => ['is_unique' => 'メール重複'],
    ];
    protected $skipValidation = false;  // trueで検証スキップ（非推奨）
}
```
```php
// save/insert/update 時に自動で検証が走る
if (! $userModel->save($data)) {
    $errors = $userModel->errors();   // Modelのエラー取得
}
```
- **Model経由（`save`/`insert`/`update`）なら `$validationRules` が自動適用**される（`database_query_builder.md` 参照：Builder直叩きでは走らない）。
- コントローラの `validate()` は**入口（リクエスト形）**、Modelの検証は**保存直前（永続化形）**。役割が違うので、フォーム固有チェックはコントローラ、データ整合性はModel、と分けると整理しやすい。

## 実務での定番パターン
- 失敗時は **入力値を保持して再表示**（`old()` ヘルパ／`set_value()`）し、`getErrors()` をフィールド横に出す。
- API（JSON）なら失敗時に `422` とエラー配列を返す（`request_response.md` 参照）：
```php
if (! $this->validate($rules)) {
    return $this->response->setStatusCode(422)
        ->setJSON(['errors' => $this->validator->getErrors()]);
}
```

## ハマりどころ / アンチパターン
- **サーバ検証を省略**：JSやHTML5の`required`だけに頼ると簡単に突破される。サーバ側 `validate()` は必須。
- **ルール記法ミス**：`min_length 3` ではなく `min_length[3]`。`|` と `[]` を取り違えると検証が効かない／例外。
- **`permit_empty` の付け忘れ**：任意項目で空値が `valid_email` 等に引っかかる。任意なら先頭に `permit_empty`。
- **エラー表示忘れ**：`validate()` が false なのにそのまま処理を続けない。必ず `getErrors()` を表示・返却して早期return。
- **`is_unique` だけに依存**：競合で破れることがある。DB側にも unique 制約（`database_query_builder.md`）を張る。

## 関連: [models.md](./models.md) / [request_response.md](./request_response.md)
