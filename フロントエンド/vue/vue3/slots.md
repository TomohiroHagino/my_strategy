# スロット（Slots）（Vue 3）

## ひとことで言うと
子コンポーネントの中に「**穴**」を開けておき、**親側からその穴へ中身（テンプレート）を差し込む**仕組み。`<slot/>` が穴、親が `<Child>...</Child>` の中身を埋める。

## 役割・なぜ必要か
- props は「データ」を渡すのに対し、スロットは「**マークアップ（見た目の塊）**」を渡す。レイアウトの外枠だけ子に持たせ、中身は使う側が自由に決められる。
- カード・モーダル・リスト・レイアウトなど「**枠は共通・中身は可変**」のコンポーネントを再利用可能にする。
- 名前付きスロットで複数の差込口（ヘッダー/本文/フッター）を用意でき、スコープ付きスロットで**子の内部状態を親のテンプレートへ渡せる**。→ [components.md](./components.md)

## 基本の書き方（コード）
```vue
<!-- Card.vue（子）-->
<template>
  <section class="card">
    <header class="card__head">
      <!-- 名前付きスロット（差込が無ければフォールバックを表示）-->
      <slot name="header">無題</slot>
    </header>
    <div class="card__body">
      <!-- デフォルトスロット -->
      <slot />
    </div>
  </section>
</template>
```

```vue
<!-- 親 -->
<template>
  <Card>
    <!-- #header は v-slot:header の省略記法 -->
    <template #header>
      <h2>ユーザー情報</h2>
    </template>

    <!-- 名前なしの中身はデフォルトスロットへ -->
    <p>本文がここに入ります</p>
  </Card>
</template>
```

```vue
<!-- スコープ付きスロット：子の状態を親へ渡す -->
<!-- UserList.vue（子）-->
<template>
  <ul>
    <li v-for="u in users" :key="u.id">
      <!-- :user で各行のデータを親へ公開 -->
      <slot :user="u" :index="u.id" />
    </li>
  </ul>
</template>

<!-- 親：#default="{ ... }" で子が公開した値を受け取る -->
<template>
  <UserList :users="users">
    <template #default="{ user }">
      <strong>{{ user.name }}</strong>（{{ user.email }}）
    </template>
  </UserList>
</template>
```

## 実務での使い方・定番パターン
- **レイアウトの外枠化**：`Card` / `Modal` / `Panel` のように枠だけ提供し、中身は呼び出し側に委ねる。
- **名前付きで複数差込口**：`#header` / `#footer` / デフォルト の3点セットが定番。差込が無いときの**フォールバック内容**を `<slot>ここがデフォルト</slot>` で書いておくと親が省略できる。
- **スコープ付きスロットで「描画の自由」を渡す**：データ取得・ループは子が担い、**1件の見た目だけ親が決める**（テーブル行・リスト項目）。ヘッドレスUI（ロジックだけ提供し見た目は親任せ）の基本テク。
- **デフォルトスロットの省略記法**：差込が中身だけならテンプレート不要で `<Child>中身</Child>` と書ける。スコープを受けたいときだけ `<template #default="slotProps">` を使う。
- **スロットの存在判定**：`<script setup>` 内で `useSlots()` を使い、`slots.header` の有無で枠の表示を出し分けできる。

## ハマりどころ / アンチパターン
- **スコープ付きスロットの記法を取り違える**（最頻出）：子は `<slot :user="u" />` のように**バインドで公開**、親は `#default="{ user }"` のように**分割代入で受け取る**。子で `:user`、親で `{ user }` の**名前を一致**させる必要がある。`v-bind` を忘れて `user="u"` と書くと文字列リテラルになる。
- **フォールバック内容の誤解**：`<slot>デフォルト</slot>` の中身は「親が**何も差し込まなかった時だけ**」表示される。親が空文字を差し込めばフォールバックは出ない。
- **名前付きスロットの差込先ミス**：親で `<template #header>` を付け忘れると、その中身は**デフォルトスロット**へ流れる。名前付き枠が空のまま見える。
- **スコープ付きスロットは親で `#default` が必須**：単に `<UserList />` の中に書くだけでは子のスコープ値を受け取れない。スコープを使うなら `<template #default="...">` で囲う。
- **過剰なスロット化**：何でもスロットにすると props との境界が曖昧になり読みにくい。**データは props、マークアップはスロット**の原則を守る。→ [components.md](./components.md)

## 関連
[components.md](./components.md) / [directives.md](./directives.md) / [props_emit.md](./props_emit.md)
