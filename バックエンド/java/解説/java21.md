# Java 21（言語解説）

## ひとことで言うと
JVM 上で動く静的型付け・クラスベースのオブジェクト指向言語の LTS 版で、`record`・`sealed`・パターンマッチが揃い、目玉として OS スレッドを増やさず大量の並行処理を書ける Virtual Threads が正式導入された。

## このバージョンの位置づけ（リリース / サポート・LTS / どこで使うか）
- リリース: 2023年9月。LTS（長期サポート）版。前 LTS は 17、次 LTS は 25（2025年9月）。
- LTS は数年単位でセキュリティ更新が提供されるため、業務システムは LTS を選ぶのが定番。21 は 17 の次の現役 LTS。
- 配布元: Eclipse Temurin（Adoptium）/ Amazon Corretto / Oracle JDK など。`java --version` で確認。
- 使う場所: エンタープライズバックエンド・マイクロサービス（Spring Boot）・ビッグデータ基盤（Spark/Kafka）。フレームワークは [../spring/](../spring/) を参照。
- 次LTS 25 への一言: 25 では `record` パターンや構造化並行（Structured Concurrency）がさらに整備される方向。21 で書いたコードは基本そのまま動く。

```bash
# クラスファイルのバージョンを 21 に固定してコンパイル
javac --release 21 App.java
java App
```

## 言語の基本（文法の要点）
すべてのコードはクラスの中に書く。エントリポイントは `public static void main(String[] args)`。

```java
public class App {
    public static void main(String[] args) {
        int count = 3;                 // プリミティブ型
        String name = "Java";          // 参照型
        var list = new java.util.ArrayList<String>(); // var で型推論（ローカル変数のみ）
        for (int i = 0; i < count; i++) {
            list.add(name + i);
        }
        System.out.println(list);      // [Java0, Java1, Java2]
    }
}
```

- 文末はセミコロン必須。ブロックは `{}`。
- アクセス修飾子: `public` / `protected` / `private` / （無指定=パッケージプライベート）。
- `var` は初期化必須のローカル変数限定で型推論。フィールドや引数には使えない。

## この言語の核心概念（他言語と違う・必ず押さえる）

### プリミティブ型 vs 参照型
- **何か**: `int` `long` `double` `boolean` `char` などのプリミティブは値そのものを変数に格納する。クラス・配列・インターフェースは参照型で、変数にはオブジェクトの参照（住所）が入る。各プリミティブには対応するラッパークラス（`Integer` `Long` `Boolean` 等）があり、コレクションには参照型しか入れられないため自動変換（オートボクシング）が走る。
- **具体コード**:

```java
int a = 100;               // 値そのもの
Integer b = 100;           // オートボクシング: int → Integer（参照型）
int c = b;                 // アンボクシング: Integer → int

List<Integer> list = new ArrayList<>();
list.add(5);               // int 5 が自動で Integer に箱詰めされる
int first = list.get(0);   // 取り出し時に自動で開封される
```

- **他言語と違う点/つまずき**: JS/Python は数値も「オブジェクト的」に統一されているが、Java は2系統がはっきり分かれる。`Integer` は `null` になり得るため、`null` の `Integer` をアンボクシングすると `NullPointerException`。オートボクシングはループ内で大量のオブジェクト生成を招き性能低下の原因にもなる。

### `==` は参照比較・`equals` で値比較
- **何か**: `==` は「同じオブジェクトか（参照が一致するか）」を比べる。中身が同じかは `equals()` で比べる。プリミティブ同士の `==` は値比較で問題ない。
- **具体コード**:

```java
String x = new String("hi");
String y = new String("hi");
System.out.println(x == y);       // false（別オブジェクト）
System.out.println(x.equals(y));  // true（中身が同じ）

Integer p = 127, q = 127;
System.out.println(p == q);       // true（-128〜127 はキャッシュされ同一インスタンス）
Integer r = 200, s = 200;
System.out.println(r == s);       // false（キャッシュ外は別インスタンス）
```

- **他言語と違う点/つまずき**: JS の `===` は値比較（プリミティブ）だが、Java の `==` はオブジェクトでは常に参照比較。文字列・ラッパーの比較を `==` で書く事故が定番。特に `Integer` のキャッシュ範囲（-128〜127）では `==` がたまたま通り、範囲外で突然 false になるため発見が遅れる。**オブジェクトの値比較は必ず `equals()`**。

### 値渡し（ただし「参照の値」を渡す）
- **何か**: Java の引数は常に値渡し（＝箱の中身をコピーして渡す）。プリミティブとオブジェクトで「箱に何が入っているか」が違うのが、分かりにくさの正体。

**大前提：変数は「箱」。箱に入るものが2種類ある**

プリミティブは、箱に**値そのもの**が入る:
```
int a = 10;
   a
 ┌──────┐
 │  10  │   ← 箱の中に 10 が直接入っている
 └──────┘
```
オブジェクトは、箱に**住所**が入る（本体は別の場所）:
```
User user = new User("alice");
   user                    メモリ上の 0x100 番地
 ┌────────┐              ┌──────────────────────┐
 │ 0x100  │ ───────────→ │ User{ name: "alice" }│
 └────────┘              └──────────────────────┘
 箱の中身は「住所 0x100」       本体はここにいる
```
👉 **`user` という箱は、オブジェクトを入れていない。住所を入れている。** これが全ての鍵。

**「値渡し」＝箱の中身をコピーして渡す。** `f(user)` を呼ぶと、`user` の箱の中身（住所 0x100）を引数の箱にコピーする → 2つの箱が同じ本体を指す。ここから2パターン:

- **パターン③ 中身を変更（伝わる）**: `u.name = "bob"` は住所 0x100 へ行って本体を書き換える。両方の箱が 0x100 を指すので、呼び出し元からも "bob" に見える。✓
- **パターン② 住所を再代入（伝わらない）**: `u = new User("bob")` は新しい本体を 0x200 に作り、引数の箱に 0x200 を入れるだけ。呼び出し元の箱は 0x100 のまま。✗

- **具体コード**:

```java
class User { String name; User(String n){ name = n; } }

static void rename(User u)  { u.name = "bob"; }          // ③ 中身を変更
static void replace(User u) { u = new User("bob"); }     // ② 住所を再代入

User user = new User("alice");
rename(user);
System.out.println(user.name);   // bob  … ③ 伝わる（同じ本体を書き換えた）

User user2 = new User("alice");
replace(user2);
System.out.println(user2.name);  // alice … ② 伝わらない（箱の住所を差し替えただけ）

// プリミティブは常に①：値そのものをコピー
static void inc(int x){ x = x + 1; }
int a = 10; inc(a);
System.out.println(a);           // 10 … 伝わらない
```

| 操作 | 呼び出し元に伝わる？ | なぜ |
|---|---|---|
| プリミティブの再代入 | ✗ | 箱の中の値をコピーしただけ |
| 引数（参照）を別オブジェクトに**再代入**（②） | ✗ | 箱の住所を差し替えただけ。元の箱は別の住所 |
| 引数の指すオブジェクトの**中身を変更**（③） | ✓ | 2つの箱が同じ本体を指す。その本体を書き換えた |

- **一言で**: Javaが渡すのは「本体」ではなく「**本体の住所のコピー**」。だから**中身の変更は伝わり、住所の差し替えは伝わらない**。これが「参照の値渡し」。`String` は不変なので中身を変えられず②しか起きない＝「変わらない」ように見えるのも、この延長。
- **他言語と違う点/つまずき**: 「Java は参照渡し」と誤解されがちだが正確には **参照の値渡し**。Go の構造体は逆に「住所でなく本体ごとコピー」されるので、Go で中身を変えたい時はポインタを渡す（→ [../../go/解説/go1.23.md](../../go/解説/go1.23.md) のポインタ）。
- 全11言語の比較（同じグループ／PHP・Go・Swift・Rust の独自）→ [../../../言語共通概念/値渡しと参照.md](../../../言語共通概念/値渡しと参照.md)

### `null` と NPE・`Optional`
- **何か**: 参照型は「何も指していない」状態 `null` を取り得る。`null` のメソッド呼び出しは `NullPointerException`（NPE）。値が無いかもしれないことを型で表すには `Optional<T>` を使う。
- **具体コード**:

```java
Map<String, Integer> m = new HashMap<>();
Integer v = m.get("missing");     // null（例外ではない）
// int n = m.get("missing");      // アンボクシングで NPE

Optional<Integer> found = Optional.ofNullable(m.get("missing"));
int n = found.orElse(0);          // 無ければ既定値（NPE を回避）
found.ifPresent(x -> System.out.println(x)); // あれば実行
```

- **他言語と違う点/つまずき**: Go のように「ゼロ値」は無く、初期化前の参照は `null`。NPE は最頻出の実行時例外。`Optional` は戻り値で「無いかもしれない」を表す用途であり、フィールドや引数に多用するのは非推奨。

### ジェネリクスの型消去
- **何か**: ジェネリクスはコンパイル時の型チェックのためだけにあり、**実行時には型パラメータの情報が消える**（型消去）。バイトコード上は `List<String>` も `List<Integer>` も同じ `List`。
- **具体コード**:

```java
List<String> ss = new ArrayList<>();
List<Integer> is = new ArrayList<>();
System.out.println(ss.getClass() == is.getClass()); // true（実行時は同じ List）

// if (ss instanceof List<String>) {} // コンパイルエラー: 実行時に型引数を確認できない
// T[] arr = new T[10];                // 不可: 実行時に T が分からない
```

- **他言語と違う点/つまずき**: C#（実行時も型を保持）とは異なり、Java は型を消す。そのため `instanceof List<String>` や `new T[]` が書けず、`Class<T>` を引数で受け取るなどの回避策が要る。「実行時にジェネリック型で分岐したい」場面で詰まる。

## 型・データモデル
- プリミティブ型（`int` `long` `double` `boolean` `char` 等）と参照型（クラス・配列・インターフェース）の二系統。
- プリミティブ/参照・オートボクシング・型消去の詳細は「この言語の核心概念」を参照。
- ジェネリクス（型パラメータ）でコレクションを型安全に扱う。

```java
import java.util.*;

Map<String, List<Integer>> scores = new HashMap<>();
scores.computeIfAbsent("alice", k -> new ArrayList<>()).add(90);

// ジェネリックメソッド
static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

## この言語らしさ / 特徴的な機能
### record（不変データの簡潔な定義）
フィールド・コンストラクタ・`equals`/`hashCode`/`toString` を自動生成する。

```java
public record Point(int x, int y) {}

var p = new Point(1, 2);
System.out.println(p.x());        // 1（アクセサは x()）
System.out.println(p);            // Point[x=1, y=2]
System.out.println(p.equals(new Point(1, 2))); // true
```

### sealed（継承を許可する型を限定）
`permits` で派生を限定し、網羅的な分岐を可能にする。

```java
public sealed interface Shape permits Circle, Rectangle {}
public record Circle(double r) implements Shape {}
public record Rectangle(double w, double h) implements Shape {}
```

### パターンマッチ（switch / instanceof）
Java 21 で switch のパターンマッチが正式機能になった。`sealed` と組むと全分岐の網羅をコンパイラが検査する。

```java
static double area(Shape s) {
    return switch (s) {
        case Circle c       -> Math.PI * c.r() * c.r();
        case Rectangle r    -> r.w() * r.h();
        // sealed なので default 不要（網羅チェックが効く）
    };
}

// instanceof パターン: キャストと変数束縛を同時に
static String describe(Object o) {
    if (o instanceof String str && !str.isBlank()) {
        return "text:" + str.length();
    }
    return "other";
}
```

### テキストブロック（複数行文字列）

```java
String json = """
        {
          "name": "java",
          "version": 21
        }
        """;
```

## 並行・非同期
従来はプラットフォームスレッド（= OS スレッド）が並行処理の単位だった。OS スレッドは重く、数千個立てるとメモリと切替コストで破綻する。

### Virtual Threads（21の目玉）
JVM が管理する超軽量スレッド。数百万個立てても OS スレッドを消費せず、ブロッキングI/O待ちの間に JVM が裏で OS スレッドを譲り合う。「1リクエスト1スレッド」の素直な同期コードのまま高並行を実現する。

```java
import java.util.concurrent.*;

// 1万個の仮想スレッドで並行実行
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        int id = i;
        executor.submit(() -> {
            Thread.sleep(java.time.Duration.ofSeconds(1)); // ブロッキングしてもOSスレッドを占有しない
            return id;
        });
    }
} // close で全タスク完了を待つ
```

```java
// 個別に起動する場合
Thread vt = Thread.ofVirtual().start(() -> System.out.println("on virtual thread"));
vt.join();
```

- ポイント: コードの書き方は従来のスレッドと同じ。`synchronized` の長時間保持や `ThreadLocal` の多用は仮想スレッドの利点を損なうので注意。
- 低レベルには `CompletableFuture`（非同期合成）も従来どおり使える。

## 標準ライブラリ / ツールチェーン
- ビルド: **Maven**（`mvn`）/ **Gradle**（`gradle`）。依存解決・テスト・パッケージングを担う。
- 主要 API: `java.util`（コレクション）、`java.util.stream`（Stream API）、`java.time`（日時）、`java.net.http`（HTTP クライアント）、`java.nio.file`（ファイル）。
- テスト: JUnit 5 / Mockito。
- 起動ツール: 単一ファイルなら `java App.java` で直接実行も可能（Source-File モード）。

```java
// Stream API（宣言的なデータ処理）
import java.util.*; import java.util.stream.*;
var nums = List.of(1, 2, 3, 4, 5, 6);
List<Integer> evensDoubled = nums.stream()
        .filter(n -> n % 2 == 0)
        .map(n -> n * 2)
        .collect(Collectors.toList()); // [4, 8, 12]
```

## このバージョンの新機能・トピック（17→21 の主な確定機能）
- Virtual Threads（JEP 444）正式化 — 上記の目玉。
- Pattern Matching for switch（JEP 441）正式化。
- Record Patterns（JEP 440）正式化 — record を分解束縛できる。

```java
record Point(int x, int y) {}
static String loc(Object o) {
    if (o instanceof Point(int x, int y)) {   // 分解パターン
        return x + "," + y;
    }
    return "?";
}
```
- Sequenced Collections（JEP 431） — 先頭/末尾アクセスを統一する `getFirst()` / `getLast()` / `reversed()` を追加。

```java
var list = new java.util.ArrayList<>(java.util.List.of("a", "b", "c"));
System.out.println(list.getFirst()); // a
System.out.println(list.getLast());  // c
System.out.println(list.reversed()); // [c, b, a]
```
- プレビュー: String Templates（JEP 430）、Structured Concurrency（JEP 453）、Unnamed Patterns `_` などは 21 時点でプレビュー（`--enable-preview` が必要）。次 LTS で安定化が進む。

## ハマりどころ
- `var` はローカル変数限定。フィールド・引数・戻り値には使えない。
- `==` は参照比較。文字列やラッパーの値比較は `equals()` を使う（`Integer` のキャッシュ範囲 -128〜127 で `==` がたまたま通り、範囲外で失敗する事故が典型）。
- ジェネリクスは型消去のため、実行時に `List<String>` と `List<Integer>` を区別できない。`instanceof List<String>` は書けない。
- 仮想スレッドで `synchronized` ブロック内の長時間ブロッキングはキャリアスレッドを固定（pinning）し利点を消す。`ReentrantLock` の利用を検討。
- プレビュー機能はコンパイル/実行両方に `--enable-preview --release 21` が必要。本番では安定機能のみ使う。

## 関連
- フレームワーク: [../spring/](../spring/)（Spring / Spring Boot）
- 言語概要: [../README.md](../README.md)
