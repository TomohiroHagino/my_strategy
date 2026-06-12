# async/await とイベントループ（Express 5）

## ひとことで言うと
Node.js は **シングルスレッドのイベントループ**でノンブロッキングに動く実行モデル。Express のハンドラは `async/await` で書き、I/O 待ちの間も他のリクエストを処理し続ける。Express 5 では `async` 関数が reject すると自動でエラーハンドラへ転送される。

## 役割・なぜ必要か
- Node は「1本のスレッドで待ち時間を捨てずに回す」ことで多数の同時接続をさばく。DB・HTTP・ファイルなどの **I/O は待たずに次へ進み、終わったらコールバック/Promise で戻る**。
- だから「待ち」は得意だが「**重い計算（CPU）でループを占有する処理は全リクエストを止める**」。ここを理解しないと、たった1本の重い処理でサーバ全体が固まる。
- `async/await` は Promise を同期的な見た目で書ける糖衣構文。エラーは `try/catch` で受けられ、Express 5 なら投げっぱなしでも error middleware に届く。

---

## イベントループとは（ノンブロッキングの核）
```
[受信] → コールスタックで同期処理を実行
         I/O（DB/HTTP/FS）は OS/スレッドプールへ委譲 → スタックは空く
         完了したものをキューから拾って続きを実行（= ループが回る）
```
- 単一スレッドなので、**コールスタックを長時間占有する処理（重い for ループ、巨大 JSON、暗号計算、同期 I/O）はループを止める**。その間、他の全リクエストが待たされる。
- I/O は委譲されるので「待ち」では止まらない。止めるのは **CPU バウンドな同期処理**。

## イベントループを塞がない（最重要）
```js
// NG: 重い同期計算でループを止める（他のリクエストが全部固まる）
app.get('/fib', (req, res) => {
  const n = Number(req.query.n)
  res.json({ result: heavyFib(n) }) // ループ占有 → サーバ全体が無反応に
})

// NG: 同期 I/O（fs.readFileSync 等）もループを止める
const data = fs.readFileSync('/big/file') // 起動時の一度きり以外は避ける
```
- **対策1: worker_threads** に逃がす（CPU 計算用の別スレッド）。
- **対策2: 別プロセス/ジョブキュー**（BullMQ など）に投げて非同期化。
- **対策3: 同期 I/O を非同期版に**（`fs.promises.readFile` を `await`）。
```js
// OK: 重い計算は worker に逃がす
const { Worker } = require('node:worker_threads')
app.get('/fib', (req, res, next) => {
  const w = new Worker('./fib-worker.js', { workerData: Number(req.query.n) })
  w.once('message', (result) => res.json({ result }))
  w.once('error', next)
})
```

## async ハンドラの基本
```js
// OK: I/O は await で待つ。Express 5 は throw/reject を自動転送
app.get('/users/:id', async (req, res) => {
  const user = await db.user.findById(req.params.id) // 待っても他は止まらない
  if (!user) return res.status(404).json({ error: 'not found' })
  res.json(user)
})
```
- 4 では `try/catch` して `next(err)` する/ラッパで包む必要があったが、**5 は不要**（詳細は [error_handling.md](./error_handling.md)）。
- それでも **業務上の分岐を握り潰さない**ため、必要なら `try/catch` を明示する。

## Promise の基本（await の土台）
```js
// 直列: 前の結果が必要なときだけ
const u = await getUser(id)
const posts = await getPosts(u.id)

// 並行: 互いに独立なら同時に走らせて待ち時間を短縮
const [a, b] = await Promise.all([getA(), getB()])

// 一部失敗を許容して全件待つ
const results = await Promise.allSettled([f1(), f2(), f3()])
```
- **直列の `await` 連発は遅い**。独立な I/O は `Promise.all` で並行化する。

## 実務での使い方・定番パターン
```js
// 共通: 業務エラーは投げて error middleware に集約（Express 5）
app.post('/orders', async (req, res) => {
  const order = await createOrder(req.body) // 失敗 → 自動で error 段へ
  res.status(201).json(order)
})

// タイムアウト付き呼び出し（外部 API が無反応でも詰まらせない）
function withTimeout(promise, ms) {
  return Promise.race([
    promise,
    new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), ms)),
  ])
}
const data = await withTimeout(fetchExternal(), 3000)
```
- **重い処理は「リクエスト中にやらない」**：レスポンスは即返し、実処理はジョブキューへ。
- 大量データは一括ロードせず**ストリーム/バッチ**で。メモリと GC でループが詰まるのを防ぐ。

## ハマりどころ / アンチパターン
- **重い同期処理でループ停止**（最頻・最重要）：1本の重い計算で全リクエストが固まる → `worker_threads` / 別プロセスへ。
- **`await` 忘れ**：`const u = db.find(id)`（await なし）で Promise を値扱い → undefined/型ズレ。`return` 漏れも同様にバグる。
- **未処理 Promise rejection**：`await` も `.catch()` も付けない reject は `unhandledRejection`。プロセス全体を巻き込みうる。
- **`forEach` 内の `async`**：`arr.forEach(async ...)` は待たれない。`for...of` ＋ `await`、または `Promise.all(arr.map(...))`。
- **`try/catch` の握り潰し**：catch して何もしない/ログだけ → 失敗が静かに消える。
- **同期 I/O の常用**：`readFileSync` 等はリクエスト処理中に使わない。

```js
// プロセス全体の安全網（最後の砦。個別 handle が本筋）
process.on('unhandledRejection', (reason) => {
  console.error('unhandledRejection', reason)
  // ログ送信後、安全に終了 → プロセスマネージャ(pm2 等)で再起動
})
```

## 関連
[error_handling.md](./error_handling.md) / [middleware.md](./middleware.md)
