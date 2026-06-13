# PHP 8.4（言語解説）

## ひとことで言うと
リクエストごとに走る shared-nothing なサーバサイド言語。動的型に型宣言を足し、連想配列・トレイト・`match`/enum・属性（Attributes）で書く。8.4 ではプロパティフック、非対称可視性、`new` の括弧省略が入り、オブジェクト指向の表現力がさらに上がった。

## このバージョンの位置づけ（リリース時期 / サポート・EOL / どこで使うか）
- リリース: PHP 8.4.0 は 2024年11月21日。
- サポート: 各マイナー版は約2年のアクティブサポート＋約2年のセキュリティ修正で計約4年。8.4 はおおむね2026年末までアクティブ、2028年末ごろまでセキュリティ修正。本番は 8.1 以上、できれば 8.2〜8.4 を使う。
- どこで使うか: Web アプリ/API（Laravel・Symfony・CodeIgniter）、CMS（WordPress 等）、CLI バッチ（`php artisan` / `symfony console`）。実行は PHP-FPM ＋ nginx が定番。

## 言語の基本（文法の要点）
- ファイルは `<?php` で始まり、HTML に埋め込みもできるが、現代は純 PHP ファイルが基本。文末は `;`。変数は `$` 始まり。
```php
<?php
function greet(string $name): string {
    return "Hello, {$name}";
}
echo greet("Aki");
```
- 型宣言（任意だが推奨）: 引数・戻り値・プロパティに型を書ける。union 型 `int|string`、nullable `?int`、`never`、`void`。
```php
function half(int|float $n): float {
    return $n / 2;
}
```
- `match` 式（厳密一致・値を返す・フォールスルーなし）:
```php
$label = match($status) {
    "pass"        => "合格",
    "fail"        => "不合格",
    default       => "不明",
};
```
- アロー関数とクロージャ（無名関数）:
```php
$double = fn(int $n): int => $n * 2;          // 外側の変数を自動キャプチャ
$nums   = array_map($double, [1, 2, 3]);      // [2, 4, 6]
```
- コンストラクタプロモーション（引数宣言とプロパティ定義を同時に）:
```php
class User {
    public function __construct(
        public readonly string $name,
        public int $age = 20,
    ) {}
}
```
- 名前付き引数:
```php
new User(name: "Aki", age: 30);
```
- null 合体 `??` と null 合体代入 `??=`、null セーフ呼び出し `?->`:
```php
$bio = $user?->profile?->bio ?? "未設定";
```

## この言語の核心概念（他言語と違う・必ず押さえる）

### 連想配列が主役（リストもマップも同じ array）
- 何か: PHP の `array` は「順序付き連想配列」一本で、添字配列（リスト）も連想配列（マップ）も同じ型。挿入順を保持し、`array_*` 関数群で操作する。
```php
$list = [1, 2, 3];                          // 実体はキー 0,1,2 の連想配列
$user = ["name" => "Aki", "age" => 30];     // 文字列キーのマップ
$user["role"] = "admin";                    // キー追加
$names = array_map(fn($u) => $u["name"], $users);
$adults = array_filter($users, fn($u) => $u["age"] >= 18);
$total  = array_reduce([1,2,3], fn($c, $n) => $c + $n, 0);   // 6
array_is_list([1,2,3]);                     // true（連番キーか判定・8.1+）
```
- 違う点 / つまずき: Python の list と dict、JS の Array と Object のような型の区別が無い。`array_filter` はキーを保持するので穴あきになり、`[0]` で取れず詰まる→ `array_values()` で振り直す。「配列」が無数の役割を兼ねるので、API 境界では何が入るのか型で表しにくい。

### 型ジャグリングと `===` vs `==`
- 何か: `==` は型をまたいで暗黙変換してから比較する（緩い比較）。`===` は型も値も一致を要求する（厳密比較）。
```php
0 === "0";        // false（型が違う）
0 == "0";         // true
"1" == 1;         // true（数値文字列は数値化）
0 == "";          // false（PHP 8.0 で改善。7.x では true だった）
0 == "abc";       // false（PHP 8.0 で改善。7.x では 0 == "abc" が true で事故多発）
null == false;    // true
in_array("1", [1, 2], true);   // 第3引数 true で厳密比較
```
- 違う点 / つまずき: 多くの言語の `==` は厳密寄りだが PHP の `==` は変換が激しい。PHP 8.0 で「数値 vs 非数値文字列」の比較が文字列側基準に直り `0 == "abc"` は false になったが、`"1" == 1` 等は今も真。原則 `===` を使い `declare(strict_types=1)` を併用する。

### shared-nothing 実行モデル（リクエスト都度プロセス）
- 何か: 1リクエスト＝1回スクリプトを最初から実行し、終わると全状態が消える。リクエスト間でメモリ上の変数を共有しない（共有は DB/Redis/セッション経由）。`$_GET`/`$_POST`/`$_SERVER` などスーパーグローバルでリクエスト情報を受け取る。
```php
$id   = $_GET["id"]    ?? null;        // クエリ文字列
$name = $_POST["name"] ?? null;        // フォーム送信
$ua   = $_SERVER["HTTP_USER_AGENT"];   // リクエストヘッダ
// このスクリプトが返したら、上の変数は消える。次のリクエストはまっさら
```
- 違う点 / つまずき: Node.js/Go/Java は常駐プロセスがリクエスト間でグローバル変数やキャッシュをメモリ保持できるが、PHP は基本できない（毎回リセット）。これは水平スケールが楽な反面、「起動時に1回読み込んで常駐」が効かない。常駐させたいときだけ Swoole / RoadRunner / Laravel Octane を使う（その途端、リクエスト間で状態が残るので従来の前提が崩れバグになりやすい）。

### `$this` とトレイト（水平再利用）
- 何か: メソッド内で自分自身を指すのは `$this`（明示的に書く）。トレイトは単一継承の制約下でメソッド群を複数クラスに「水平に」混ぜ込む仕組み。
```php
trait Timestampable {
    public ?string $createdAt = null;
    public function touch(): void { $this->createdAt = date("c"); }  // $this を使える
}
trait Identifiable {
    public function shortId(): string { return substr($this->id, 0, 8); }
}
class Post {
    use Timestampable, Identifiable;     // 複数トレイトを合成
    public function __construct(public string $id) {}
}
$p = new Post("abcdef123456");
$p->touch();
```
- 違う点 / つまずき: Python/JS のメソッドは `self`/暗黙の `this` 規約だが、PHP は必ず `$this->` を書く（省略するとローカル変数扱い）。トレイトは継承ではなくコピペ的合成なので、同名メソッドが衝突したら `insteadof` / `as` で明示解決が要る。Ruby の module とは違いメソッド解決順序ではなく単純展開。

### 参照 `&`（代入の参照渡し・foreach の罠）
- 何か: PHP の変数代入・引数渡しは基本コピー（値渡し、配列も含む）。`&` を付けたときだけ同じ実体を指す参照になる。
```php
$a = [1, 2, 3];
$b = $a;                 // コピー（独立）
$b[] = 4;                // $a は変わらない
$c = &$a;                // 参照（同じ実体）
$c[] = 4;                // $a も [1,2,3,4] に変わる

foreach ($items as &$item) { $item *= 2; }   // 参照で各要素を書き換え
unset($item);            // ★これを忘れると次の foreach で $item が残り最後の要素を破壊
```
- 違う点 / つまずき: 配列が値渡し（コピー）なのは Python/JS と逆で驚きやすい一方、`foreach (... as &$item)` の後に `unset($item)` を忘れると、参照が居残って後続の `foreach` で最後の要素が上書きされる典型バグを踏む。`&` は必要なときだけ、使ったら必ず `unset`。
- 「配列はコピー・オブジェクトは参照・`&` で明示参照」を他言語と並べた図解 → [../../../言語共通概念/値渡しと参照.md](../../../言語共通概念/値渡しと参照.md)

### `null` と Nullsafe `?->`、enum
- 何か: 値が無いことは `null`。`?->` は左が `null` なら呼ばずに `null` を返す。enum（8.1+）は取りうる値を型として固定し、メソッドも持てる。
```php
$bio = $user?->profile?->bio ?? "未設定";   // どこかが null でも例外にならない

enum Status: string {
    case Active = "active";
    case Banned = "banned";
    public function label(): string {
        return match($this) {
            Status::Active => "有効",
            Status::Banned => "凍結",
        };
    }
}
Status::from("active");        // Status::Active（不正値は例外）
Status::tryFrom("xxx");        // null（例外を投げたくないとき）
Status::Active->value;         // "active"
Status::cases();               // 全ケースの配列
```
- 違う点 / つまずき: enum はただの定数ではなくオブジェクトなので `===` で安全に比較でき、`switch` の文字列ミスタイプ事故を防げる。`from()` は不正値で例外、`tryFrom()` は null を返すので使い分ける。`?->` は付けすぎると null が静かに連鎖し、原因箇所が追えなくなるため境界で明示検証する。

## 型・データモデル
- 動的型＋漸進的型付け: 型宣言は任意。付けると実行時の型エラー検出に加え、PHPStan / Psalm の静的解析で安全度が上がる。`declare(strict_types=1);` を先頭に書くと暗黙の型変換を止められる。
- スカラー型: `int` `float` `string` `bool`、`null`。複合型: `array`、`object`、`callable`、`iterable`（連想配列・enum は「この言語の核心概念」を参照）。
- `readonly`: 初期化後に変更不可なプロパティ/クラス。値オブジェクトに向く。

## この言語らしさ / 特徴的な機能
（shared-nothing 実行モデル・トレイト・参照 `&`・enum は「この言語の核心概念」を参照）
- 属性（Attributes・8.0+）: クラスやメソッドにメタ情報を `#[...]` で付ける。フレームワークのルーティングや ORM マッピングに使われる。
```php
#[Route("/users", methods: ["GET"])]
public function index(): array { return []; }
```
- インターフェイスと抽象クラスで契約を定義。`instanceof` で型判定。
- 例外処理は `try/catch/finally`、`throw` は式としても使える。
```php
$value = $maybe ?? throw new InvalidArgumentException("missing");
```

## 並行・非同期
- 基本は同期1リクエスト処理。並行が必要なら以下を使う。
- Fibers（8.1+）: 実行を中断・再開できる低レベル機構。これを土台に ReactPHP / Amp などの非同期ランタイムがノンブロッキング I/O を提供する。アプリで直接 Fiber を書くことは少なく、ライブラリ越しに使うのが普通。
```php
$fiber = new Fiber(function (): void {
    $value = Fiber::suspend("paused");   // 中断して値を返す
    echo "resumed with {$value}";
});
$fiber->start();          // "paused" を受け取る
$fiber->resume("go");     // "resumed with go"
```
- プロセスレベルの並行: PHP-FPM が複数ワーカープロセスでリクエストを並列処理する（言語というより実行基盤の並行）。
- 常駐・長時間処理: Swoole / RoadRunner / Laravel Octane でアプリをメモリ常駐させ、コルーチンで並行 I/O を捌く。重いバックグラウンドは Laravel Queue 等のジョブに逃がす。

## 標準ライブラリ / ツールチェーン
- パッケージ管理: Composer（`composer.json` に依存、`composer install` で `composer.lock` 固定）。配布は Packagist。
- オートロード: PSR-4（名前空間とディレクトリを対応づける）。`composer dump-autoload`。
- 標準関数群: 配列（`array_map`/`array_filter`/`array_reduce`）、文字列（`str_contains`/`str_starts_with`/`sprintf`）、JSON（`json_encode`/`json_decode`）、ファイル、`PDO`（DB アクセス、プレースホルダで SQL インジェクション対策）。
```php
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);          // パラメータ化クエリ
```
- 品質ツール: PHPStan / Psalm（静的解析）、PHP CS Fixer / Pint（整形）。テストは PHPUnit / Pest。
- 標準仕様 PSR: PSR-4（オートロード）、PSR-7（HTTP メッセージ）、PSR-12（コーディング規約）など、相互運用の共通ルール。

## このバージョンの新機能・トピック
- プロパティフック（property hooks）: プロパティに `get`/`set` のロジックを直接書ける。getter/setter メソッドを別途用意しなくてよい。
```php
class User {
    public string $fullName {
        get => "{$this->first} {$this->last}";
        set (string $value) {
            [$this->first, $this->last] = explode(" ", $value, 2);
        }
    }
    public function __construct(public string $first, public string $last) {}
}
$u = new User("Aki", "Sato");
echo $u->fullName;          // "Aki Sato"（get フック）
$u->fullName = "Ken Ito";   // set フックで分解
```
- 非対称可視性（asymmetric visibility）: 読み取りと書き込みで可視性を変えられる。外からは読めるが書けない、を1行で表現。
```php
class Counter {
    public private(set) int $count = 0;   // 読み取りは public / 書き込みは private
    public function inc(): void { $this->count++; }
}
```
- `new` の括弧省略（new without parentheses）: 生成直後のメソッド/プロパティアクセスに括弧が不要になった。
```php
$name = new User("Aki", "Sato")->fullName;   // 8.3 までは (new User(...))->fullName が必要
```
- `#[\Deprecated]` 属性: 関数・メソッドの非推奨をコードで宣言でき、呼び出すと警告が出る。
- 新しい配列関数: `array_find` / `array_any` / `array_all` などが追加され、ループを書かずに条件検索ができる。
```php
$adult = array_find($users, fn($u) => $u->age >= 18);
```
- DOM/HTML5 対応の新クラス、`BcMath\Number` オブジェクトなど標準ライブラリの拡充。

## ハマりどころ
- 緩い比較 `==` の罠: `0 == "a"` などの暗黙変換が事故のもと。基本は厳密比較 `===` を使い、`declare(strict_types=1)` を入れる。
- 配列の値渡し: 配列は代入・引数渡しでコピーされる（参照ではない）。意図的に共有したいときだけ `&`。
- null セーフ `?->` の付けすぎ: nil が静かに伝播してバグの原因が追いにくくなる。境界で明示的に検証する。
- 文字列補間の波括弧: `"{$obj->prop}"` は OK だが、`"$obj->prop->nested"` のような深い式は `{}` で囲む。
- 静的解析なしの大規模開発: 動的型ゆえ型バグが実行時まで出ない。PHPStan/Psalm 併用が前提。
- 古い記事の混入: 歴史が長く「8.x 以前の古い書き方」の記事が多い。年代と PHP バージョンを確認する。

## 関連
- フレームワーク: [../laravel/](../laravel/)（Laravel）/ [../codeigniter/](../codeigniter/)（CodeIgniter）。
- 言語概要: [../README.md](../README.md)。
