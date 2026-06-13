# Rust 1.85 / edition 2024（言語解説）

## ひとことで言うと
所有権・借用・ライフタイムでメモリ安全をガベージコレクションなしに保証する静的型付け言語。Rust 1.85 は新しい安定エディション「edition 2024」を提供したリリースで、`async` クロージャや一部の構文ルール変更が入った。フロントエンド領域では Rust → WebAssembly(WASM) にコンパイルしてブラウザで動かす用途で使われる。

## このバージョンの位置づけ（リリース / サポート / どこで使うか）
- Rust 1.85（2025 年初頭）で edition 2024 が安定化した。エディションはソースの互換単位で、新エディションは新しい構文・既定を選べる一方、古いエディションのクレートとも混在できる。
- エディションは `Cargo.toml` の `edition = "2024"` で指定する。2015 / 2018 / 2021 / 2024 が選べ、`cargo fix --edition` で移行を半自動化できる。
- フロントエンドでの用途は WASM。`web-sys` / `wasm-bindgen` 経由で DOM を操作し、Leptos / Yew / Dioxus などのフレームワーク上で UI を書く。重い計算や型安全性が要る箇所で JS/TS を補完する位置づけ。

```bash
rustc --version
cargo new my_app          # プロジェクト雛形（Cargo.toml に edition が入る）
cargo build               # ビルド
cargo run                 # 実行
```

## 言語の基本（文法の要点）
変数は既定で不変。`mut` を付けると可変になる。型推論が効く。

```rust
let name = "Rust";        // 不変
let mut count = 0;        // 可変
count += 1;
let pi: f64 = 3.14;       // 明示型
```

関数は `fn`。最後の式が戻り値になる（`return` は早期脱出用）。

```rust
fn greet(name: &str) -> String {
    format!("Hello, {name}!")   // 末尾の式が戻り値
}
```

`if` / `match` / `loop` / `while` / `for` が制御構文。`if` と `match` は式（値を返す）。

```rust
let grade = match score {
    90..=100 => "A",
    70..=89 => "B",
    _ => "C",            // _ はワイルドカード
};
```

## この言語の核心概念（他言語と違う・必ず押さえる）
Rust の全ては「所有権・借用・ライフタイム」から派生する。文法の前に、この6点を押さえる。

### 所有権（ownership）／ ムーブ
①何か：各値にはちょうど 1 つの所有者があり、所有者がスコープを抜けると値は解放される。代入・関数渡しは（Copy でない型では）所有権の移動＝ムーブで、移動元の変数はもう使えなくなる。これが Rust 最大の核心。

②具体コード：
```rust
let s = String::from("hi");
let t = s;            // 所有権が t にムーブ
// println!("{s}");   // ← s はもう使えずコンパイルエラー
println!("{t}");

fn take(v: String) { /* v を消費 */ }
let a = String::from("x");
take(a);              // a の所有権が関数へ移動
// println!("{a}");   // ← ここでもエラー
```

③他言語と違う点/つまずき：GC 言語では代入は参照のコピーで、両方の変数が同じオブジェクトを指し続ける。Rust は「使えなくなった変数」をコンパイラが追跡してエラーにする。`i32` などの `Copy` 型はムーブでなく複製されるので動く一方、`String`/`Vec` はムーブする、という線引きが最初の壁。必要なら `.clone()` で明示的に複製する。

### 借用（`&` / `&mut`）と借用規則
①何か：所有権を移さずに値を参照するのが借用。共有参照 `&T`（複数同時に持てる・読み取り専用）と可変参照 `&mut T`（同時に 1 つだけ）がある。「ある時点で、可変参照は 1 つだけ、または不変参照は複数」という規則をコンパイラが強制する。

②具体コード：
```rust
fn len(s: &String) -> usize { s.len() }   // 借用、所有権は呼び出し元に残る

let mut v = vec![1, 2, 3];
let r1 = &v;            // 不変借用（複数可）
let r2 = &v;
println!("{r1:?} {r2:?}");
let m = &mut v;         // 可変借用は同時に 1 つだけ
m.push(4);
```

③他言語と違う点/つまずき：他言語は「誰でもいつでも読み書きできる参照」が普通で、データ競合や反復中の変更は実行時に壊れる。Rust は「可変1つ or 不変複数」を型レベルで保証し、データ競合をコンパイル時に防ぐ。不変借用が生きている間に可変借用を作るとエラーになる。慣れるまでこのエラーに戸惑うが、これが安全性の源泉。

### ライフタイム（参照の生存期間 `'a`）
①何か：参照が指す先がいつまで有効かをコンパイラに伝える注釈。多くは推論されるが、関数が参照を返す場面など、複数の参照の生存関係を結びつける必要があるときに `'a` を明示する。

②具体コード：
```rust
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }  // 戻り値は a,b と同じ寿命
}
```

③他言語と違う点/つまずき：GC 言語には存在しない概念。`'a` は実行時に何かを増やすのではなく、純粋に「ダングリング参照（解放済みを指す参照）を作らせない」ための静的チェック。参照を返す関数で `'a` を要求されたとき、多くは「所有する値（`String`）を返す」設計に直すと注釈が消える。

### GC なし・`Drop`（スコープ終了で解放）
①何か：ガベージコレクタがなく、値はスコープを抜けた時点で確定的に解放される。解放時の後始末は `Drop` トレイトで書ける。所有権モデルがあるので「いつ解放されるか」がコードから読める。

②具体コード：
```rust
struct Guard(&'static str);
impl Drop for Guard {
    fn drop(&mut self) { println!("release {}", self.0); }
}
fn main() {
    let _g = Guard("file");
    println!("working");
}   // ここで _g がスコープを抜け、自動で drop（"release file"）
```

③他言語と違う点/つまずき：GC 言語では解放タイミングが非決定的（いつ回収されるか不明）。Rust はスコープ終了で即解放され、`Drop` が RAII（資源確保＝初期化、解放＝破棄）として働く。`finally` や `defer` を書かなくてもファイルやロックが確実に閉じる。逆に、循環参照を `Rc` で作ると参照カウントが 0 にならず解放されない（そこは `Weak` で切る）。

### `Result` / `Option` と `?`、`match`（網羅）
①何か：エラーは例外でなく型で表す。失敗しうる処理は `Result<T, E>`、値の有無は `Option<T>` を返す。`?` 演算子は Err/None なら呼び出し元へ早期 return し、`match` はパターンで全バリアントを網羅する（漏れはコンパイルエラー）。

②具体コード：
```rust
fn parse(s: &str) -> Result<i32, std::num::ParseIntError> {
    let n: i32 = s.parse()?;   // 失敗なら早期 return で Err
    Ok(n * 2)
}
let v = vec![1, 2, 3];
match v.first() {              // Option<&i32>
    Some(x) => println!("{x}"),
    None => println!("empty"),
    // 全バリアントを書く。漏れるとコンパイルエラー
}
```

③他言語と違う点/つまずき：try/catch の例外と違い、失敗が関数の戻り値の型に現れるので「どの関数が失敗しうるか」が型で分かる。`?` は例外の伝播に似るが、あくまで `Result`/`Option` を返す関数の中でしか使えない。`.unwrap()` は中身を強制的に取り出すが None/Err でパニックするので、本番コードでは `?`・`match`・`unwrap_or` で扱う。

### trait と所有権、`unsafe` の位置づけ
①何か：`trait` は共有する振る舞いの定義で、型に対して `impl` し、ジェネリクスの境界にも使う。所有権モデルと結びつき、`Send`/`Sync` のようなマーカートレイトがスレッド安全性を型で表す。どうしても所有権・借用規則の外に出たい箇所だけ `unsafe` ブロックで囲む。

②具体コード：
```rust
trait Area { fn area(&self) -> f64; }
struct Sq(f64);
impl Area for Sq { fn area(&self) -> f64 { self.0 * self.0 } }

fn print_area(x: &impl Area) { println!("{}", x.area()); }

let raw = &10 as *const i32;
let val = unsafe { *raw };   // 生ポインタの参照外しは unsafe 内だけ
```

③他言語と違う点/つまずき：Java のインターフェースに似るが、`trait` は所有権・ライフタイム・`Send`/`Sync` と一体で機能し、コンパイラがスレッド間の受け渡し安全性まで型で検査する（データ競合をコンパイル時に弾く）。`unsafe` は「安全性チェックを無効化」ではなく「ここの正しさは人間が保証する」という限定的な宣言で、Rust の大部分は safe のまま書ける。`unsafe` を全体に広げるのは誤用。

## 型・データモデル
- 基本型は整数（`i32` / `u64` 等）、浮動小数（`f64`）、`bool`、`char`、文字列（所有する `String` と借用 `&str`）。複合型に tuple・配列・`Vec<T>`・`HashMap<K, V>`。
- struct と enum でデータを表す。enum はバリアントごとに値を持てる。

```rust
struct Point { x: i32, y: i32 }

enum Shape {
    Circle(f64),
    Rect { w: f64, h: f64 },   // 名前付きフィールドも可
}
```

エラーは例外でなく型で表す。`Result<T, E>`（成功/失敗）と `Option<T>`（あり/なし）。`?` 演算子でエラーを呼び出し元へ伝播する。

```rust
fn parse(s: &str) -> Result<i32, std::num::ParseIntError> {
    let n: i32 = s.parse()?;   // 失敗なら早期 return で Err を返す
    Ok(n * 2)
}

let first = vec![1, 2, 3].first().copied();  // Option<i32>
let v = first.unwrap_or(0);                  // None なら 0
```

`match` はパターンで網羅する。enum の全バリアントを書かないとコンパイルエラーになる。

```rust
fn area(s: &Shape) -> f64 {
    match s {
        Shape::Circle(r) => std::f64::consts::PI * r * r,
        Shape::Rect { w, h } => w * h,
        // 全バリアントを書けば _ 不要、漏れはコンパイルエラー
    }
}
```

## この言語らしさ / 特徴的な機能
所有権が核。各値には所有者が 1 つだけあり、所有者がスコープを抜けると値は解放される。GC がないのに解放漏れも二重解放も起きない。

```rust
let s = String::from("hi");
let t = s;          // 所有権が t に移動（ムーブ）
// println!("{s}");  // ← s はもう使えずコンパイルエラー
println!("{t}");
```

借用は所有権を移さず参照する。共有参照 `&T`（複数可・読み取り）と可変参照 `&mut T`（同時に 1 つだけ）。この規則がデータ競合を防ぐ。

```rust
fn len(s: &String) -> usize { s.len() }   // 借用、所有権は呼び出し元に残る

let mut v = vec![1, 2, 3];
let r = &mut v;       // 可変借用は同時に 1 つ
r.push(4);
```

ライフタイムは参照が有効な期間をコンパイラに伝える注釈。多くは推論されるが、関数が参照を返す場面で明示が要ることがある。

```rust
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }  // 'a は a,b と戻り値の関係を表す
}
```

trait は共有する振る舞いの定義。型に対して実装し、ジェネリクスの境界にも使う。

```rust
trait Area { fn area(&self) -> f64; }
impl Area for Point {
    fn area(&self) -> f64 { 0.0 }
}

fn print_area(x: &impl Area) {   // Area を実装した任意の型
    println!("{}", x.area());
}
```

## 並行・非同期
非同期は `async` / `.await`。`async fn` は即実行されず Future を返し、実行にはランタイム（標準ライブラリには含まれず tokio / async-std などを使う）が要る。

```rust
async fn fetch() -> String {
    // 実際の I/O はランタイム上で待つ
    "data".to_string()
}

async fn run() {
    let result = fetch().await;   // .await で完了を待つ
    println!("{result}");
}
```

並行と所有権が結びつくのが Rust の特徴。`Send`（スレッド間で渡せる）/ `Sync`（共有参照を渡せる）というマーカ trait をコンパイラが追跡し、データ競合になるコードをコンパイル時に弾く。共有可変状態は `Arc<Mutex<T>>` のように所有権と排他を型で包む。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];
for _ in 0..4 {
    let c = Arc::clone(&counter);   // 参照カウントを共有
    handles.push(thread::spawn(move || {
        *c.lock().unwrap() += 1;    // ロックして変更
    }));
}
for h in handles { h.join().unwrap(); }
```

## 標準ライブラリ / ツールチェーン
- 標準ライブラリに `Vec` / `HashMap` / `String`、`Option` / `Result`、イテレータ（`map` / `filter` / `collect`）、スレッド・同期プリミティブ、`Box`（ヒープ確保）/ `Rc` / `Arc`（参照カウント）がある。非同期ランタイムや HTTP は外部クレート。
- パッケージ管理とビルドは `cargo`。`Cargo.toml` に依存を書き、レジストリは crates.io。
- `rustfmt`（整形）、`clippy`（lint）、`cargo test`（テスト）、`cargo doc`（ドキュメント生成）。WASM 向けには `wasm-pack` や `trunk` を併用する。

```toml
# Cargo.toml（抜粋）
[package]
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }
```

## このバージョンの新機能・トピック
- edition 2024 の安定化（1.85 の主題）。エディションは破壊的変更を局所化する仕組みで、新規則を選びつつ旧クレートと共存できる。
- async クロージャ：クロージャが async ブロックを返せるようになり、非同期の高階関数が書きやすくなった。

```rust
let f = async |x: i32| x + 1;   // edition 2024 で安定
```

- `let ... else` や `if let` の一時値の生存ルール調整、`gen` などの予約、`unsafe` 周りの扱いの厳格化（一部の操作で `unsafe` が必須化）といった、エディションに紐づくルール変更が入っている。移行は `cargo fix --edition` が助けになる。
- 周辺として WASM ターゲット（`wasm32-unknown-unknown`）のツールが成熟しつつあり、フロントエンド用途の実用度が上がっている。ただしフレームワークは依然 0.x で API 変化が速い。

## ハマりどころ
- ムーブで「使えなくなった値」：所有権が移った変数を再利用しようとしてコンパイルエラー。必要なら `.clone()` するか参照を借りる。
- 借用規則：可変借用は同時に 1 つ、共有借用と可変借用は同居できない。エラーはコンパイル時に出るが慣れるまで戸惑う。
- `.unwrap()` の多用：`Option` / `Result` を `unwrap` で開くと None / Err でパニック。本番コードでは `?` や `match`、`unwrap_or` で扱う。
- ライフタイム注釈：参照を返す関数で `'a` を求められることがある。多くは設計を見直す（所有する値を返す）と消える。
- async ランタイム：`async fn` を呼んでも `.await` しなければ何も動かない。実行にはランタイムのセットアップが要る。
- WASM のサイズと相互運用：バイナリが大きくなりがちで、JS とのデータ受け渡しは `wasm-bindgen` 経由のコストがある。

## 関連
- [../leptos/](../leptos/) … 細粒度リアクティブ・SSR 対応の Rust フロントエンドフレームワーク Leptos の版別リファレンス。
- 同フォルダの [README.md](../README.md) … Rust フロントエンド / WASM 領域の概要・主要フレームワーク比較・強み弱み。
- 実務の主流は [../../javascript/](../../javascript/) / [../../typescript/](../../typescript/)（README の関連リンク参照）。
