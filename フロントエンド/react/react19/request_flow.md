# データの流れ・各部分は何を返すか（React 19・純クライアント）

> ⚠️ React は純クライアント（ライブラリ）。**サーバの「層」は無い**。ここで書くのは **状態(state)→UI の一方向データフロー** と、**データ取得フロー**（API を叩いて state を更新する経路）。「リクエストが層を降りる」図ではない点に注意。

## ひとことで言うと
**初期表示/ユーザー操作 → コンポーネント（state/props）→ データ取得（useEffect or TanStack Query → fetch → state更新）→ 再レンダリング → DOM更新**。state が単一の源で、UI は state の関数（`UI = f(state)`）。流れは常に **state → UI の一方向**。

## 全体の流れ（図）
```
初期表示 / ユーザー操作（クリック等）
   │
   ▼
[コンポーネント]  state / props を読んで JSX を組む   受:props/state → 返:JSX
   │
   ├─ データが要るとき ───────────────────────────┐
   │   ▼                                          │
   │ [データ取得]  useEffect or TanStack Query
   │   │  ▼
   │   │ [fetch(API)]  外部APIを叩く     受:URL → 返:データ(JSON)
   │   │  ▼
   │   │ setState(data)  ← 取得結果で state を更新
   │   ▼
   ▼
[再レンダリング]  state が変わった部分を React が再評価
   │  受:新しいstate → 返:新しいJSX
   ▼
[DOM 更新]  仮想DOM差分を実DOMへ反映
   ▼
画面（ユーザーが見る）
   │ 操作するとまた上へ（イベント → setState → 再レンダリング）
   ▲────────────────────────────────────────────┘
   一方向（state → UI）。DOMを手で触らず state を変える。
```

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 備考 |
|---|---|---|---|
| **コンポーネント** | `props`（親→子）/ 自分の `state` | **JSX**（→DOM） | `UI = f(state)` |
| **useState** | 初期値 | `[値, setter]` | setter 呼び出しで再レンダリング |
| **useEffect** | 依存配列 + 副作用関数 | なし（副作用を実行） | マウント/依存変化時に走る |
| **fetch(API)** | URL / options | **データ(JSON)** | 結果を `setState` に渡す |
| **TanStack Query** | queryKey / queryFn | `{ data, isLoading, error }` | キャッシュ・再取得を内蔵 |
| **イベントハンドラ** | イベント | なし（`setState` を呼ぶ） | 操作 → state変更の起点 |

- **state が変わると、それを使うコンポーネントだけ再レンダリングされ、DOMに反映**される。
- **データは「上から下」（props）／更新は「state経由」**。親の DOM を子が直接いじることはしない。

## コードで通して見る
```tsx
// 1) コンポーネント：state/props を読んで JSX を返す
function UserView({ id }: { id: string }) {       // props（親→子）
  const [user, setUser] = useState<{ name: string } | null>(null); // state
  const [loading, setLoading] = useState(true);

  // 2) データ取得：useEffect → fetch → setState
  useEffect(() => {
    let active = true;
    fetch(`https://api.example.com/users/${id}`)   // 受:URL
      .then((res) => res.json())                   // 返:データ(JSON)
      .then((data) => {
        if (active) { setUser(data); setLoading(false); } // state更新 → 再レンダリング
      });
    return () => { active = false; };              // クリーンアップ
  }, [id]);                                        // id が変われば再取得

  if (loading) return <p>loading...</p>;
  return <h1>{user!.name}</h1>;                    // 返り＝JSX（→DOM）
}

// 推奨：取得は TanStack Query に任せると簡潔（キャッシュ・再取得つき）
function UserView2({ id }: { id: string }) {
  const { data, isLoading } = useQuery({
    queryKey: ["user", id],
    queryFn: () => fetch(`/api/users/${id}`).then((r) => r.json()),
  });                                              // 返り＝{ data, isLoading, error }
  if (isLoading) return <p>loading...</p>;
  return <h1>{data.name}</h1>;
}
```

## 実務での使い方・定番パターン
- **取得は基本 TanStack Query / SWR**：`useEffect`手書きより、ローディング/エラー/キャッシュ/再取得を内蔵で扱える。→ [data_fetching.md](./data_fetching.md)
- **state は単一の源**：派生値は計算で出す（重複して持たない）。→ [state.md](./state.md)
- **親→子は props、子→親は コールバック**：データは下、変更通知は上へ。→ [props.md](./props.md)
- **副作用は `useEffect`、依存配列を正しく**：取得・購読・タイマー等。→ [useeffect.md](./useeffect.md)

## ハマりどころ / アンチパターン
- **DOMを手で操作（`document.querySelector` で書き換え）**：Reactの管理と衝突。state を変える。
- **`useEffect` の依存漏れ**：古い値で動く/無限ループ。依存配列を正しく列挙。→ [useeffect.md](./useeffect.md)
- **サーバ状態をローカルstateに二重化**：取得データは Query のキャッシュに任せ、複製しない。
- **取得を `useEffect` で手書きし続ける**：競合・キャンセル漏れ。ライブラリを使う。→ [pitfalls.md](./pitfalls.md)

## 関連
[state.md](./state.md) / [props.md](./props.md) / [useeffect.md](./useeffect.md) / [data_fetching.md](./data_fetching.md) / [hooks.md](./hooks.md)
