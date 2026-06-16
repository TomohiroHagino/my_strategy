# Node.js

## 一言で
**サーバ側で JavaScript を動かすランタイム**（V8エンジン上）。**イベントループによる非同期・ノンブロッキングI/O**が核で、大量の同時接続を少ないリソースで捌くのが得意。フロントと同じ言語(JS/TS)でバックエンドを書ける。

## 特徴
- **イベントループ / ノンブロッキングI/O**：1スレッドで「I/O待ちの間に別の処理」を回す（→ I/O並行が強い）。
- **シングルスレッド前提**：CPUバウンドな重い処理は苦手（別ワーカー/別プロセスへ）。
- **npm エコシステムが巨大**：パッケージ数が世界最大級。
- **CommonJS と ESM**：`require`（CJS）と `import`（ESM）の2系統。新規はESM寄り。
- **TypeScript と相性が良い**：型を付けて大規模化に耐える。

## どういう使い方をするのか
- **REST / GraphQL API・BFF**（フロントと同言語で書ける）。
- **リアルタイム**（WebSocket / Socket.IO・チャット・通知）。
- **マイクロサービス・サーバレス**（軽量・起動速い）。
- ツール・CLI・ビルド基盤（フロント開発の土台でもある）。

## 強み / 弱み
- 強み：I/O並行に強い・起動軽量・フロントと言語統一・エコシステム巨大。
- 弱み：CPUバウンド処理が苦手（イベントループを塞ぐと全体停止）・コールバック/非同期の設計が要る・パッケージ品質のばらつき。

## エコシステム・周辺
- パッケージ: npm / pnpm / yarn、実行: `node` / nodemon / tsx
- フレームワーク: **Express**（→ [express/](./express/)）/ Fastify / **NestJS**（構造化）/ Hono
- ORM: Prisma / Sequelize / TypeORM、Mongo: Mongoose
- テスト: Jest / Vitest ＋ supertest

## このフォルダの構成
- [express/](./express/) … Express（定番の軽量Webフレームワーク）。フラッグシップの版別リファレンス。
- [hono/](./hono/) … Hono（マルチランタイム・Edge）。フラッグシップの版別リファレンス。
- [落とし穴.md](./落とし穴.md) … **Node特有の現象と対策カタログ**（イベントループ遮断・未処理reject/クラッシュ・CJS/ESM・ストリームのバックプレッシャ・error-firstコールバック地獄・EventEmitterリーク・nextTick/setImmediate・cluster/Worker・環境変数/秘密）。JS共通の罠は [../../フロントエンド/javascript/落とし穴.md](../../フロントエンド/javascript/落とし穴.md)。
