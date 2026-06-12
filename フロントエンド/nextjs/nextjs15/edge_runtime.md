# Edge Runtime（エッジサーバー）（Next.js 15）

## ひとことで言うと
**ユーザーに地理的に近い世界各地の「エッジ」で動く、軽量なJavaScriptランタイム**。Node.js ではなく Web 標準API（`fetch`/`Request`/`Response`/Web Crypto 等）ベースで、起動が速く低レイテンシ。ただし **Node.js のAPI（`fs` 等）やネイティブ依存は使えない**。Next.js では middleware が常にこれで動き、Route Handler / ページも `export const runtime = "edge"` で選べる。

## 役割・なぜ必要か
- 通常の「Node.js サーバ（1リージョンに1拠点）」だと、遠いユーザーほど往復が遅い。**エッジは世界中の拠点で実行**するので、近くて速い。
- 起動が軽い（V8 isolate ベース）。認証チェック・リダイレクト・地域別出し分け・A/B など「**全リクエストの手前で軽く判定**」する処理に最適。
- ただし**実行環境が制限される**（フルNodeではない）ので、何でも置けるわけではない。

## Edgeサーバーは具体的に何をしている？

Edgeは、リクエストが**アプリ本体に届く前**に、**ユーザーの近く**で**軽い処理**を済ませる層。主にこういう仕事をする：

- **認証チェック**：ログイン状態（Cookie/トークン）を見て、未ログインならログインページへリダイレクト
- **リダイレクト / URL書き換え（rewrite）**：古いURLを新URLへ飛ばす、内部的に別パスへ繋ぐ
- **地域・言語の出し分け**：アクセス元の国・地域を見て、言語・通貨・表示を切り替える
- **A/Bテストの振り分け**：ユーザーをパターンAかBに割り当てる
- **アクセス制御**：Bot判定、簡単なレート制限、IP制限
- **軽量API**：外部APIへ問い合わせて、結果を整形して返すだけ、のような軽い処理

### 流れ（イメージ）
```
ユーザー
  ↓（地理的に近いEdge拠点に届く）
Edge：軽い判定（ログイン済み？地域は？AかBか？）
  ├─ ここで完結 → そのまま返す（速い）
  └─ 本体が必要 → アプリ本体（Node・1リージョン）へ渡す
```

ポイント：Edgeは **「重い仕事はしない」**。DB直結・重い計算・Node資産が要る処理は、後ろの**アプリ本体（Node）**の担当。Edgeは「手前で軽くさばく」役に徹する。

## 2つのランタイム（Edge vs Node.js）

| | Edge Runtime | Node.js Runtime（既定） |
|---|---|---|
| 実行場所 | 世界各地のエッジ | 1リージョンのサーバ |
| ベースAPI | Web標準（fetch / Request / Response / Web Crypto） | フルNode.js（fs, net, Buffer …） |
| 起動 | 速い（軽量isolate） | 比較的重い |
| npm/ネイティブ依存 | 制限あり（多くのNodeライブラリ不可） | 自由 |
| 向く用途 | 認証 / リダイレクト / 地域出し分け / 軽いAPI | DB直結 / 重い処理 / Node資産 |

## ローカル開発（npm run dev）と本番のEdgeは別物

混同しやすいが、`npm run dev` のサーバと「Edgeサーバー」は別物。

### まず2つだけ押さえる
- **`npm run dev`** … あなたのPCで**開発中に自分だけが確認するための仮サーバー**。本番でもEdgeでもない。
- **本番サーバー** … デプロイ後に動く本物。**Node（既定・万能）** と **Edge（速いが制限あり）** の2種類があり、ファイルごとに選ぶ。

### ローカルサーバの中身（Node部分とEdge"真似"部分が混在）
ローカルの dev サーバ自体は **あなたのPCの Node**。その中で：
- **何も指定していない部分** → ふつうに **Node** で動く
- **`runtime = "edge"` を付けた部分（と middleware）** → dev サーバが **「Edgeのフリ」をして動かす**（できることを制限する）

つまり1つのローカルサーバの中に、**Nodeで動く部分**と**Edgeを真似る部分**が混在する。

> ローカルでも"Edgeのフリ"をするのは、「Edge では `fs` が使えない」等のエラーを**手元で先に出す**ため。動いても実体は「あなたのPCのNode」。

### "場所"（世界中への分散）は真似しない
真似しているのは **使えるAPIの制限だけ**。**世界中の拠点で動く**ことは再現せず、ローカルはPC1台のまま。地理的な分散はデプロイ後に初めて起きる。

| 場面 | 何がどこで動くか |
|---|---|
| `npm run dev` | あなたのPC1台。Node部分はNode、Edge指定部分はEdgeのフリ |
| `npm run build` → `npm start` | 同じくPC1台で本番ビルドを動かす（本物のEdge分散ではない） |
| デプロイ（Vercel等） | Node部分→1リージョン／Edge部分→世界中の拠点。"Edgeサーバー"が実在するのはここ |

> **`runtime = "edge"` を自分で書かない限り、Edgeの"真似"部分は出てこない＝全部ふつうのNode。** 最初はEdgeを気にしなくてよい。

## Edgeで書けるもの / 書けないもの

Edge は **「Web標準のJavaScript」しか動かない**。**ブラウザで動くようなコードはOK、Node.js専用のものはNG**、と考えると分かりやすい。

### ✅ 書ける（Web標準API）
- `fetch` / `Request` / `Response` / `Headers`（HTTP通信）
- `URL` / `URLSearchParams`
- `crypto`（Web Crypto：`crypto.subtle`、`crypto.randomUUID()`）
- `TextEncoder` / `TextDecoder`、`atob` / `btoa`
- `ReadableStream` など（ストリーミング）
- `JSON`・配列・`Map`/`Set` などふつうのJS
- `process.env`（環境変数の**読み取り**はOK）
- Next.js の `NextRequest` / `NextResponse`、`req.cookies` / `req.headers` / geo情報

### ❌ 書けない（Node.js専用）
- `fs`（ファイル読み書き）・`path`・`os`・`child_process`・`net` などNodeモジュール
- `Buffer`（代わりに `Uint8Array` / `TextEncoder` を使う）
- `process` の大半（`process.env` 以外。`process.cwd()` 等は不可）
- **TCP接続のDBドライバ**（`pg`・`mysql2` の生TCP接続など）→ 動かない
- ネイティブ依存のnpm（bcrypt のネイティブ版 等）
- `eval` / `new Function`（動的なコード実行はセキュリティで禁止）
- 重い処理・大きな依存（CPU時間・コードサイズに上限）

### よくある壁：DB接続
Edge は TCP で直接 DB に繋げないことが多い。対策は2つ：
- **HTTP/サーバレス対応のドライバ**を使う（Neon / PlanetScale のサーバレスドライバ、Turso、Upstash Redis、Prisma Accelerate など。HTTP経由で叩ける）
- それが無理なら、その処理は **Node Runtime（既定）** のままにする（`runtime` を指定しない）

### 迷ったら
- ブラウザでも動きそうなコード（`fetch`で通信・文字列処理・軽い判定）→ **Edgeでも書ける**
- ファイル・OS・生のDB接続・重い計算・Node資産が要る → **Node（既定）に置く**

## 基本の書き方（コード）

### middleware は常に Edge で動く
```ts
// middleware.ts（プロジェクト直下）— 必ず Edge Runtime
import { NextRequest, NextResponse } from "next/server";

export function middleware(req: NextRequest) {
  // 例：未ログインならリダイレクト（全リクエストの手前で軽く判定）
  if (!req.cookies.get("token")) {
    return NextResponse.redirect(new URL("/login", req.url));
  }
  return NextResponse.next();
}
export const config = { matcher: ["/dashboard/:path*"] };
```

### Route Handler / ページを Edge にする
```ts
// app/api/hello/route.ts
export const runtime = "edge";   // ← これで Edge 実行（既定は "nodejs"）

export async function GET() {
  return Response.json({ hello: "from the edge" });
}
```
```tsx
// app/page.tsx — ページ単位でも指定できる
export const runtime = "edge";
```

## 実務での使い方・定番パターン
- **認証・リダイレクト・i18n・A/B**：middleware（Edge）で全リクエストの手前に判定。
- **地域別の出し分け**：リクエストの地理情報（国/都市）で内容を変える（CDN/ホスティング側の geo 情報を利用）。
- **低レイテンシな軽量API**：外部APIへ `fetch` して整形して返すだけ、のようなものは Edge 向き。
- **ストリーミング**：Edge はストリーミング応答と相性が良い。
- 逆に **DB直結・重い計算・Node資産が要る処理は Node.js Runtime（既定）**のままにする。

## ハマりどころ / アンチパターン
- **Node.js API を使ってしまう**：`fs`・`path`・`Buffer`・多くのnpmパッケージは Edge で動かない（ビルド/実行エラー）。Edge は Web API のみ。
- **重いライブラリを Edge に載せる**：Edge はコードサイズ上限が小さめ。大きい依存は Node 側へ。
- **DBコネクション**：従来のTCP接続ドライバは Edge 不可なことが多い。**HTTP/サーバレス対応ドライバ**（Neon / PlanetScale のサーバレスドライバ、Prisma Accelerate 等）を使う。
- **middleware に重い処理を入れる**：全リクエストで走るので、重いと全体が遅くなる。判定だけ軽く。
- **「Edgeにすれば速い」と全部Edge化**：Node資産やDB直結が要る所までEdgeにすると逆に苦労する。**手前の軽い判定だけEdge**が基本。

## 関連
[middleware.md](./middleware.md) / [route_handlers.md](./route_handlers.md) / [rendering.md](./rendering.md) / [deployment.md](./deployment.md)
