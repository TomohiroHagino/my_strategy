# ミドルウェア（Middleware）（Next.js 15）

## ひとことで言うと
プロジェクト直下の **`middleware.ts`** に書く、**リクエストがページ/APIに届く前に割り込む関数**。認証ガード・リダイレクト・URL書き換え・A/Bテストなどを「ルートに入る前」に一括で処理する。

## 役割・なぜ必要か
- 各 `page.tsx` / `route.ts` で個別に「未ログインなら弾く」を書くと散らかる。**入口で一括処理**すれば重複が消える。
- リクエストとレスポンスの**間に立つ層**。ここで `NextResponse` を返すことで、通す / リダイレクト / 書き換え / ヘッダ付与 を決められる。
- **Edge ランタイム**（ユーザーに近い場所）で動くため軽量で速い。Cookie を見てログイン状態を判定する、地域でコンテンツを出し分ける、といった用途に向く。
- 典型ユースケース：**認証ガード**（保護ルートを守る）、**リダイレクト**（旧URL→新URL）、**rewrite**（URLは変えず内部的に別パスを表示）、**A/B テスト**（Cookie で出し分け）。

## 基本の書き方（コード）
`middleware.ts` はプロジェクトのルート（`app/` と同階層、`src/` 構成なら `src/` 直下）に置く。**1ファイルだけ**。

```tsx
// middleware.ts
import { NextRequest, NextResponse } from "next/server";

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const token = request.cookies.get("session")?.value;

  // 認証ガード：未ログインで /dashboard 配下なら /login へ
  if (pathname.startsWith("/dashboard") && !token) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("from", pathname); // 元の行き先を覚えておく
    return NextResponse.redirect(loginUrl);
  }

  // 書き換え（rewrite）：URLは /blog のまま、内部的に別パスを表示
  if (pathname === "/blog") {
    return NextResponse.rewrite(new URL("/blog/latest", request.url));
  }

  // 何もなければそのまま通す（レスポンスにヘッダを足すことも可能）
  const response = NextResponse.next();
  response.headers.set("x-custom-header", "hello");
  return response;
}

// matcher で「どのパスでmiddlewareを動かすか」を限定する
export const config = {
  matcher: ["/dashboard/:path*", "/blog", "/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

## 実務での使い方・定番パターン
- **`matcher` で対象パスを絞る**のが基本。指定しないと**全リクエストで実行**され、`_next/static` などの静的アセットにも無駄に走る。「保護したいパスだけ」に限定する。
- **認証ガード**：Cookie のセッショントークン有無で `redirect`。本格的なトークン検証（JWT署名チェック等）は Edge で重くなりがちなので、**存在チェックなど軽い判定に留め**、厳密な認可は Server Component / Route Handler 側で行うのが定番。
- **A/B テスト**：初回アクセスで Cookie を振り分け、`rewrite` でバリアントを出し分ける。`NextResponse` に `cookies.set()` して以降を固定する。

```tsx
// A/Bテスト：初回にバケットを決めてCookieで固定
export function middleware(request: NextRequest) {
  const bucket = request.cookies.get("ab")?.value
    ?? (Math.random() < 0.5 ? "a" : "b");
  const url = request.nextUrl.clone();
  url.pathname = `/home-${bucket}`;        // /home-a or /home-b を内部表示
  const res = NextResponse.rewrite(url);
  res.cookies.set("ab", bucket, { maxAge: 60 * 60 * 24 * 30 });
  return res;
}
```

- **ヘッダ操作**：`NextResponse.next()` にセキュリティヘッダ（CSP 等）を足す用途にも使える。
- **API（Route Handlers）の前段**にも効く。レート制限の前さばきや共通認証ヘッダ確認など。→ [route_handlers.md](./route_handlers.md)

## ハマりどころ / アンチパターン
- **Edge ランタイムの制約**：`middleware.ts` は Edge で動くため、**Node.js 固有 API が使えない**（`fs` でファイル読み書き、`net`、一部の Node 専用ライブラリ等）。`bcrypt` のようなネイティブ依存ライブラリも基本動かない。DB へ直接接続するような重い処理も不向き。
- **`matcher` 設定ミス**：絞り忘れて全リクエストに走らせる → 静的アセットまで巻き込んで遅くなる。逆に絞りすぎて保護したいパスが漏れる、も事故。`((?!_next/...).*)` のような除外パターンは定番だがテスト必須。
- **重い処理を載せる**：middleware は**全対象リクエストで毎回動く**ホットパス。DB問い合わせ・外部API待ち・重い暗号処理を入れると全体が遅くなる。重い処理は通常のページ/Route Handler 側へ。
- **`middleware.ts` の場所間違い**：`app/` の中に置いても効かない。**プロジェクト直下（または `src/` 直下）に1つだけ**。
- **`return` 忘れ**：`NextResponse` を返さないと意図通りに制御できない。素通しなら `NextResponse.next()` を明示的に返す。

## 関連: [route_handlers.md](./route_handlers.md)
