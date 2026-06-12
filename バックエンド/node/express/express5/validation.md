# バリデーション（zod / express-validator）（Express 5）

## ひとことで言うと
クライアントから来る入力（body / query / params / headers）を**信頼せず検証**する仕組み。**zod**（型＋検証を統合・人気）/ `express-validator` / `joi` などを使い、**検証をミドルウェア化してルートの前に挟む**。失敗したら処理に進まず **400 ＋ エラー詳細**で返す。

## 役割・なぜ必要か
- 外部から来るデータは何でもあり得る（欠損・型違い・巨大値・悪意）。**検証なしで DB やビジネスロジックに流すと、バグ・データ破損・セキュリティ事故**になる。
- フロント側の検証は UX 用で**信頼できない**（直接 API を叩かれる）。**サーバ側の検証が必須の防御線**。
- TypeScript の型は「コンパイル時」だけで実行時は素通り。**実行時の検証**が別途要る。zod なら型と実行時検証を**一本化**できる。

---

## なぜサーバ側が必須か（信頼しない）
```
ブラウザのJSバリデーション → 迂回可能（curl / Postman で直接POST）
                            ↓
        サーバ側バリデーション ← ここが本当の防御線（必須）
```
- 「フロントで弾いてるから大丈夫」は通用しない。**境界（ルート入口）で必ず検証**する。

## zod の基本（型＋検証統合・人気）
```js
const { z } = require('zod')

const CreateUser = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).max(150),
  name: z.string().min(1).max(100),
})

app.post('/users', (req, res) => {
  const parsed = CreateUser.safeParse(req.body)
  if (!parsed.success) {
    return res.status(400).json({ errors: parsed.error.flatten() })
  }
  const data = parsed.data // ← 検証済み。TS なら型も付く
  // ... 以降は安全なデータで処理
})
```
- `safeParse` は throw せず成否を返す（`success` を分岐）。`parse` は失敗で throw（Express 5 なら error 段へ自動転送）。

## 検証をミドルウェアにして前に挟む（定番）
```js
// middleware/validate.js : スキーマを受けて検証ミドルウェアを返す
const validate = (schema, target = 'body') => (req, res, next) => {
  const result = schema.safeParse(req[target])
  if (!result.success) {
    return res.status(400).json({ errors: result.error.flatten() })
  }
  req[target] = result.data // 正規化済みで上書き
  next()
}
module.exports = { validate }
```
```js
// ルートで前段に挟む → ハンドラには検証済みデータしか来ない
const { validate } = require('./middleware/validate')
app.post('/users', validate(CreateUser), createUserHandler)
app.get('/users', validate(ListQuery, 'query'), listUsersHandler)
```
- これで**ハンドラ本体は「正しい入力」前提**で書ける（責務分離）。ミドルウェア全般は [middleware.md](./middleware.md)。

## express-validator の基本（別の定番）
```js
const { body, validationResult } = require('express-validator')

app.post('/users',
  body('email').isEmail(),
  body('age').isInt({ min: 0 }),
  (req, res) => {
    const errors = validationResult(req)
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() })
    }
    // ... 処理
  }
)
```
- ルールをチェーンで宣言 → 最後に `validationResult` でまとめて判定。

## 失敗時のレスポンス（400 ＋ 詳細）
```js
// 一貫した形で返すと、フロントが扱いやすい
res.status(400).json({
  success: false,
  error: 'ValidationError',
  details: parsed.error.flatten().fieldErrors, // { email: ['Invalid email'] }
})
```
- ステータスは **422 Unprocessable Entity** を使う流派もある。チーム内で統一する（[request_response.md](./request_response.md)）。
- **エラーメッセージに内部情報（スタックトレース・SQL）を載せない**。

## 実務での使い方・定番パターン
- **入力ごとにスキーマを1つ**作り、`validate(schema)` でルート前段に並べる。
- **query / params も検証**する（数値化・範囲・列挙）。`z.coerce.number()` で文字列→数値も統一。
- **出力(レスポンス)整形にも zod を流用**できる（過剰なフィールドの除去）。
- TS なら `z.infer<typeof CreateUser>` で**型を生成**し、手書き型との二重管理を排除。

```js
const PageQuery = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
})
```

## ハマりどころ / アンチパターン
- **サーバ側検証を省略**：フロント検証だけは迂回される。**必須**。
- **検証場所がバラバラ**：ハンドラ内に検証ロジックが散らばる → **ミドルウェアに集約**。
- **型と実行時検証の二重化**：TS の `interface` と検証ルールを別々に書くと乖離する → **zod で一本化**（`z.infer` で型生成）。
- **query/params の未検証**：`req.query.page` は文字列。数値前提のコードが壊れる。`coerce`/`Number` で正規化。
- **エラー詳細の出しすぎ**：内部例外をそのまま返すと情報漏洩。整形して返す。
- **巨大ボディの未制限**：`express.json({ limit: '100kb' })` 等でサイズ上限を設ける。

## 関連
[request_response.md](./request_response.md) / [middleware.md](./middleware.md)
