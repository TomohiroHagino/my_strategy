# バリデーション / フォームリクエスト（Laravel 11）

## ひとことで言うと
受け取ったリクエスト値が**正しい形か**をルールで検証し、ダメなら自動でエラー表示（または422）に戻す仕組み。検証済みの値だけを安全に取り出せる。

## 役割・なぜ必要か
- 外部入力（フォーム／JSON）は信用できない。**処理前に「必須か」「メール形式か」「重複しないか」を検査**し、ダメなら入力画面へ戻す・APIなら 422 を返すためにある。
- Railsの **Strong Parameters（許可リスト）＋ model の validates** が一体になったもの、と捉えると分かりやすい。Laravelは「**検証ルール**」を通すことで、許可＋形式チェックを同時に行い、`validated()` で**検証済みの値だけ**を受け取れる（許可していないキーは含まれない＝mass assignment 対策にもなる）。→ [strong_parameters.md](../../../ruby/rails/rails7/strong_parameters.md) の対比。
- 「どこで検証するか」を整理することで、コントローラを薄く保てる。

## 基本の書き方（コード）
**(A) コントローラ内で `$request->validate()`**（手軽）
```php
public function store(Request $request)
{
    // 失敗時：webは自動で前のページへリダイレクト＋エラー、apiは422 JSON
    $validated = $request->validate([
        'title' => ['required', 'string', 'max:255'],
        'email' => ['required', 'email', 'unique:users,email'],
        'tags'  => ['array'],            // 配列
        'tags.*'=> ['integer'],          // 配列の各要素
    ]);

    Post::create($validated);            // $validated は検証済みキーだけ
    return redirect('/posts')->with('status', '作成しました');
}
```

**(B) フォームリクエスト**（再利用・肥大化対策の本命）
```bash
php artisan make:request StorePostRequest
```
```php
// app/Http/Requests/StorePostRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        // 認可：このユーザーがこの操作をしてよいか（falseなら403）
        return $this->user()?->can('create', \App\Models\Post::class) ?? false;
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'body'  => ['required'],
        ];
    }

    public function messages(): array          // 任意：日本語メッセージ
    {
        return ['title.required' => 'タイトルは必須です'];
    }
}
```
```php
// コントローラ：型ヒントするだけで「検証→authorize」が自動実行される
public function store(StorePostRequest $request)
{
    Post::create($request->validated());   // ここに来た時点で検証済み
    return redirect('/posts');
}
```

## 実務での使い方・定番パターン
- **エラーの表示**：失敗するとエラーは**セッションにフラッシュ**され、ビューで `$errors`（全ビューに自動共有）から読める。入力値は `old()` で復元する。→ [blade.md](./blade.md)
  ```blade
  <input name="title" value="{{ old('title') }}">
  @error('title') <p class="text-red-500">{{ $message }}</p> @enderror
  ```
- **フォームリクエストに寄せる**：複数アクションで同じルールを使う、ルールが長い、認可も絡む——なら `make:request`。`store`/`update` で別クラスにし、共通部分は基底クラスに切り出すと散らからない。
- **`update` の unique 除外**：自分自身は重複扱いしない。`Rule::unique('users')->ignore($this->user)`。
- **カスタムルール**：`php artisan make:rule Uppercase` で `validate()` を実装、または `rules()` 内にクロージャを書く。
  ```php
  'code' => [function ($attr, $value, $fail) {
      if ($value === 'banned') $fail('使用できません');
  }],
  ```
- **`prepareForValidation()`** で検証前に整形（トリム・小文字化）。**`passedValidation()`** で検証後フックも可能。

## ハマりどころ / アンチパターン
- **`authorize()` が `false` で 403**：フォームリクエストは `make:request` 直後 `authorize()` が `return false;` のことがあり、**全リクエストが 403** になる。認可不要なら `return true;`、必要なら Policy/Gate を書く。「なぜか403」の典型原因。
- **`validate()` と `validated()` の取り違え**：`$request->validate([...])` は**検証＋検証済み配列を返す**。フォームリクエスト側では `$request->validated()` で取り出す（`->all()` だと未検証キーまで混ざる）。
- **バリデーション場所の散らかり**：あるアクションは `$request->validate()`、別は手書きの `if`、別はフォームリクエスト…と混在すると保守不能。**チームで「フォームリクエストに寄せる」など方針を1つに**。
- **配列ルールの書き忘れ**：`tags` を `['array']`、各要素を `tags.*` で検証しないと中身が素通り。Railsの `tag_ids: []` 同様、配列は明示が要る。
- **API で `validate()` の挙動**：`Accept: application/json` なら自動で 422＋JSON（`errors` キー）。HTMLフォームならリダイレクト。クライアント種別で戻りが変わる点に注意。

## 関連
[controller.md](./controller.md) / [blade.md](./blade.md) / [auth.md](./auth.md)
