# 始め方 / ディレクトリ構成（Nuxt 3）

## ひとことで言うと
Nuxt 3 プロジェクトを `nuxi` で作り、**決められたディレクトリ名（規約）にファイルを置くだけ**で、ルーティング・自動インポート・サーバAPI などが有効になる仕組み。設定より規約（Convention over Configuration）。

## 役割・なぜ必要か
- Vue 単体だと、ルータ・SSR・ビルド・データ取得を自前で組む必要がある。Nuxt は **それらを最初から束ねた土台** を提供する。
- ディレクトリ名そのものが機能のスイッチ。`pages/` を置けばルーティング、`components/` を置けば自動インポート、`server/` を置けばAPI、という具合に **「どこに置くか」で意味が決まる**。
- だから最初に覚えるべきは API ではなく **規約（どのフォルダが何を担うか）**。これを外すと「動かない」ではなく「そもそも認識されない」になる。

## 基本の書き方（コード）
```bash
# 1. プロジェクト作成（app という名前で）
npx nuxi@latest init app
cd app

# 2. 依存インストール（init 時に選択可。後からでも可）
npm install

# 3. 開発サーバ起動（既定 http://localhost:3000）
npm run dev

# 4. 本番ビルド / プレビュー
npm run build      # .output/ を生成
npm run preview    # ビルド結果をローカル確認
```

```bash
# 主要ディレクトリ規約（すべて任意。置けば有効になる）
app/
├─ app.vue              # ルートコンポーネント（最初の入口）
├─ nuxt.config.ts       # Nuxt 全体の設定
├─ pages/               # ファイルベースルーティング（置くと有効）
│   └─ index.vue        #   → "/"
├─ components/          # 自動インポートされるコンポーネント
├─ composables/         # 自動インポートされる use 関数
├─ layouts/             # 共通レイアウト（default.vue など）
├─ middleware/          # ルートミドルウェア（遷移前の処理）
├─ server/              # Nitro サーバ（server/api/ で API）
├─ public/              # そのまま配信する静的ファイル
└─ assets/              # ビルド対象の CSS / 画像など
```

## 実務での使い方・定番パターン
- **まず `app.vue` だけで動く**。Vue の最小アプリと同じ感覚で書ける。ここに `<NuxtPage />` を置いた瞬間から `pages/` が効く。
- **ページを増やしたくなったら `pages/` を作る**。`app.vue` は「全ページ共通の外枠（ヘッダ・フッタ・`<NuxtPage />`）」に役割が変わる。
- `nuxt.config.ts` は **最初は空に近くてよい**。modules（`@nuxt/image` 等）や `runtimeConfig` を足す場所。→ [config.md](./config.md)
- `npm run dev` は **ホットリロード**付き。`.nuxt/`（自動生成）は触らない・コミットしない（`.gitignore` 済み）。
- TypeScript は標準同梱。`nuxi typecheck` で型チェックできる（`npx nuxi typecheck`）。

```vue
<!-- app.vue：pages/ を使う最小形 -->
<template>
  <div>
    <header>共通ヘッダ</header>
    <NuxtPage />   <!-- ここに pages/ の各ページが描画される -->
  </div>
</template>
```

```vue
<!-- app.vue：pages/ を使わない最小形（1画面アプリ） -->
<template>
  <main>
    <h1>Hello Nuxt 3</h1>
  </main>
</template>
```

## ハマりどころ / アンチパターン
- **規約のフォルダ名は固定**。`page/`（単数）や `Components/`（大文字）では認識されない。複数形・小文字を守る。
- **`pages/` が無いとルーティングは無効**。その場合 `app.vue` が唯一の画面。URL を分けたいのに `pages/` を作っていない、が頻出。→ [pages_routing.md](./pages_routing.md)
- **`app.vue` に `<NuxtPage />` を書き忘れる**と、`pages/` を作っても表示されない（`app.vue` の中身だけが出る）。`pages/` を使うなら必ず置く。
- **`app.vue` を消す**と Nuxt が用意する既定の入口が使われる。意図せず消して「レイアウトが消えた」となりやすい。
- **`.nuxt/` を手で編集／コミット**しない。型がおかしい時はここを再生成して直す（後述の自動インポート項参照）。
- `public/` と `assets/` の混同：**URL でそのまま欲しいなら `public/`**、ビルドで最適化したいなら `assets/`。

## フォルダ構成（始動直後）
```
myapp/
├── app.vue                 # ルートコンポーネント（最初の入口）
├── nuxt.config.ts          # Nuxt 全体の設定
├── package.json            # scripts: dev / build / preview
├── tsconfig.json           # TS設定（.nuxt/ の生成型を参照）
├── README.md               # 雛形の説明
├── .gitignore              # .nuxt/ や node_modules 等を除外
├── server/                 # Nitro サーバ
│   └── tsconfig.json       # サーバ側の TS 設定
├── public/                 # そのまま配信する静的ファイル
│   ├── favicon.ico         # ファビコン
│   └── robots.txt          # クローラ設定
├── pages/                  # ファイルベースルーティング（# 自分で作る）
├── components/             # 自動インポートされる部品（# 自分で作る）
├── layouts/                # 共通レイアウト（# 自分で作る）
├── composables/            # 自動インポートの use 関数（# 自分で作る）
├── server/api/             # サーバAPI（# 自分で作る）
├── .nuxt/                  # 自動生成（触らない・コミットしない）
└── node_modules/           # 依存パッケージ（npm install で生成）
```
- `pages/` を作るとファイルベースルーティングが有効に。`server/api/` でバックエンドも書ける。

## 関連
[pages_routing.md](./pages_routing.md) / [config.md](./config.md) / [components_autoimport.md](./components_autoimport.md)
