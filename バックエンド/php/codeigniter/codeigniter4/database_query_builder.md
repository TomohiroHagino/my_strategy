# DB / Query Builder / マイグレーション（CodeIgniter 4）

## ひとことで言うと
**Query Builder** は、SQLをPHPのメソッド連結で組み立てDBを操作する仕組み。`$db->table('users')->where(...)->get()` のように書き、**値は自動でバインド（プレースホルダ化）されてSQLインジェクションを防ぐ**。マイグレーションはスキーマ変更をPHPで記述しバージョン管理する。

## 役割・なぜ必要か
- 生SQLの文字列結合をやめ、**安全（自動エスケープ／バインド）かつDB横断（MySQL/PostgreSQL/SQLite）** に書けるようにするためにある。
- スキーマ管理（マイグレーション）・初期データ投入（シーダ）・トランザクションまでをフレームワーク標準で提供し、チームでDBの状態を再現可能にする。

---

## 接続：\Config\Database::connect()
```php
$db = \Config\Database::connect();        // 既定グループ（.envのdatabase.default.*）
$db = \Config\Database::connect('tests'); // 別グループを指定
```
- 接続設定は `app/Config/Database.php` ＋ `.env`（`database.default.hostname` 等）。**接続情報は .env に置きコミットしない**。
- Modelの中では `$this->db` で同じ接続を共有できる。

## Query Builder の基本
```php
$db = \Config\Database::connect();
$builder = $db->table('users');

// SELECT（?で自動バインド → 安全）
$users = $builder->where('active', 1)
                 ->where('age >=', 20)
                 ->orderBy('created_at', 'DESC')
                 ->limit(10)
                 ->get()
                 ->getResult();        // オブジェクト配列。getResultArray()で連想配列

$one = $db->table('users')->where('id', $id)->get()->getRow(); // 1行

// INSERT / UPDATE / DELETE
$db->table('users')->insert(['name' => $name, 'email' => $email]);
$id = $db->insertID();
$db->table('users')->where('id', $id)->update(['name' => $newName]);
$db->table('users')->where('id', $id)->delete();

// JOIN・集計・LIKE（LIKEもエスケープされる）
$rows = $db->table('posts p')
           ->select('p.id, p.title, u.name')
           ->join('users u', 'u.id = p.user_id')
           ->like('p.title', $keyword)
           ->get()->getResultArray();
```
- `getResult()`=オブジェクト配列／`getResultArray()`=連想配列／`getRow()`=1件／`getRowArray()`=1件連想配列。
- **`where('age >=', 20)` のように演算子はキー側に書く**。値は常にバインドされる。

## 生クエリ（query）の注意
```php
// バインドを使えば生SQLも安全
$db->query('SELECT * FROM users WHERE email = ?', [$email]);
// NG：文字列結合は絶対に避ける（SQLインジェクション）
// $db->query("SELECT * FROM users WHERE email = '$email'");
```
- 複雑な集計や DB固有構文が必要なときだけ生クエリ。**必ず `?` バインドか名前付きバインドを使う**。

## Model 経由との使い分け
```php
// Model は内部で Query Builder を使い、検証・自動タイムスタンプ・ソフトデリートを足す
$users = $model->where('active', 1)->findAll();   // Modelでもbuilderメソッドが使える
```
- **基本はModel**（`models.md`）を使う：バリデーション・`created_at/updated_at` 自動更新・イベントが効く。
- **Query Builder直叩きは**「Modelに乗らない複雑JOIN・一時的な集計・バッチ処理」など限定的に。Modelを通さないと `$validationRules` は走らない点に注意。

## マイグレーション
```bash
php spark make:migration CreateUsersTable   # app/Database/Migrations/ に生成
php spark migrate            # 未適用を適用
php spark migrate:rollback   # 直前バッチを取り消し
php spark migrate:status     # 適用状況
php spark migrate:refresh    # 全rollback→再適用（開発用）
```
```php
// app/Database/Migrations/2026-..._CreateUsersTable.php
public function up()
{
    $this->forge->addField([
        'id'    => ['type' => 'INT', 'unsigned' => true, 'auto_increment' => true],
        'email' => ['type' => 'VARCHAR', 'constraint' => 255],
        'name'  => ['type' => 'VARCHAR', 'constraint' => 100, 'null' => true],
        'created_at datetime null',
        'updated_at datetime null',
    ]);
    $this->forge->addKey('id', true);          // 主キー
    $this->forge->addUniqueKey('email');       // ユニーク制約
    $this->forge->createTable('users');
}
public function down() { $this->forge->dropTable('users'); }
```
- **ファイル名の日時順 ＝ 適用順**。後から間に割り込ませると順序が崩れるので注意。

## シーダ（初期データ投入）
```bash
php spark make:seeder UserSeeder
php spark db:seed UserSeeder      # 実行
```
```php
public function run()
{
    $this->db->table('users')->insertBatch([
        ['email' => 'a@example.com', 'name' => 'A'],
        ['email' => 'b@example.com', 'name' => 'B'],
    ]);
    // 他のシーダ呼び出し: $this->call('RoleSeeder');
}
```

## トランザクション
```php
$db->transStart();                 // ここから
$db->table('orders')->insert($order);
$db->table('stocks')->where('id', $id)->update(['count' => $count - 1]);
$db->transComplete();              // 自動commit、途中で失敗なら自動rollback

if ($db->transStatus() === false) {
    log_message('error', '注文処理に失敗');
}
// 手動制御: transBegin() / transCommit() / transRollback()
```

## ハマりどころ / アンチパターン
- **Query BuilderとModelの使い分け**：何でもQuery Builder直叩きにするとバリデーション・タイムスタンプが効かない。原則Model、例外でBuilder。
- **マイグレーション順**：ファイル名の日時が適用順。外部キー先の親テーブルを後に作ると失敗する。順序を意識して命名。
- **生クエリの文字列結合**：`"... '$x'"` は厳禁。必ず `?` バインド。Query Builderの `where/like` は自動エスケープされるので原則そちらを使う。
- **`get()` の使い回し**：Builderは状態を持つ。`get()` 後は条件がリセットされる場合があるので、再利用せず都度 `$db->table()` から組み立てる。
- **本番マイグレーション**：カラム削除など破壊的変更はロック・不可逆に注意。段階移行（追加→移行→削除）が安全。

## 関連: [models.md](./models.md)
