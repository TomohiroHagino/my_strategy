# 通信（fetch / axios）（React Native）

## ひとことで言うと
サーバーのAPIと**HTTPでデータをやり取りする**仕組み。RNには標準の **`fetch`** が組み込まれており、`axios` などのライブラリも使える。Webと同じ感覚で書けるが、**実機からの`localhost`** と **平文HTTP通信** で必ずハマるので注意。

## 役割・なぜ必要か
- アプリ単体では完結しない。ログイン・一覧取得・投稿などは**バックエンドのAPI**を叩いて実現する。
- RNはブラウザではないが、`fetch` / `Promise` / `async-await` といったWeb標準APIをそのまま使える。だからWebのReactで通信を書いた経験がほぼ流用できる。
- レスポンスは多くがJSON。`res.json()` でパースしてReactの**state**へ入れ、画面に反映する流れが基本。→ [components_state.md](./components_state.md)

## 基本の書き方（コード）
```tsx
import { useEffect, useState } from 'react';
import { View, Text, ActivityIndicator } from 'react-native';

type User = { id: number; name: string };

// 標準の fetch（async/await）
async function fetchUsers(signal: AbortSignal): Promise<User[]> {
  const res = await fetch('https://api.example.com/users', { signal });
  if (!res.ok) {
    // fetch は 404/500 でも reject しない。ok を自分で見る（ハマりどころ）
    throw new Error(`HTTP ${res.status}`);
  }
  return res.json() as Promise<User[]>;
}

export function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();
    // タイムアウト（fetch 標準には無いので自前で）
    const timer = setTimeout(() => controller.abort(), 10_000);

    fetchUsers(controller.signal)
      .then(setUsers)
      .catch((e) => setError(String(e)))
      .finally(() => {
        clearTimeout(timer);
        setLoading(false);
      });

    return () => controller.abort(); // 画面を抜けたら中断
  }, []);

  if (loading) return <ActivityIndicator />;
  if (error) return <Text>エラー: {error}</Text>;
  return (
    <View>
      {users.map((u) => (
        <Text key={u.id}>{u.name}</Text>
      ))}
    </View>
  );
}
```

## 実務での使い方・定番パターン
- **axios** を使うと「JSON自動パース」「`baseURL`・共通ヘッダ」「interceptorでトークン付与」「タイムアウト1行」が楽。チームで通信が増えるなら導入価値あり。
```tsx
import axios from 'axios';
const api = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 10_000,
  headers: { 'Content-Type': 'application/json' },
});
api.interceptors.request.use((cfg) => {
  cfg.headers.Authorization = `Bearer ${getToken()}`; // 認証トークン注入
  return cfg;
});
const { data } = await api.get<User[]>('/users');
```
- **POST（JSON送信）** は `fetch` なら `method`・`headers`・`body: JSON.stringify(...)` をセット。axiosは第2引数にオブジェクトを渡すだけ。
- **ローディング / エラー / 成功** の3状態を必ずstateで持つ。RNはネットワークが不安定な実機で動くので、エラー表示・再試行は必須。
- データ取得が増えたら **TanStack Query（React Query）** でキャッシュ・再取得・ローディング管理を任せると定番。自前のuseEffect地獄を避けられる。

## ハマりどころ / アンチパターン
- **実機から `localhost` / `127.0.0.1` を叩く → 必ず失敗**。それは「スマホ自身」を指す。開発マシンのAPIへは次のどちらか:
  - **開発マシンのLAN IP** を使う（例 `http://192.168.1.10:3000`、同じWi-Fi必須）。
  - Androidなら `adb reverse tcp:3000 tcp:3000` で端末の`localhost:3000`をPCへ転送。
  - Androidエミュレータは特殊で、ホストPCは **`10.0.2.2`**。
- **平文HTTP（`http://`）がブロックされる**。本番は`https`が前提だが、開発で`http`を使うなら明示許可が要る:
  - **iOS**: `Info.plist` の **ATS（App Transport Security）** で例外設定。
  - **Android**: `AndroidManifest.xml` で **`android:usesCleartextTraffic="true"`**（Expoは `app.json` の設定経由）。
- **`fetch` は HTTPエラーで reject しない**。`res.ok` / `res.status` を自分で判定しないと、エラーJSONを正常データとして処理してしまう。
- **タイムアウト未設定**で画面が永久ローディング。`fetch`は`AbortController`、axiosは`timeout`で必ず上限を。
- **エラーの握り潰し**（`catch` で何もしない）。ユーザーにエラー表示と再試行手段を出す。
- **CORSはRNには無い**（ブラウザの概念）。Webで動かないからとCORS設定を疑うのは的外れ。落ちる原因は大抵URL（localhost）か平文HTTP。

## 関連: [components_state.md](./components_state.md)
