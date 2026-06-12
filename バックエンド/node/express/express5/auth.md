# 認証・認可（JWT / セッション / Passport）（Express 5）

## ひとことで言うと
- **認証（Authentication）= 「あなたは誰か」** を確かめること（ログイン）。
- **認可（Authorization）= 「その操作をやってよいか」** を判断すること（本人・権限チェック）。
- Express は **認証機能を内蔵しない**。自前で組むか Passport 等の枠組みを使う。2大方式＝**JWT**（ステートレス）と**セッション**（ステートフル）。

## 役割・なぜ必要か
- 認証だけでは「ログイン済みなら何でもできる」状態。**他人のリソースを操作できてしまう（IDOR）**。
- 認可は「このユーザーが、この対象に、この操作を許されているか」を1か所に集約する。
- **JWT**: トークンに本人情報を署名して持たせる。サーバは状態を持たない（=スケールしやすい、SPA/モバイルAPI向き）。
- **セッション**: サーバ側にログイン状態を保存し、Cookie の session id で照合（=失効・無効化が容易、Webアプリ向き）。

## 基本の書き方（コード）
### パスワードハッシュ（bcrypt）— 平文保存は厳禁
```js
import bcrypt from "bcrypt";

const hash = await bcrypt.hash(password, 12);      // 登録時：保存するのはこのhashだけ
const ok = await bcrypt.compare(password, hash);   // ログイン時：一致判定（true/false）
```

### 方式A：JWT（jsonwebtoken・ステートレス）
```js
import jwt from "jsonwebtoken";

// ログイン成功時にトークンを発行
const token = jwt.sign(
  { sub: user.id, role: user.role },   // payload（機密は入れない＝誰でも中身は読める）
  process.env.JWT_SECRET,              // 署名鍵（.envで管理）
  { expiresIn: "1h" }                  // 期限。短め＋リフレッシュトークン運用が定番
);
res.json({ token });

// 保護ミドルウェア：トークン検証 → req.user に載せる
function requireAuth(req, res, next) {
  const header = req.headers.authorization || "";
  const token = header.startsWith("Bearer ") ? header.slice(7) : null;
  if (!token) return res.status(401).json({ error: "no token" });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET); // 改ざん/期限切れなら例外
    next();
  } catch {
    return res.status(401).json({ error: "invalid token" });
  }
}
```

### 方式B：セッション（express-session ＋ ストア・ステートフル）
```js
import session from "express-session";
import { RedisStore } from "connect-redis";   // 本番はメモリストア禁止

app.use(session({
  store: new RedisStore({ client: redisClient }), // 既定のMemoryStoreは再起動で消え・本番不可
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, sameSite: "lax", maxAge: 3600_000 },
}));

// ログイン：サーバ側セッションに本人を記録
req.session.userId = user.id;

// 保護ミドルウェア：セッションの有無で判定
function requireAuth(req, res, next) {
  if (!req.session.userId) return res.status(401).json({ error: "login required" });
  next();
}
```

### 方式C：Passport（認証戦略の枠組み）
```js
import passport from "passport";
import { Strategy as LocalStrategy } from "passport-local";

passport.use(new LocalStrategy(async (email, password, done) => {
  const user = await findUserByEmail(email);
  if (!user || !(await bcrypt.compare(password, user.hash))) {
    return done(null, false);              // 認証失敗
  }
  return done(null, user);                 // 成功 → req.user に載る
}));
app.post("/login", passport.authenticate("local", { session: false }), (req, res) => {
  res.json({ ok: true, userId: req.user.id });
});
```
> Passport は「**戦略（Strategy）を差し替える枠組み**」。local / jwt / google など多数の戦略を同じ作法で扱える。

## 実務での使い方・定番パターン
- **保護はミドルウェアに集約**：`router.use(requireAuth)` でルータ単位、または個別ルートに付ける。検証成功で `req.user` を確定させ、以降のハンドラはそれを信頼する。
- **認可の第一歩＝スコープで絞る**：取得時に必ず本人条件を入れる。`prisma.post.findFirst({ where: { id, userId: req.user.sub } })`。URLのIDを差し替えても他人に届かない。
- **権限チェックのミドルウェア化**：
```js
const requireRole = (role) => (req, res, next) =>
  req.user?.role === role ? next() : res.status(403).json({ error: "forbidden" });
app.delete("/admin/users/:id", requireAuth, requireRole("admin"), handler);
```
- **JWTのリフレッシュ**：アクセストークンは短命（15分〜1h）、リフレッシュトークンは長命でDB管理＆失効可能に。
- **方式の選択**：SPA/モバイル＝JWT、サーバレンダリングのWeb＝セッション。混在させない。

## ハマりどころ / アンチパターン
- **認証だけして認可を忘れる（最頻・最重大）**：ログインさえ通れば `params.id` で他人のリソースを編集・削除できる（IDOR）。必ずスコープ or 権限チェックで絞る。
- **403と401の混同**：未ログイン＝401、ログイン済みだが権限なし＝403。
- **JWT secret の弱さ/漏れ**：短い文字列・コミット混入は致命的。`.env` で長いランダム値を。漏れたら全トークン失効が必要。
- **JWTの期限・検証漏れ**：`jwt.verify`（検証）を `jwt.decode`（検証なし）と取り違えない。`expiresIn` 必須。
- **JWTは無効化しづらい**：ログアウト即時失効が要るならブラックリスト or セッション併用。
- **セッションを本番でMemoryStore運用**：プロセス再起動で全ログアウト・水平スケール不可。**Redis等の外部ストア必須**。
- **平文パスワード保存・平文ログ出力**：`bcrypt` で必ずハッシュ。ログにパスワードを出さない。

## 関連
[middleware.md](./middleware.md) / [security.md](./security.md)
