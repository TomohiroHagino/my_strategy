# TypeScript 5.x（言語解説）

## ひとことで言うと
JavaScript に静的な型を載せ、コンパイル時に型チェックして JS へ変換する言語。構造的型付けとジェネリクス・union/交差/条件型/mapped types で表現力の高い型を書け、5.x では `const` 型パラメータ・`using`（Disposable）・標準デコレータなどが入った。

## このバージョンの位置づけ（リリース / サポート・LTS / どこで使うか）
- 5.0 は 2023年3月。以後 5.1 / 5.2 / 5.3 / 5.4 / 5.5 … と数か月ごとにマイナー更新。LTS の概念はなく、最新の安定版を追うのが基本。
- 型は実行時に消える（型消去）。`tsc` または esbuild/swc 等が型を落として JS を出力する。
- 使う場所: React/Vue/Next の型付きコンポーネント（→ [../react/](../react/)）、Node のサーバ実装（→ [../../バックエンド/node/](../../バックエンド/node/)）、ライブラリの型定義。土台の言語仕様は [../javascript/](../javascript/) を参照。

```bash
npm i -D typescript
npx tsc --init     # tsconfig.json を生成
npx tsc            # 型チェック＋JS出力
```

```jsonc
// tsconfig.json の要点
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "strict": true,            // 型安全の基本セットを有効化
    "noUncheckedIndexedAccess": true, // 配列/オブジェクト添字アクセスに undefined を含める
    "moduleResolution": "Bundler"
  }
}
```

## 言語の基本（文法の要点）
JS の構文に型注釈を足す。`: 型` を変数・引数・戻り値に付ける。

```ts
function greet(name: string, times: number = 1): string {
  return (name + " ").repeat(times).trim();
}

let count: number = 3;
const ids: number[] = [1, 2, 3];
const pair: [string, number] = ["age", 30]; // タプル
```

- `interface` と `type` でオブジェクトの形を定義する。

```ts
interface User {
  id: number;
  name: string;
  email?: string;       // オプショナル
  readonly createdAt: Date;
}

type Point = { x: number; y: number };
```

## この言語の核心概念（他言語と違う・必ず押さえる）

### 構造的型付け
- **何か**: 型の同一性を「名前」ではなく「形（プロパティの集合）」で判定する。必要な形さえ満たしていれば、`implements` のような明示宣言なしで代入できる。
- **具体コード**:

```ts
interface HasName { name: string }
function printName(o: HasName) { console.log(o.name); }

const dog = { name: "pochi", legs: 4 };
printName(dog); // HasName を明示実装していなくても、形が合うので通る

class Cat { constructor(public name: string) {} }
const c: HasName = new Cat("tama"); // クラスの継承関係も無関係に代入可
```

- **他言語と違う点/つまずき**: Java/C# などの公称型（名前で一致を判定）と正反対。「別物のつもりの型がたまたま同じ形で代入できてしまう」ことがある。逆に、余分なプロパティを持つオブジェクトも通る（過剰プロパティチェックはオブジェクトリテラルを直接代入した時だけ働く）。

### 型は実行時に消える
- **何か**: 型注釈・`interface`・`type` はコンパイル時のチェックにだけ使われ、出力された JS からは完全に消える。実行時に型で分岐したり検証したりはできない。
- **具体コード**:

```ts
interface User { id: number; name: string }

// NG: 実行時に User は存在しないので検証にならない
const data = (await res.json()) as User; // as は「コンパイラを黙らせる」だけ

// 推奨: zod など実行時バリデーションで型を“作る”
import { z } from "zod";
const UserSchema = z.object({ id: z.number(), name: z.string() });
const user = UserSchema.parse(await res.json()); // 形が違えば例外
```

- **他言語と違う点/つまずき**: Java のように実行時に型情報が残らない（C# のジェネリクスとも違う）。`if (x instanceof User)` は interface には使えない。外部入力（API レスポンス・フォーム）を `as` で型付けしても **無検証** なので、必ず zod 等のランタイム検証を挟む。

### union 型と絞り込み（narrowing）／型ガード
- **何か**: 複数の型のどれか、を `A | B` で表す。`typeof`・`in`・等値比較・カスタム型ガードで、条件分岐の中で具体的な型に絞り込む（narrowing）と、コンパイラがその型として扱ってくれる。
- **具体コード**:

```ts
type Status = "idle" | "loading" | "error"; // リテラルユニオン

function render(s: Status | null) {
  if (s === null) return "none"; // ここで s は null に絞られる
  switch (s) {
    case "loading": return "...";
    default: return s;           // 残り "idle" | "error" に絞られる
  }
}

// typeof / in / カスタム型ガード
function len(v: string | string[]) {
  return typeof v === "string" ? v.length : v.length;
}
type Fish = { swim: () => void };
function isFish(x: unknown): x is Fish {            // 戻り値 `x is Fish` がガード
  return typeof x === "object" && x !== null && "swim" in x;
}
```

- **他言語と違う点/つまずき**: 多くの言語の列挙型と違い、union は「値の集合」を型として直接書ける。絞り込みはフローに依存するため、絞った後に別関数へ渡したりクロージャに入れると narrowing が解けて再びユニオンに戻る、という落とし穴がある。

### `any` vs `unknown` vs `never`
- **何か**: `any` は型チェックを完全に無効化する最終手段。`unknown` は「何でも入るが、使う前に必ず絞り込みが要る安全な any」。`never` は「起こり得ない値」を表し、網羅性チェックなどに使う。
- **具体コード**:

```ts
let a: any = "x";    a.foo.bar;        // 何でも通る（型安全が崩れる）
let u: unknown = "x";
// u.length;                            // エラー: 絞り込み前は使えない
if (typeof u === "string") u.length;    // 絞れば使える

function assertNever(x: never): never { throw new Error("unexpected " + x); }
function area(s: "circle" | "square") {
  switch (s) {
    case "circle": return 1;
    case "square": return 2;
    default: return assertNever(s); // ケース追加忘れをコンパイル時に検出
  }
}
```

- **他言語と違う点/つまずき**: `any` を一度使うとそこから先の型安全が連鎖的に崩れる。外部入力は `any` ではなく `unknown` で受けて絞り込むのが定石。`never` は「ここには来ないはず」を型で保証する独特の道具で、他言語に直接の対応物が少ない。

### 型と値で宣言空間が別
- **何か**: TypeScript には「型の名前空間」と「値の名前空間」が独立して存在する。同じ名前を型と値の両方に持てる。`as const` や `satisfies` は値側に型レベルの意味を与えるための道具。
- **具体コード**:

```ts
// 同名でも型と値は別空間に共存できる
type Color = "red" | "blue";
const Color = { red: "red", blue: "blue" } as const;

const palette = { primary: "red", size: 12 } as const; // 値をリテラル型に固定
type Primary = typeof palette["primary"]; // "red"（値→型）

const cfg = { port: 3000, host: "localhost" }
  satisfies Record<string, string | number>; // 形だけ検査し、型は広げない
cfg.port; // number のまま
```

- **他言語と違う点/つまずき**: 「型を import したつもりが値も import していた（あるいは逆）」が起きる。`import type` で型だけ取り込める。`as const` でリテラルを固定し、`satisfies` で「型を保ったまま制約だけ検査」できる点は他言語の型注釈とは別物の発想。

## 型・データモデル
### ジェネリクス・条件型・mapped types

```ts
// ジェネリクス
function first<T>(arr: readonly T[]): T | undefined {
  return arr[0];
}

// 条件型（型レベルの if）
type ElementType<T> = T extends (infer U)[] ? U : T;
type A = ElementType<string[]>; // string

// mapped types（全プロパティを変換）
type Optional<T> = { [K in keyof T]?: T[K] };
type PartialUser = Optional<User>;

// ユーティリティ型（標準提供）
type UserPreview = Pick<User, "id" | "name">;
type WithoutId = Omit<User, "id">;
```

（`any`/`unknown`/`never`・実行時に型が消える点・`as const`/`satisfies` は「この言語の核心概念」を参照）

## この言語らしさ / 特徴的な機能
### 型推論
明示しなくても多くは推論される。書きすぎないのがコツ。

```ts
const nums = [1, 2, 3];               // number[]
const doubled = nums.map((n) => n * 2); // number[]（コールバック引数も推論）
```

## 並行・非同期
非同期は土台の JS と同じく `Promise` / `async`・`await`。TS では Promise の解決値に型が付く。

```ts
async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`status ${res.status}`);
  return UserSchema.parse(await res.json());
}

// 並行実行
const [a, b] = await Promise.all([fetchUser(1), fetchUser(2)]);
```

- 詳しいイベントループ/非同期の仕組みは [../javascript/](../javascript/) を参照。TS の役割は「`await` した値の型を正しく追跡する」こと。

## 標準ライブラリ / ツールチェーン
- コンパイラ: `tsc`（型チェックと JS 出力）。バンドラ（Vite/esbuild/swc）は高速に型を落とすが型チェックはしないため、CI で `tsc --noEmit` を併用するのが定番。
- 設定: `tsconfig.json`。`strict` を基本 ON にする。
- 実行時検証: **zod** 等（型と検証を一本化）。
- 型定義の配布: `@types/*` パッケージ、またはライブラリ同梱の `.d.ts`。

```bash
npx tsc --noEmit          # 出力せず型チェックのみ（CI 向け）
```

## このバージョンの新機能・トピック
### const 型パラメータ（5.0）
ジェネリックの推論を「広げず」リテラルのまま取る。

```ts
function asTuple<const T extends readonly unknown[]>(t: T): T {
  return t;
}
const t = asTuple(["a", "b"]); // 型: readonly ["a", "b"]（string[] に広がらない）
```

### Decorators 標準仕様（5.0）
ECMAScript 提案準拠のデコレータが正式サポート（旧 `experimentalDecorators` とは別物）。

```ts
function logged<This, Args extends any[], Return>(
  target: (this: This, ...args: Args) => Return,
  ctx: ClassMethodDecoratorContext
) {
  return function (this: This, ...args: Args): Return {
    console.log(`call ${String(ctx.name)}`);
    return target.call(this, ...args);
  };
}

class Service {
  @logged
  run(x: number) { return x * 2; }
}
```

### using / Disposable（5.2）
`using` 宣言でスコープを抜けるとき自動でリソース解放（`Symbol.dispose`）を呼ぶ。

```ts
class FileHandle {
  [Symbol.dispose]() { console.log("closed"); }
  read() { return "data"; }
}

function readFile() {
  using f = new FileHandle(); // ブロックを抜けると自動で dispose
  return f.read();
}                              // ここで "closed" が出る
```

- 非同期版は `await using`（`Symbol.asyncDispose`）。
- その他: `--moduleResolution bundler`（5.0）、`NoInfer<T>` ユーティリティ型・正規表現構文チェック（5.4/5.5）、JSDoc からの型推論強化など、版ごとに型周りが継続強化されている。

## ハマりどころ
- `as` は実行時チェックではない。外部入力には必ずバリデーションを挟む。
- `any` を一度使うとそこから先の型安全が崩れる。`unknown` ＋ 絞り込みを使う。
- 構造的型付けゆえ、余計なプロパティを持つオブジェクトも通ってしまう（過剰プロパティチェックはオブジェクトリテラル直接代入時のみ働く）。
- `enum` は実行時にコードを生成し挙動が独特。リテラルユニオン or `as const` オブジェクトで代替するチームが多い。
- 型エラーが消えても実行時バグは別問題。型は「形」を保証するだけで「値の中身」までは保証しない。

> 各現象の「なぜ起きる／どう直す」を網羅した辞書 → **[../落とし穴.md](../落とし穴.md)**（JS共通の罠は [../../javascript/落とし穴.md](../../javascript/落とし穴.md)）。

## 関連
- 土台の言語: [../javascript/](../javascript/)
- UI: [../react/](../react/)
- サーバ: [../../バックエンド/node/](../../バックエンド/node/)
- 言語概要: [../README.md](../README.md)
