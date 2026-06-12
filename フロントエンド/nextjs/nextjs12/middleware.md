# ミドルウェア（middleware.ts）（Next.js 12・Pages Router）

## ひとことで言うと
プロジェクト直下の `middleware.ts` に書いた関数が、**リクエストがページ/APIに届く前に Edge で走り**、`NextRequest` を見て `NextResponse` でリダイレクト・書き換え・ヘッダ付与・cookie操作を行う——認証ガードやABテスト、i18nの入口。

## 役割・なぜ必要か
- 「ログインしていなければ `/login` へ」のような**ページに入る前の判定**を、各ページの `getServerSideProps` に書かず1か所で済ませられる。
- Edge（CDN相当の場所）で動くので**速く・全リクエストに横断的**にかけられる。ABテストの振り分け、地域別リダイレクト、ヘッダ注入に向く。
- Next.js 12.2 で安定版になった機能。`config.matcher` で**どのパスに効かせるか**を絞れる。

## 基本の書き方（コード）

```ts
// middleware.ts（プロジェクト直下・pages/ や app/ と同階層）
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(req: NextRequest) {
  // 認証ガード：cookie が無ければ /login へ
  const token = req.cookies.get("token");
  if (!token) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("from", req.nextUrl.pathname); // 戻り先を渡す
    return NextResponse.redirect(url);
  }
  // 通す（ヘッダだけ足す例）
  const res = NextResponse.next();
  res.headers.set("x-app", "pages-router");
  return res;
}

// どのパスに効かせるか（指定しないと全パス）
export const config = {
  matcher: ["/dashboard/:path*", "/settings/:path*"],
};
```

```ts
// 書き換え（rewrite）：URLはそのままで内部的に別ページを返す（ABテスト等）
export function middleware(req: NextRequest) {
  const bucket = req.cookies.get("ab")?.value ?? (Math.random() < 0.5 ? "a" : "b");
  const url = req.nextUrl.clone();
  url.pathname = `/home-${bucket}`; // 見た目のURLは "/" のまま中身を差し替え
  const res = NextResponse.rewrite(url);
  res.cookies.set("ab", bucket); // 振り分けを固定
  return res;
}
```

```ts
// matcher を複数 / 否定で絞る（静的アセットや api を除外）
export const config = {
  matcher: [
    // _next/static・_next/image・favicon を除く全パス
    "/((?!_next/static|_next/image|favicon.ico).*)",
  ],
};
```

## 実務での使い方・定番パターン
- **認証ガード**：保護したいパスを `matcher` で列挙し、cookie/JWTを見て未認証なら `redirect`。各ページに書くより漏れない。
- **i18nロケール判定**：`Accept-Language` やパスを見て `/ja`/`/en` へ rewrite/redirect。
- **ABテスト/段階リリース**：`rewrite` で内部的に別バリアントを返し、cookie で振り分けを固定。
- **ヘッダ注入**：セキュリティヘッダや相関IDを `NextResponse.next()` に足す。
- **`matcher` で範囲を最小化**：全リクエストに走らせると無駄。効かせたいパスだけに絞る。

## ハマりどころ / アンチパターン
- **Edge ランタイムの制約**：Node 専用API（`fs`、多くのDBドライバ、Node の `crypto` 一部）は使えない。重い処理やDBアクセスは API ルートや `getServerSideProps` 側で。
- **`NextResponse` を返し忘れる**：何も返さない/`undefined` だと意図せず素通り。明示的に `NextResponse.next()`/`redirect`/`rewrite` を返す。
- **`matcher` 未指定で全パス**：`_next/*` や静的アセットにも走り重くなる。否定マッチで除外する。
- **`redirect` ループ**：`/login` 自体にもガードが効くと無限リダイレクト。`matcher` から除外するか、パスを条件分岐。
- **App Router 専用機能と混同**：`middleware.ts` 自体は両Routerで共通だが、ここで `getServerSideProps` 的なデータ取得はしない。ミドルウェアは判定・書き換え専用。
- **重い同期処理**：Edgeは短時間実行前提。外部API待ちを大量に挟むと全リクエストが遅くなる。

## 関連
[api_routes.md](./api_routes.md) / [data_fetching.md](./data_fetching.md) / [deployment.md](./deployment.md) / [pitfalls.md](./pitfalls.md)
