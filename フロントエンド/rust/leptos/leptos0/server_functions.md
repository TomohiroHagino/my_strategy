# サーバ関数（`#[server]`）（Leptos）

> ⚠️ 0.x 系は API 変動が大きい。本書は **0.7 系**（`#[server]` / `ServerFnError`）前提。実装時は最新の公式ドキュメントで確認すること。

## ひとことで言うと
`#[server]` を付けた **async 関数は「サーバ側でだけ実行される処理」**。クライアント（WASM）からはその関数を **普通に `.await` で呼ぶ**だけで、Leptos が裏で HTTP リクエストに変換してサーバの本体を呼んでくれる。フルスタックの目玉。

## 役割・なぜ必要か
- ブラウザ上の WASM から **DB 接続・秘密鍵・ファイルアクセス**などを直接やるのは不可能／危険。これらは必ずサーバで実行したい。
- 従来は「サーバに API エンドポイントを作り、クライアントから `fetch` し、JSON を組み立て直す」という往復の配線が必要だった。`#[server]` は **その配線（ルーティング・シリアライズ・通信）を自動生成**する。
- 結果として **「関数を呼ぶだけ」でサーバ処理を実行**できる。型はそのまま Rust の関数シグネチャで共有されるので、API の手書き定義やズレが減る。
- `Resource` の fetcher として組み合わせると、「依存に応じてサーバから取り直す」が自然に書ける。→ [async_resources.md](./async_resources.md)

## 基本の書き方（コード）
```rust
use leptos::prelude::*;
use serde::{Serialize, Deserialize};

// 引数・戻り値は Serialize / Deserialize 可能であること（通信で運ばれるため）
#[derive(Clone, Serialize, Deserialize)]
struct User { id: u32, name: String }

#[server]
async fn get_user(id: u32) -> Result<User, ServerFnError> {
    // ここは「サーバ側だけ」で動く。DB アクセスや秘密の取得 OK。
    // クライアント用ビルドにはこの中身は含まれない。
    let pool = expect_context::<DbPool>(); // 例：サーバ側の context から取得
    let user = query_user(&pool, id).await
        .map_err(|e| ServerFnError::new(format!("db error: {e}")))?;
    Ok(user)
}

#[component]
fn Profile() -> impl IntoView {
    // クライアントからは「普通の async 関数」として呼ぶだけ
    let data = Resource::new(
        || 1u32,
        |id| async move { get_user(id).await },   // 内部で HTTP に変換される
    );

    view! {
        <Suspense fallback=move || view! { "読込中…" }>
            {move || data.get().map(|res| match res {
                Ok(user) => view! { <p>{user.name}</p> }.into_any(),
                Err(e)   => view! { <p>"失敗: " {e.to_string()}</p> }.into_any(),
            })}
        </Suspense>
    }
}
```

戻り値は基本 **`Result<T, ServerFnError>`**。`T` も引数も「通信で運べる型」である必要がある。

## 実務での使い方・定番パターン
- **`cargo-leptos` でビルド・実行**する。サーバ用バイナリとクライアント用 WASM を1つの crate から作り分け、`#[server]` の本体はサーバ側だけに含める仕組み。`feature`（例 `ssr`）で分岐する構成が定番。
- **`Resource` の fetcher にする**のが王道。データ取得＝server function、表示＝`Suspense`、という流れ。
- **フォーム送信（mutation）**にも使う。`<ActionForm action=…>` や `Action::new` と組み合わせ、「送信→サーバ処理→結果で再取得」を配線する。
- **サーバ側 context** を `expect_context`/`use_context` で受け、DB プールや認証情報を server function 内で取り出す。→ [context_state.md](./context_state.md)
- 失敗時は `ServerFnError::new(...)` などで包んで返し、クライアント側で `Err` を表示・リカバリする。

## ハマりどころ / アンチパターン
- **`#[server]` の中身を「クライアントでも動く」と思い込む。** 本体はサーバ専用。秘密・DB はここに置いてよいが、クライアント限定 API（`window` 等）は置けない。
- **引数／戻り値がシリアライズ不能。** `Serialize`/`Deserialize` を実装していない型、参照、ハンドル類は運べずビルドエラー／実行時エラーになる。素直な値型にする。
- **`ServerFnError` を握りつぶす。** `?` で素通しせず、文脈を付けて返さないと、クライアント側で何が起きたか分からなくなる。
- **CSR 単体（trunk のみ）で動かそうとする。** server function はサーバ前提。`cargo-leptos` ＋ SSR 構成が必要。描画モードの理解が前提。→ [ssr_csr.md](./ssr_csr.md)
- **`feature` 分岐ミス**で、サーバ専用クレート（DB ドライバ等）が WASM ビルドに混入してコンパイルが壊れる。`ssr` フィーチャの中だけで使う。

## 関連
[async_resources.md](./async_resources.md) / [ssr_csr.md](./ssr_csr.md) / [context_state.md](./context_state.md)
