# データの流れ・各部分は何を返すか（React Native）

## ひとことで言うと
モバイルは「サーバのリクエスト層」ではなく、**状態→UIの一方向データフロー**で考える。**ユーザー操作 → state が変わる → コンポーネントが再レンダリングされる → ネイティブ UI が更新される**。データ取得（fetch）は state を更新する手段で、最終的にすべて state 経由で画面に反映される。

## 全体の流れ（図）
```
ユーザー操作（onPress / onChangeText）
   │
   ▼
[state 更新]   setCount(...) / dispatch(action)（Redux）
   │           ※ 直接 UI をいじらない。state を変えるだけ
   ▼
[再レンダリング]  state を読むコンポーネントの関数が再実行
   │
   ▼
[ネイティブ UI 更新]  返した要素ツリーを RN がネイティブビューに反映
   │
   ▼
  画面（描画）

── データ取得フロー（横入り）──────────────
[API]  fetch / axios で取得
   │ データ（JSON）を返す
   ▼
[state 更新]  setData(json)（useEffect 内で取得 → set）
   │
   ▼  以降は上と同じ：再レンダリング → ネイティブ UI 更新
```
一方向（状態 → UI）。UI から state への戻り矢印は無く、操作は「次の state」を作るだけ。

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 役割 |
|---|---|---|---|
| **ユーザー操作** | onPress / onChangeText | ハンドラ呼び出し | 状態変更のきっかけ |
| **API** | URL / params | **データ（JSON）**（fetch の Promise） | 外部からデータ取得 |
| **state** | 操作・取得結果 | **再レンダリング時の現在値** | 画面の元になる唯一の真実 |
| **コンポーネント** | props + state | **要素ツリー（JSX）** | 状態から UI を組み立てる |
| **ネイティブ UI** | — | 描画される View/Text 等 | 運ばれる側 |

- **state が「真実」**：画面は state の写像。`setState` 系で state を変えれば UI が追従する。
- **コンポーネントは状態の関数**：同じ state なら同じ UI を返す。

## コードで通して見る
```tsx
// パターンA：useState ＋ 操作
function Counter() {
  const [count, setCount] = useState(0);          // state（唯一の真実）

  return (
    <View>
      <Text>{count}</Text>                         {/* state → UI */}
      <Button title="+1" onPress={() => setCount(c => c + 1)} /> {/* 操作 → state 更新 */}
    </View>
  );
}
```

```tsx
// パターンB：fetch(API) → state（データ取得フロー）
function UserView() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('https://api.example.com/user')          // API → データ（JSON）を返す
      .then(res => res.json())
      .then((json: User) => { setUser(json); setLoading(false); }); // state 更新
  }, []);

  if (loading) return <ActivityIndicator />;
  return <Text>{user?.name}</Text>;                 // state → ネイティブ UI
}
```

## 実務での使い方・定番パターン
- **ローカル state は useState、共有 state は Redux/Zustand**：画面をまたぐ状態はストアへ。→ [storage_state.md](./storage_state.md)
- **取得は useEffect 内で fetch → set**：loading/error も state として持ち、UI を分岐させる。→ [networking.md](./networking.md)
- **直接 UI を操作しない**：常に「state を変える」→ RN が再レンダリング。命令的なビュー更新の発想を持ち込まない。

## ハマりどころ / アンチパターン
- **レンダー本体で fetch**：useEffect を使わず関数本体で fetch すると毎回走り無限ループ。
- **state を直接書き換える**：`user.name = ...` では再レンダリングされない。必ず `setState` 系で新しい値を渡す（不変更新）。
- **`<View>` 直下にテキストを置く**：文字は必ず `<Text>` の中に。`<div>` 感覚で置くと落ちる。
- **依存配列の付け忘れ**：useEffect の deps を誤ると取得が走りすぎ／走らなさすぎになる。

## 関連
[components_state.md](./components_state.md) / [networking.md](./networking.md) / [storage_state.md](./storage_state.md) / [core_components.md](./core_components.md)
