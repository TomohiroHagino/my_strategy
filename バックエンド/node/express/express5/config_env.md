# 設定・環境変数（dotenv / process.env）（Express 5）

## ひとことで言うと
- アプリの**設定値を環境変数で外部化**し、コードから分離する仕組み。
- **dotenv** が `.env` ファイルを読み込んで `process.env` に流し込む。
- `NODE_ENV`（development / production / test）で**環境ごとの挙動を分岐**するのが定番。

## 役割・なぜ必要か
- **環境差を吸収**：DB接続先・APIキー・ポートは開発と本番で違う。コードを変えずに値だけ差し替えたい。
- **機密の分離**：secret をソースに書かない（漏洩防止）。`.env` は**コミットしない**。
- **設定の一元化**：散らばった `process.env.X` を1オブジェクトに集約すると、起動時に欠落を検知でき、参照も型安全に近づく。
- **環境分岐**：ログ詳細度・CORS・エラー表示などを `NODE_ENV` で切り替える。

## 基本の書き方（コード）
### .env の読み込み（アプリ最初の1行で）
```js
import "dotenv/config";   // これだけで .env → process.env に反映（他のimportより前に）
// 旧来の書き方: import dotenv from "dotenv"; dotenv.config();

console.log(process.env.PORT);   // "3000"（文字列で入る点に注意）
```

```bash
# .env（リポジトリにコミットしない）
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://user:pass@localhost:5432/app
JWT_SECRET=長いランダム文字列をここに
CORS_ORIGINS=http://localhost:5173
```

### 設定を1オブジェクトに集約＋起動時に検証
```js
import "dotenv/config";

function required(name) {
  const v = process.env[name];
  if (v === undefined || v === "") {
    throw new Error(`Missing env var: ${name}`);   // 起動時に即落とす（fail fast）
  }
  return v;
}

export const config = {
  env: process.env.NODE_ENV ?? "development",
  port: Number(process.env.PORT ?? 3000),   // 数値は明示変換（env は常に文字列）
  databaseUrl: required("DATABASE_URL"),
  jwtSecret: required("JWT_SECRET"),
  corsOrigins: (process.env.CORS_ORIGINS ?? "").split(",").filter(Boolean),
  isProd: process.env.NODE_ENV === "production",
};
```

### NODE_ENV で挙動を分岐
```js
import express from "express";
import { config } from "./config.js";

const app = express();

if (config.isProd) {
  app.set("trust proxy", 1);                       // 本番：プロキシ背後
} else {
  const morgan = (await import("morgan")).default; // 開発：詳細ログ
  app.use(morgan("dev"));
}

// エラー表示も分岐：本番は詳細を隠す
app.use((err, req, res, next) => {
  console.error(err);                              // サーバ側には常に詳細を残す
  res.status(500).json(
    config.isProd ? { error: "Internal Server Error" } : { error: err.message, stack: err.stack }
  );
});

app.listen(config.port, () => console.log(`listening on ${config.port} (${config.env})`));
```

### 環境ごとの .env ファイル
```bash
# 起動時に NODE_ENV を渡す（クロスプラットフォームは cross-env が便利）
NODE_ENV=production node server.js

# .env.example をコミットして「必要なキーの一覧」を共有（値はダミー）
cp .env.example .env   # 各自が値を埋める
```

## 実務での使い方・定番パターン
- **`.env.example` をコミット**：必要なキーの一覧と形式を共有。実値の `.env` は除外する。
- **設定は `config.js` 1か所**：アプリ全体は `config` 経由で参照。直接 `process.env.X` を散らさない。
- **起動時バリデーション**：必須キーが無ければ `throw` で即停止（本番で初リクエスト時に落ちるより安全）。
- **本番は秘密管理サービス**：AWS Secrets Manager / Doppler / Vault 等。`.env` ファイルはローカル開発向き。
- **型変換を明示**：`Number()` / `=== "true"` で number・boolean に変換（env は全部 string）。
- **`NODE_ENV=test`**：テスト時はDBやログを切り替え、副作用を隔離する。

## ハマりどころ / アンチパターン
- **`.env` をコミットしてしまう（最頻・最重大）**：secret 漏洩。`.gitignore` に `.env`／`.env.*`（`.env.example` は除外解除）を入れる。漏れたら即ローテーション。
- **`process.env.X` が undefined**：`dotenv/config` を**他importより前**に読まないと、参照時点で未設定になる。読込順が原因の典型。
- **`NODE_ENV` 未設定**：多くのライブラリは未設定だと開発モード扱い。本番で `NODE_ENV=production` を必ず設定（性能・セキュリティに影響）。
- **数値・真偽の比較ミス**：`process.env.PORT === 3000` は常に false（文字列 "3000" だから）。変換してから比較する。
- **`if (process.env.FLAG)`**：文字列 `"false"` も truthy。明示的に `=== "true"` で判定。
- **設定をハードコード**：URL や鍵をコードに直書きすると環境移行で破綻。必ず env 経由にする。

## 関連
[getting_started.md](./getting_started.md) / [security.md](./security.md)
