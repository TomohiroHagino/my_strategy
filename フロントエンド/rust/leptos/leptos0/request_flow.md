# データの流れ・各部分は何を返すか（Leptos 0.x）

> ⚠️ Leptos の **CSR モードは純クライアント**（サーバの「層」は無い）。ここではまず **Signal(状態)→view の一方向データフロー** を書き、続けて **SSR モード**（HTTP→server function→HTML→hydrate）の流れも示す。CSR と SSR で図が分かれる点に注意。
> ⚠️ Leptos 0.x は API がよく変わる。実装時は最新の公式ドキュメントで確認すること。

## ひとことで言うと
- **CSR**：操作 → `Signal`(状態)変化 → `view!` をリアクティブに再評価 → DOM更新。流れは **状態 → UI の一方向**（細粒度：変わった部分だけ更新）。
- **SSR**：HTTP → server function → データ → コンポーネント → HTML → hydrate。

## 全体の流れ（図）
```
【CSR：純クライアント】一方向データフロー
   ユーザー操作（クリック等）
      │
      ▼
   [Signal 更新]  set_count(...)        受:新しい値 → 返:なし（依存に通知）
      │
      ▼
   [view! の該当部分を再評価]  Signal を読んでいた“その箇所だけ”更新
      │  受:Signalの値 → 返:DOMノード（差し替え）
      ▼
   [DOM 更新]（再レンダリング＆diff はしない＝細粒度）
      ▼
   画面
      │ 操作するとまた上へ
      ▲────────────────────────┘  一方向（Signal → view）

【SSR：フルスタック】
   ブラウザ
      │ HTTP リクエスト
      ▼
   [server function (#[server])]  サーバで実行（DB等）  受:引数 → 返:Result<データ>
      │
      ▼
   [コンポーネント]  データを受けて view! を組む        返:HTML
      │
      ▼
   [HTML 送信 → hydrate]  ブラウザでイベントを接続
      ▼
   画面（以降は CSR と同じ Signal 駆動）
```

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | モード |
|---|---|---|---|
| **Signal（`signal()`）** | 初期値 | `(getter, setter)`（読み/書き） | CSR/SSR共通 |
| **`view!`** | Signal等の値 | **DOMノード**（リアクティブに更新） | CSR/SSR共通 |
| **Effect / Memo** | 依存する Signal | 副作用実行 / 派生値 | CSR/SSR共通 |
| **Resource** | 取得元（async） | **非同期データ**（`Suspense`で待つ） | CSR/SSR |
| **server function（`#[server]`）** | 引数（シリアライズ可能） | **`Result<データ>`**（サーバ実行） | SSR |
| **コンポーネント（`#[component]`）** | props | view（→DOM/HTML） | CSR/SSR共通 |

- **CSRは細粒度**：Signal を読んでいる箇所だけが更新される（React のような全体再評価＆diff はしない）。
- **server function は呼ぶ側からは普通の async 関数に見える**が、実体はサーバで動きクライアントから RPC される。

## コードで通して見る
```rust
// 1) CSR：Signal → view の一方向。操作で set_count、view が追従
#[component]
fn Counter() -> impl IntoView {
    let (count, set_count) = signal(0);          // Signal：(getter, setter)
    view! {
        // count を読む箇所だけがリアクティブに更新される
        <button on:click=move |_| set_count.update(|n| *n += 1)>
            "count: " {move || count.get()}      // 返り＝DOMノード（count変化で更新）
        </button>
    }
}

// 2) SSR：server function（サーバ実行）→ Resource で取得 → view へ
#[server]
async fn get_user(id: u32) -> Result<String, ServerFnError> {
    // ここはサーバで動く（DBアクセス等）
    let user = db::find_user(id).await?;          // 受け＝引数 id
    Ok(user.name)                                 // 返り＝Result<データ>
}

#[component]
fn UserView(id: u32) -> impl IntoView {
    let user = Resource::new(move || id, |id| get_user(id)); // 非同期データ
    view! {
        <Suspense fallback=move || view! { "loading..." }>
            {move || user.get().map(|name| view! { <h1>{name}</h1> })}
        </Suspense>                               // 返り＝DOM（取得後に描画）
    }
}
```

## 実務での使い方・定番パターン
- **状態は Signal、派生は Memo**：状態から計算する値は `Memo` で（重複保持しない）。→ [signals.md](./signals.md) / [effects_memos.md](./effects_memos.md)
- **非同期取得は Resource ＋ Suspense**：ローディングを宣言的に扱う。→ [async_resources.md](./async_resources.md)
- **サーバ処理は `#[server]`**：DB等をクライアントから安全に呼ぶ（RPC）。→ [server_functions.md](./server_functions.md)
- **横断状態は context**：`provide_context`/`use_context` で配る。→ [context_state.md](./context_state.md)

## ハマりどころ / アンチパターン
- **`view!` 内で Signal を直読みして静的化**：`move || sig.get()` のクロージャで包まないと再評価されない。→ [view_macro.md](./view_macro.md)
- **server function に非シリアライズ引数/戻り値**：RPC境界を越えられない。型を見直す。→ [server_functions.md](./server_functions.md)
- **CSR/SSR の前提取り違え**：CSRに「層」は無い。SSRのみ server function がサーバ実行。→ [ssr_csr.md](./ssr_csr.md)
- **0.x の API 変化に追従しない**：`create_signal`→`signal()` 等。最新ドキュメント確認。→ [pitfalls.md](./pitfalls.md)

## 関連
[signals.md](./signals.md) / [view_macro.md](./view_macro.md) / [async_resources.md](./async_resources.md) / [server_functions.md](./server_functions.md) / [ssr_csr.md](./ssr_csr.md)
