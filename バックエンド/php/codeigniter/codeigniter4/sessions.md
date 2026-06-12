# セッション（Session）（CodeIgniter 4）

## ひとことで言うと
- **セッション = リクエストをまたいで値を保持する仕組み**（HTTP はステートレスなので、ログイン状態などを覚えるために使う）。
- CI4 では **`session()` ヘルパ**（または `\Config\Services::session()`）で扱う。
- 値の保存先は**ドライバ**で選べる：**File / Redis / Database / Memcached**。

## 役割・なぜ必要か
- ログイン状態・カート・直前の入力値など、**ページをまたいで覚えておきたい状態**を保持する。
- Cookie はサイズが小さく改ざんリスクがあるため、実体はサーバ側（ファイル/DB/Redis）に置き、**Cookie にはセッションIDだけ**を持たせるのが基本。
- フォーム送信後のメッセージ（成功/失敗）など、**1回だけ表示して消したい値**は「フラッシュデータ」で扱う。

## 基本の書き方（コード）
### 取得・保存（session() ヘルパ）
```php
$session = session();             // シングルトン取得

// set：保存（キー=値、または配列でまとめて）
$session->set('user_id', 42);
$session->set(['user_id' => 42, 'role' => 'admin']);

// get：取得（無ければ null）
$userId = $session->get('user_id');
$role   = session('role');        // 引数つき呼び出しでも取得できる

// has / remove / destroy
if ($session->has('user_id')) { /* ログイン中 */ }
$session->remove('user_id');      // 個別削除
$session->destroy();              // セッション全消去（ログアウト時など）
```

### フラッシュデータ（次の1リクエストだけ生存）
```php
// 保存（リダイレクト前にメッセージを積む）
$session->setFlashdata('message', '保存しました');

// 取得（次のリクエストで読む。読んだ後、自動で消える）
$msg = $session->getFlashdata('message');

// View 側
// <?= session()->getFlashdata('message') ?>

// keepFlashdata：もう1リクエスト延命したいとき
$session->keepFlashdata('message');
```

### Tempdata（指定秒数だけ生存）
```php
// 第3引数 = 寿命（秒）。300秒後に自動で消える
$session->setTempdata('otp', '123456', 300);
$otp = $session->getTempdata('otp');
```

### Config\Session でドライバ設定（app/Config/Session.php）
```php
<?php
namespace Config;
use CodeIgniter\Config\BaseConfig;
use CodeIgniter\Session\Handlers\FileHandler;
// use CodeIgniter\Session\Handlers\RedisHandler;
// use CodeIgniter\Session\Handlers\DatabaseHandler;

class Session extends BaseConfig
{
    // ドライバ（保存先ハンドラ）
    public string $driver = FileHandler::class;
    public string $cookieName = 'ci_session';
    public int $expiration = 7200;          // 秒。0 でブラウザを閉じるまで
    public string $savePath = WRITEPATH . 'session'; // File の場合の保存先
    public bool $matchIP = false;
    public int $timeToUpdate = 300;         // セッションID再生成の間隔（秒）
    public bool $regenerateDestroy = false;
}
```
```bash
# Redis ドライバに切り替える例（.env で savePath を上書き）
# .env
# session.driver = 'CodeIgniter\Session\Handlers\RedisHandler'
# session.savePath = 'tcp://127.0.0.1:6379'
```

## 実務での使い方・定番パターン
- **ログイン状態の保持**：`$session->set('user_id', $id)` → 以降のリクエストで `session('user_id')` を確認。ログアウトは `$session->destroy()`。
- **PRG パターン（Post/Redirect/Get）**：フォーム処理後は `setFlashdata` でメッセージを積んでリダイレクト。再読み込みでの二重送信を防ぐ。
- **Filters と併用**：認証チェックを Filter に置き、未ログインなら `redirect()->to('/login')`。セッションの確認はここに集約する。
- **本番は File 以外を推奨**：複数台構成（ロードバランサ配下）では Redis / Database ドライバを使い、**全ノードでセッションを共有**する。
- **ログイン直後にセッションID再生成**：固定化攻撃対策に `$session->regenerate()` を呼ぶ。

## ハマりどころ / アンチパターン
- **保存ドライバの設定ミス**：Redis を指定したのに接続情報（`savePath`）が間違っていて、セッションが毎回飛ぶ。`.env` の `session.driver` / `session.savePath` を環境ごとに正しく設定する。
- **フラッシュデータの寿命を誤解**：`setFlashdata` は**次の1リクエストで消える**。リダイレクトを挟まず同一リクエストで読もうとすると意図とズレる。「次のページで1回だけ表示」が正しい使い所。
- **File ドライバのまま本番スケール**：複数サーバだと**サーバごとに別ファイル**になり、ログインが維持されない／ランダムにログアウトされる。本番の水平スケールでは Redis / Database に切り替える。
- **`destroy()` 後に同じインスタンスを使い回す**：ログアウト処理後は値を読み書きしない。新規ログイン時は `regenerate()` でIDを作り直す。
- **`writable/session` の権限不足**：File ドライバで保存先に書き込めずセッションが効かない。`writable/` の書き込み権限を確認。
- **大きなオブジェクトをセッションに詰める**：肥大化して読み書きが重くなる。最小限のID等だけ置き、本体はDBから引く。

## 関連
[config_env.md](./config_env.md) / [auth_shield.md](./auth_shield.md)
