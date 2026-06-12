# セキュリティ（helmet / CORS / レート制限）（Express 5）

## ひとことで言うと
- Web API を守るための**標準的な防御層**を、ミドルウェアで足していく作業。
- 4本柱＝**helmet**（セキュリティヘッダ）／**cors**（オリジン制御）／**express-rate-limit**（レート制限）／**入力検証・サニタイズ**。
- いずれも Express 本体には**入っていない**。明示的に導入しない限り無防備。

## 役割・なぜ必要か
- **helmet**: ブラウザに「安全側の挙動」を指示するHTTPヘッダを一括設定（XSS緩和・クリックジャッキング防止等）。
- **cors**: どのオリジンからのブラウザ越しアクセスを許すかを制御。デフォルトは同一オリジンのみ。
- **express-rate-limit**: 同一IP等からの大量リクエストを制限し、総当たり・DoS・スクレイピングを抑止。
- **入力検証・サニタイズ**: 外部入力を信用しない。インジェクション（SQLi/NoSQLi）やXSSの根を断つ。
- **secret管理**: 鍵・パスワードはコードに書かず `.env`／秘密管理に外出し。

## 基本の書き方（コード）
### helmet：セキュリティヘッダを一括付与
```js
import helmet from "helmet";

app.use(helmet());   // X-Content-Type-Options, X-Frame-Options, HSTS など妥当な既定を一括

// API＋自前フロントなら CSP を明示すると強い（雛形は環境に合わせて調整）
app.use(helmet.contentSecurityPolicy({
  directives: { defaultSrc: ["'self'"], objectSrc: ["'none'"], frameAncestors: ["'none'"] },
}));
```

### cors：許可オリジンを明示（ワイルドカード乱用は禁）
```js
import cors from "cors";

const allowed = (process.env.CORS_ORIGINS || "").split(",");  // 例: "https://app.example.com"
app.use(cors({
  origin: (origin, cb) =>
    !origin || allowed.includes(origin) ? cb(null, true) : cb(new Error("CORS blocked")),
  credentials: true,   // Cookie/セッション併用時はtrue。ただし origin:"*" とは併用不可
}));
```

### express-rate-limit：レート制限
```js
import rateLimit from "express-rate-limit";

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,   // 15分
  max: 100,                   // IPあたり100リクエストまで
  standardHeaders: true,
  legacyHeaders: false,
});
app.use("/api/", limiter);

// ログイン等は別枠で厳しく（総当たり対策）
app.use("/api/login", rateLimit({ windowMs: 15 * 60 * 1000, max: 5 }));
```

### CSRF：セッション（Cookie認証）を使うときのみ必要
```js
// JWT を Authorization ヘッダで送る方式なら原則不要。Cookieセッション時に対策する。
import csrf from "csurf";
app.use(csrf());                                  // フォーム送信にトークンを要求
app.get("/form", (req, res) => res.json({ csrfToken: req.csrfToken() }));
// SameSite=Lax/Strict のCookie設定も併用すると堅い
```

### 入力サニタイズ／インジェクション対策
```js
// SQL：文字列連結は厳禁。必ずパラメータ化（プレースホルダ）
// NG: db.query(`SELECT * FROM users WHERE id = ${req.params.id}`)
db.query("SELECT * FROM users WHERE id = $1", [req.params.id]);   // OK

// NoSQL（Mongo）：オブジェクト注入を防ぐ。値が文字列か検証してから使う
const id = String(req.params.id);                 // {"$gt":""} 等の注入を無効化
const user = await User.findById(id);
```

## 実務での使い方・定番パターン
- **導入順序**：`helmet` → `cors` → ボディ解析（`express.json()`）→ `rate-limit` → ルータ、という順でアプリ先頭に並べる。
- **環境で分岐**：開発は CORS を緩め、本番は許可オリジンを限定。`NODE_ENV` で切り替える（config_env.md 参照）。
- **検証ライブラリ併用**：境界で zod / express-validator により型・形を検証してからロジックへ（validation.md）。
- **エラーで情報を漏らさない**：本番はスタックトレースを返さず、汎用メッセージ＋サーバ側で詳細ログ。
- **HTTPS前提のCookie**：`secure: true` / `httpOnly: true` / `sameSite` を必ず設定。
- **依存の脆弱性監視**：`npm audit` を CI に組み込み、放置しない。

## ハマりどころ / アンチパターン
- **CORS設定ミス**：`origin: "*"` ＋ `credentials: true` は仕様上両立せずブラウザが拒否。許可オリジンは配列で明示する。プリフライト（OPTIONS）が通らない時は許可ヘッダ/メソッド設定漏れを疑う。
- **helmet/レート制限を未導入のまま本番投入**：デフォルト無防備。最低限この2つは入れる。
- **secrets漏れ**：`JWT_SECRET` 等をコードや `.env` ごとコミット。`.gitignore` に `.env` を入れ、漏れたら即ローテーション。
- **文字列連結クエリ**：SQLi の温床。プレースホルダ必須。ORM任せでも raw クエリでは要注意。
- **NoSQLにオブジェクトをそのまま渡す**：`{ email: req.body.email }` に `{$ne:null}` を注入される。型を固定してから使う。
- **CSRFを全部に付ける/全く付けない**：JWTヘッダ方式に CSRF は不要、Cookieセッションには必要。方式に応じて判断する。
- **レート制限がプロキシ背後で効かない**：`app.set("trust proxy", 1)` を設定しないと全リクエストが同一IP扱いになる。

## 関連
[config_env.md](./config_env.md) / [validation.md](./validation.md)
