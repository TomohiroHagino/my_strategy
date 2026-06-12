# リスト描画と key（React）

## ひとことで言うと
配列を `.map()` で **JSX の配列に変換して並べる**のがリスト描画。そのとき各要素には **`key`（兄弟の中で安定した一意のID）が必須**。`key` は React が「どの要素がどれか」を追跡するための目印で、画面には出ない。

## 役割・なぜ必要か
- 件数が動的（API取得・絞り込み・追加削除）なリストを、要素を手書きせず**データから機械的に**描画できる。
- React は再レンダリング時、**新旧のリストを `key` で突き合わせて差分だけ更新**する。`key` が安定していれば「3番目を消した」「先頭に1件足した」を正しく判断し、最小限のDOM操作で済む。
- `key` が無い／不安定だと突き合わせが狂い、**入力中の値が別行に移る・チェック状態がズレる・無駄な再生成が起きる**といったバグになる。

## 基本の書き方（コード）
```tsx
type Todo = { id: string; title: string; done: boolean };

function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map((todo) => (
        // keyは「map直下の要素」に、安定した一意のID（=todo.id）を渡す
        <li key={todo.id}>
          <input type="checkbox" defaultChecked={todo.done} />
          {todo.title}
        </li>
      ))}
    </ul>
  );
}

// 複数要素を返したいが余分なDOMを足したくない → Fragmentにkeyを付ける
import { Fragment } from "react";

function DefinitionList({ items }: { items: { id: string; term: string; desc: string }[] }) {
  return (
    <dl>
      {items.map((it) => (
        <Fragment key={it.id}>
          <dt>{it.term}</dt>
          <dd>{it.desc}</dd>
        </Fragment>
      ))}
    </dl>
  );
}

// 安定IDが無い静的データなら、内容由来の一意キーで代用してもよい
const FRUITS = ["apple", "banana", "cherry"];
function FruitList() {
  return (
    <ul>
      {FRUITS.map((name) => (
        <li key={name}>{name}</li> // 重複も並び替えも無いと分かっている場合
      ))}
    </ul>
  );
}
```

## 実務での使い方・定番パターン
- **DBの主キー（`id`）を `key` にする**のが王道。サーバから来たデータはほぼ常に一意IDを持つ。
- 新規作成でまだIDが無い行は、`crypto.randomUUID()` などで**生成時に一度だけIDを付け**、そのIDを使い続ける（毎レンダリングで作り直さない）。
- `key` は **`map` が返す一番外側の要素**に付ける。内側の子に付けても効果がない。
- 並び替え・フィルタ・ページング・楽観的更新を伴うリストでは、安定IDの有無が体感品質を左右する。→ [state.md](./state.md)
- 長いリストは `react-window` 等の仮想化を検討（描画コスト削減）。→ [performance.md](./performance.md)

## ハマりどころ / アンチパターン
- **`key={index}` にしない**。並び替え・先頭への挿入・途中削除でインデックスがズレ、React が別物を同一視する。結果、**入力欄の値・チェック状態・フォーカスが行間で入れ替わる**。`index` が許されるのは「並び替えも増減も絶対に無い静的リスト」だけ。
- **`key` 無し**は開発時に "Each child in a list should have a unique key prop." の警告が出る。放置すると上記の追跡バグの温床。
- **`key` を間違った階層に付ける**：`<>{items.map(i => <Row key={i.id} .../>)}</>` のように、`map` の直下の要素へ付ける。ラッパーの外側に付けても無意味。
- **`key` をコンポーネント内で props として読もうとしない**。`key` は React 専用で子には渡らない。値が欲しければ別 prop（`id={...}`）で渡す。
- **`Math.random()` を `key` に使わない**。毎レンダリングで変わるため、全要素が毎回作り直され（アンマウント→再マウント）、状態が消えて遅くなる。
- **`key` の一意性は「同じ親の兄弟内」でだけ必要**。別のリスト間で値が被っても問題ない。

## 関連: [components_jsx.md](./components_jsx.md) / [state.md](./state.md)
