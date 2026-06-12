# データの流れ・各部分は何を返すか（Vue 3・純クライアント）

> ⚠️ Vue（単体）は純クライアント（ライブラリ）。**サーバの「層」は無い**。ここで書くのは **状態(ref/reactive)→UI の一方向データフロー** と、**データ取得フロー**（API を叩いて状態を更新する経路）。SSR込みは [../../nuxt/nuxt3/](../../nuxt/nuxt3/) を参照。

## ひとことで言うと
**初期表示/操作 → コンポーネント（ref/reactive）→ 取得（onMounted/composable → fetch → 状態更新）→ 再描画**。状態(ref)が変わると、それを使うテンプレートが自動で再描画される。流れは常に **状態 → UI の一方向**。

## 全体の流れ（図）
```
初期表示 / ユーザー操作（クリック等）
   │
   ▼
[コンポーネント setup]  ref/reactive を読んでテンプレートを組む  受:props/状態 → 返:VNode
   │
   ├─ データが要るとき ───────────────────────────┐
   │   ▼                                          │
   │ [取得]  onMounted or composable（useXxx）
   │   │  ▼
   │   │ [fetch(API)]  外部APIを叩く     受:URL → 返:データ(JSON)
   │   │  ▼
   │   │ state.value = data  ← 取得結果で ref を更新
   │   ▼
   ▼
[再描画]  リアクティビティが依存を追跡し、その部分だけ更新
   │  受:新しい状態 → 返:新しいVNode
   ▼
[DOM 更新]  仮想DOM差分を実DOMへ反映
   ▼
画面（ユーザーが見る）
   │ 操作するとまた上へ（イベント → 状態変更 → 再描画）
   ▲────────────────────────────────────────────┘
   一方向（状態 → UI）。DOMを手で触らず状態を変える。
```

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 備考 |
|---|---|---|---|
| **コンポーネント（setup）** | `props` / 自分の状態 | **VNode**（→DOM） | テンプレート＝状態の関数 |
| **ref / reactive** | 初期値 | リアクティブな値 | `.value` 代入で再描画トリガ |
| **onMounted** | コールバック | なし（マウント後に実行） | 取得・購読の起点 |
| **composable（useXxx）** | 引数 | `ref`/`computed` 等の状態 | ロジック再利用の単位 |
| **fetch(API)** | URL / options | **データ(JSON)** | 結果を `ref.value` に代入 |
| **Pinia ストア** | — | `state`/`getters`/`actions` | アプリ横断の共有状態 |

- **状態(ref)が変わると、それに依存するテンプレート部分だけ**が自動で再描画される。
- **データは props で下へ、変更は emit で上へ**。子が親の DOM を直接いじることはしない。

## コードで通して見る
```vue
<!-- 1) コンポーネント：状態を読んでテンプレートを返す／onMountedで取得 -->
<script setup lang="ts">
import { ref, onMounted } from "vue";

const props = defineProps<{ id: string }>();   // props（親→子）
const user = ref<{ name: string } | null>(null); // 状態（ref）
const loading = ref(true);

// 2) 取得：onMounted → fetch → ref 更新
onMounted(async () => {
  const res = await fetch(`https://api.example.com/users/${props.id}`); // 受:URL
  user.value = await res.json();   // 返:データ(JSON) を ref に代入 → 再描画
  loading.value = false;
});
</script>

<template>
  <p v-if="loading">loading...</p>
  <h1 v-else>{{ user.name }}</h1>  <!-- 状態を使って描画 → DOM -->
</template>
```
```ts
// 別案：composable に取得ロジックを切り出す（再利用）
export function useUser(id: string) {
  const user = ref<{ name: string } | null>(null);
  onMounted(async () => {
    user.value = await fetch(`/api/users/${id}`).then((r) => r.json());
  });
  return { user };   // 返り＝状態（ref）をコンポーネントへ
}
```

## 実務での使い方・定番パターン
- **取得は `onMounted` or composable**：ロジックを `useXxx` に切り出して再利用。→ [composition_api.md](./composition_api.md)
- **横断状態は Pinia**：複数コンポーネントで共有する状態はストアへ。→ [pinia.md](./pinia.md)
- **親→子は props、子→親は emit**：`defineProps`/`defineEmits`。→ [props_emit.md](./props_emit.md)
- **派生値は `computed`**：状態から計算して持つ（重複保持しない）。→ [computed_watch.md](./computed_watch.md)

## ハマりどころ / アンチパターン
- **`ref` の `.value` 付け忘れ**：script内では `.value` 必須（templateでは不要）。→ [reactivity.md](./reactivity.md)
- **DOMを手で操作**：Vueの管理と衝突。状態を変える。`ref`（テンプレート参照）は最小限。
- **reactive をデストラクチャしてリアクティビティ喪失**：`toRefs` を使う。→ [pitfalls.md](./pitfalls.md)
- **取得を `setup` 同期で書き SSR/初期描画を阻害**：副作用は `onMounted` 等の適切なフックで。

## 関連
[reactivity.md](./reactivity.md) / [composition_api.md](./composition_api.md) / [lifecycle.md](./lifecycle.md) / [props_emit.md](./props_emit.md) / [pinia.md](./pinia.md)
