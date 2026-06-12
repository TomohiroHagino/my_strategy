# テスト（Testing）（Hono v4）

## ひとことで言うと
Honoは **`app.request()`** でテストする。Web標準の `Request` を渡し、返ってきた `Response` を検証するだけ。**サーバを起動せず**（ポートを開かず）、`fetch` 同等の入出力でルートを叩ける。テストランナーは **Vitest** が定番。

## 役割・なぜ必要か
- Honoのアプリは「`Request` を受けて `Response` を返す関数」そのもの。だから**ランタイム起動不要**で、`app.request('/users')` を呼ぶだけで結合テストができる。
- 起動・ポート・待ち時間が無いので**高速かつ安定**。Workers/Deno/Bun/Nodeどの環境向けコードでも、テストは同じ書き方で回せる。
- HTTPステータス・JSONボディ・ヘッダを Web標準のまま検証でき、実態に近い。

## 基本の書き方（コード）
```ts
// src/index.ts（テスト対象）
import { Hono } from 'hono'
const app = new Hono()
app.get('/users', (c) => c.json([{ id: 1, name: 'Taro' }]))
app.post('/users', async (c) => {
  const body = await c.req.json<{ name: string }>()
  return c.json({ id: 2, ...body }, 201)
})
export default app
```
```ts
// src/index.test.ts（Vitest + app.request）
import { describe, it, expect } from 'vitest'
import app from './index'

describe('users API', () => {
  it('一覧が200を返す', async () => {
    const res = await app.request('/users')        // GET 既定
    expect(res.status).toBe(200)
    const data = await res.json()                  // ← await 必須
    expect(data).toHaveLength(1)
  })

  it('POSTで作成され201を返す', async () => {
    // 第2引数で method / body / headers を指定（fetch同様）
    const res = await app.request('/users', {
      method: 'POST',
      body: JSON.stringify({ name: 'Hanako' }),
      headers: { 'Content-Type': 'application/json' },
    })
    expect(res.status).toBe(201)
    expect(await res.json()).toEqual({ id: 2, name: 'Hanako' })
  })
})
```
```ts
// c.env（バインディング）が要るルートはモックを第3引数で注入
const MOCK_ENV = { DB: fakeD1, TOKEN: 'test' }
const res = await app.request('/protected', {
  headers: { Authorization: 'Bearer test' },
}, MOCK_ENV)   // ← 第3引数 env でWorkers環境を差し込む
expect(res.status).toBe(200)
```
```bash
# Vitest 実行
npm i -D vitest
npx vitest run          # 一括実行
npx vitest              # watch
npx vitest run --coverage
```

## 実務での使い方・定番パターン
- **`app.request(path, init?, env?)` が主役**：第1引数はパス（または `Request` オブジェクト）、第2引数は `fetch` と同じ `init`（method/body/headers）、第3引数で `c.env` のモックを注入できる。
- **AAA構成**：Arrange（入力組み立て）→ Act（`await app.request(...)`）→ Assert（`res.status` / `await res.json()`）。Railsのrequest specに近い感覚で書ける。
- **テストピラミッド**：純粋ロジック/ユーティリティはユニットで、ルート結合は `app.request` で、重要フローだけ実際のランタイム上でE2E（少数）。
- **`c.env` のモック**：D1やシークレットは第3引数で偽物を渡す。`vi.fn()` でDBクライアントをスタブ化し、外部依存だけ差し替える。
- **`vitest.config.ts`**：Workers向けの挙動を厳密に再現したいなら `@cloudflare/vitest-pool-workers`、Node環境で十分なら素のVitestでよい。

## ハマりどころ / アンチパターン
- **`res.json()` の `await` 忘れ**：`Response.json()` は **Promise**。`await` を付けないと中身でなくPromiseをassertして落ちる。`res.text()` も同様。
- **Vitest設定の取り違え**：`c.env` やWorkers固有APIを使うのにNode環境のまま回すと挙動が合わない。バインディング依存が強いなら workers プールを検討。
- **`c.env` をモックし忘れる**：env依存ルートを第3引数なしで叩くと `undefined` 参照で落ちる。必要なバインディングを渡す。
- **POSTで `Content-Type` 漏れ**：`c.req.json()` 側がJSONとして読めず失敗。`headers` に `application/json` を必ず付ける。
- **実サーバを立ててテスト**：`serve()` でポートを開いて叩くのは遅く不安定。`app.request` で十分（起動不要が利点）。
- **状態リーク**：モジュールスコープの可変状態（キャッシュ等）がテスト間で残る。`beforeEach` で初期化。

## 関連
[vitest.md](./vitest.md) / [routing.md](./routing.md)
