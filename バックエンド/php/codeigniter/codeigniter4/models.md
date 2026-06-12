# モデル（Model）（CodeIgniter 4）

## ひとことで言うと
DBのテーブル1つに対応するクラスで、**Query Builder のラッパ＋バリデーション・タイムスタンプ・ソフトデリート等を内蔵**したデータアクセス層。`CodeIgniter\Model` を継承する。

## 役割・なぜ必要か
- テーブル `users` ↔ クラス `UserModel`、という対応で「生SQLを毎回書かずにPHPでCRUDを扱う」ためにある。
- 素の Query Builder（`$db->table()`）と違い、**`find()`/`save()` 等の高水準メソッド**、**保存時の自動バリデーション**、**`created_at`/`updated_at` の自動付与**、**ソフトデリート**をフレームワークが面倒みてくれる。
- MVC の中で「データの取得・保存ルール」をここへ集約し、コントローラを薄く保つ。

## 基本の書き方（コード）
```php
<?php
// app/Models/UserModel.php
namespace App\Models;

use CodeIgniter\Model;

class UserModel extends Model
{
    protected $table         = 'users';      // 対応テーブル
    protected $primaryKey    = 'id';         // 主キー（既定 id）
    protected $returnType    = 'array';      // 'array' か Entityクラス
    protected $useTimestamps = true;         // created_at/updated_at 自動
    protected $useSoftDeletes = true;        // deleted_at で論理削除
    protected $dateFormat    = 'datetime';   // datetime/date/int

    // ★最重要：mass assignment を許すカラムのホワイトリスト
    protected $allowedFields = ['name', 'email', 'role'];

    // モデル内バリデーション（save/insert/update 時に自動実行）
    protected $validationRules = [
        'email' => 'required|valid_email|is_unique[users.email,id,{id}]',
        'name'  => 'required|max_length[50]',
    ];
    protected $validationMessages = [
        'email' => ['is_unique' => 'そのメールは既に使われています'],
    ];
    protected $skipValidation = false; // true で検証スキップ（非推奨）
}
```

## 実務での使い方・定番パターン
```php
$model = new \App\Models\UserModel();

// 取得
$user  = $model->find(1);                       // 主キー1件（無ければ null）
$all   = $model->findAll();                      // 全件
$one   = $model->where('email', $mail)->first(); // 条件1件
$page  = $model->where('role', 'admin')
               ->orderBy('id', 'DESC')
               ->paginate(20);                   // ページネーション

// 作成（戻り値は挿入ID、検証失敗なら false）
$id = $model->insert(['name' => 'たろう', 'email' => 't@example.com']);

// 更新（第1引数=主キー）
$model->update(1, ['name' => 'じろう']);

// save：主キーの有無で insert/update を自動判別（便利）
$model->save(['id' => 1, 'name' => '更新']); // id あり→update
$model->save(['name' => '新規']);            // id なし→insert

// 削除（useSoftDeletes=true なら deleted_at をセットするだけ）
$model->delete(1);
$model->onlyDeleted()->findAll();   // 論理削除済みだけ
$model->withDeleted()->findAll();   // 削除済みも含む

// 検証失敗時のエラー取得
if ($model->insert($data) === false) {
    $errors = $model->errors(); // ['email' => '...'] を view へ
}
```
- **Entity クラス併用**：`$returnType = \App\Entities\User::class` にすると、取得結果が配列でなく Entity オブジェクトになり、`$user->display_name` のようなドメインロジックや getter/setter を持てる。
```php
<?php
// app/Entities/User.php
namespace App\Entities;
use CodeIgniter\Entity\Entity;

class User extends Entity
{
    protected $casts = ['is_active' => 'boolean'];
    public function getDisplayName(): string  // $user->display_name で呼べる
    {
        return $this->attributes['name'] ?? '名無し';
    }
}
```
- **コールバック**：`$beforeInsert`/`$afterUpdate` 等にメソッド名を登録し、保存前後の加工（パスワードハッシュ化など）を行える。
- 取得系は Query Builder のメソッド（`where`/`join`/`like`/`select`）をそのままチェーンできる。→ [database_query_builder.md](./database_query_builder.md)

## ハマりどころ / アンチパターン
- **`$allowedFields` 未設定（空）だと insert/update してもカラムが保存されない**。CI4 最頻出の罠。保存したいカラムは必ずここに列挙する（主キー・タイムスタンプは含めない）。
- **`$useTimestamps = true` なのに `created_at`/`updated_at` カラムが無い** → DBエラー。マイグレーションで両カラムを作る。`$useSoftDeletes = true` も同様に `deleted_at`（nullable）が必須。
- **`insert()`/`save()` の戻り値 `false` を握り潰す**：検証失敗でも例外は出ない。戻り値を必ず確認し `$model->errors()` を見る（Rails の `save` と同じ注意）。
- **`is_unique` の自分自身除外忘れ**：更新時に `is_unique[users.email,id,{id}]` の `,id,{id}` を付けないと自分のメールで弾かれる。
- **Entity と array の混同**：`$returnType` を変えると `$user['name']` ↔ `$user->name` が変わる。呼び出し側と揃える。

## 関連
[database_query_builder.md](./database_query_builder.md) / [validation.md](./validation.md) / [controllers.md](./controllers.md)
