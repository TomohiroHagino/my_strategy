# データ取得（Data Fetching）（React）

## ひとことで言うと
サーバからデータを取ってきて画面に出すこと。React 本体に取得機能は無いので、`fetch` を**いつ・どう呼ぶか**は自分で設計する。素朴には `useEffect` 内で `fetch` するが、実務では **TanStack Query（React Query）** や **SWR** といった「サーバ状態」専用ライブラリに任せるのが定番。

## 役割・なぜ必要か
- API から取った値は **サーバ状態**（自分が真の持ち主ではないデータ）。クライアント状態（`useState`）とは性質が違う。
- サーバ状態には付き物の関心事が多い：**loading / error / キャッシュ / 再取得 / 競合（race）/ 重複排除 / 再試行 / 無効化**。これを全部自前で書くと `useEffect` が肥大化する。
- ライブラリはこれらを宣言的に肩代わりしてくれる。「取得ロジック」ではなく「何が欲しいか」を書けばよくなる。

## 基本の書き方（コード）
まず素朴版（`useEffect` 直書き）。動くが、考慮漏れが起きやすい。
```tsx
// 素朴版：loading/error/race を全部手書きする羽目になる
function UserCard({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false; // ← race対策。これを忘れると古い結果で上書きされる
    setLoading(true);
    fetch(`/api/users/${id}`)
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json() as Promise<User>;
      })
      .then((data) => {
        if (!cancelled) setUser(data);
      })
      .catch((e) => {
        if (!cancelled) setError(e);
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });
    return () => {
      cancelled = true; // クリーンアップで「古いリクエスト」を無効化
    };
  }, [id]);

  if (loading) return <p>読み込み中…</p>;
  if (error) return <p>エラー: {error.message}</p>;
  return <p>{user?.name}</p>;
}
```

TanStack Query 版。上の関心事をライブラリが面倒を見る。
```tsx
import { useQuery } from "@tanstack/react-query";

const fetchUser = async (id: string): Promise<User> => {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
};

function UserCard({ id }: { id: string }) {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ["user", id], // キャッシュのキー。idごとにキャッシュ・自動で競合解決
    queryFn: () => fetchUser(id),
  });

  if (isPending) return <p>読み込み中…</p>;
  if (isError) return <p>エラー: {error.message}</p>;
  return <p>{data.name}</p>;
}
```

SWR 版（より軽量・最小API）。
```tsx
import useSWR from "swr";

const fetcher = (url: string) => fetch(url).then((r) => r.json());

function UserCard({ id }: { id: string }) {
  const { data, error, isLoading } = useSWR<User>(`/api/users/${id}`, fetcher);
  if (isLoading) return <p>読み込み中…</p>;
  if (error) return <p>エラー</p>;
  return <p>{data?.name}</p>;
}
```

## 実務での使い方・定番パターン
- **アプリ全体を Provider で包む**：TanStack Query は `QueryClientProvider`、SWR は `SWRConfig` で `queryClient` / 共通設定を渡す。
- **queryKey 設計**：`["todos", { status, page }]` のように依存値をキーに含める。キーが変われば自動で再取得＆別キャッシュ。
- **mutation（更新系）**：`useMutation` で POST/PUT/DELETE → 成功後に `queryClient.invalidateQueries` で関連キャッシュを無効化して再取得。楽観的更新（optimistic update）も `onMutate`/`onError` のロールバックで実装。
- **Suspense 連携（React 18/19）**：`useSuspenseQuery` を使うと `data` は常に存在扱いになり、`isPending` 分岐が消える。`<Suspense fallback>` と `<ErrorBoundary>` でローディング/エラーを宣言的に括る。
```tsx
<ErrorBoundary fallback={<p>失敗しました</p>}>
  <Suspense fallback={<p>読み込み中…</p>}>
    <UserCard id={id} /> {/* 中で useSuspenseQuery を使う */}
  </Suspense>
</ErrorBoundary>
```
- **再取得制御**：`staleTime`（鮮度）/ `gcTime`（キャッシュ保持）/ `refetchOnWindowFocus` を調整。リアルタイム性が要らない画面は `staleTime` を長めに。
- React 19 の `use` フックで Promise を直接 `Suspense` に渡す手もあるが、キャッシュ・無効化まで考えると専用ライブラリが依然強い。

## ハマりどころ / アンチパターン
- **`useEffect` 直書きの race / 重複**：`id` が高速に変わると古いリクエストが後着して新しい結果を上書き。クリーンアップでの無効化（`cancelled` フラグや `AbortController`）が必須。StrictMode では effect が二重実行され、重複 fetch にも気づきやすい。
- **サーバ状態を `useState` に二重保持**：`useQuery` の `data` を `useEffect` で `setState` にコピーすると、**真実の出どころが二重化**してズレる。派生値は `data` から直接計算する（メモ化したいなら `useMemo`）。
- **loading/error を毎回手書き**：DRY 違反でバグの温床。ライブラリの状態フラグに寄せる。
- **queryKey の入れ忘れ**：依存値をキーに含めないと、引数が変わっても古いキャッシュが返り続ける。
- **過剰なグローバル client store へのコピー**：Zustand/Redux にサーバ応答を丸ごと入れると、キャッシュ・無効化を自前で再実装する羽目に。サーバ状態は Query/SWR、UI状態だけ client store、と役割を分ける。
- **エラーの握り潰し**：`.catch` で握って何も出さないと無言で壊れる。UI にエラー表示、サーバ側にログ。

## 関連
[useeffect.md](./useeffect.md) / [state.md](./state.md) / [testing.md](./testing.md)
