# 実務でハマる罠まとめ（Pitfalls）（Vue 3）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、症状からの原因切り分けの入口として使う。

## 役割・なぜ必要か
- Vue 3 はリアクティビティ（`ref`/`reactive`/Proxy）の挙動が直感に反する場面が多く、「動かない／更新されない」の多くは決まったパターン。症状から該当箇所へ素早く飛ぶための索引。

## リアクティビティ
- **`ref` の `.value` 忘れ（script内）**：`<script>` 内では `count.value` でアクセス・代入。`count + 1` や `count = 1` は無効（テンプレート内は自動アンラップで `.value` 不要）。→ [reactivity.md](./reactivity.md)
- **`reactive` の分割代入でリアクティブ喪失**：`const { a } = reactive(obj)` で取り出すと**ただの値**になり追従しない。`toRefs` で分割するか `obj.a` で参照。→ [reactivity.md](./reactivity.md)
- **`reactive` をまるごと再代入**：`state = newObj` は参照が切れて反応しない。プロパティを更新するか `ref` を使い `state.value = newObj`。→ [reactivity.md](./reactivity.md)
- **配列／オブジェクトの置換は丸ごと**：要素の入れ替えは `arr.value = [...]` 等で。Vue 3 は index 代入も追従するが、置換の方が明快。→ [reactivity.md](./reactivity.md)

## テンプレート / ディレクティブ
- **`:key` に index を使う**：並び替え・挿入・削除で対応がずれ、状態や入力が混線。**安定したID**を `:key` に。→ [directives.md](./directives.md)
- **`v-if` と `v-for` の同一要素併用**：優先度と可読性の問題。外側を `<template v-for>` で回し内側で `v-if`、または `computed` で事前フィルタ。→ [directives.md](./directives.md)
- **`v-html` にユーザー入力**：XSSの危険。信頼できない値は描画しない／サニタイズする。→ [directives.md](./directives.md)

## computed / watch
- **`computed` で副作用**：API呼び出し・代入・DOM操作を入れない。`computed` は**純粋な算出専用**。副作用は `watch` / `watchEffect` へ。→ [computed_watch.md](./computed_watch.md)
- **`watch` の `deep` / `immediate`**：ネストしたオブジェクトの内部変更は既定で検知しない→ `deep: true`。初期値でも走らせたいなら `immediate: true`（`deep` はコスト高なので必要範囲に絞る）。→ [computed_watch.md](./computed_watch.md)
- **`watch` のソース指定ミス**：`ref` はそのまま、`reactive` のプロパティは `() => obj.x` のゲッターで渡す。値を直接渡すと追従しない。→ [computed_watch.md](./computed_watch.md)

## props / emit
- **props を直接変更**：`props` は読み取り専用。書き換えは警告＆親で上書きされる。ローカルに `ref`/`computed` で複製するか `emit` で親へ依頼。→ [props_emit.md](./props_emit.md)
- **`v-model` の規約ずれ**：子は `modelValue` を受け `update:modelValue` を emit。命名を外すと双方向が成立しない。→ [props_emit.md](./props_emit.md)

## 状態管理 / ルーティング
- **store の分割代入でリアクティブ喪失**：`const { count } = store` は切れる。**`storeToRefs(store)`** で state/getter を分割し、action はそのまま取り出す。→ [pinia.md](./pinia.md)
- **`<a href>` で内部遷移**：フルリロードが走りSPAの状態が消える。内部リンクは **`<RouterLink>`**、プログラム遷移は `router.push`。→ [routing.md](./routing.md)
- **ルート paramの変化で再取得漏れ**：同一コンポーネントの `/user/1`→`/user/2` は再生成されない。`watch(() => route.params.id, ...)` で再取得。→ [routing.md](./routing.md)

## テスト
- **DOM更新待ち忘れ**：`trigger` 後は非同期更新。`await nextTick()` / `await flushPromises()` で待ってから検証。→ [testing.md](./testing.md)

## 関連
[reactivity.md](./reactivity.md) / [directives.md](./directives.md) / [computed_watch.md](./computed_watch.md) / [props_emit.md](./props_emit.md) / [pinia.md](./pinia.md) / [routing.md](./routing.md) / [testing.md](./testing.md)
