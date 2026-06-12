# Vue

## 一言で
**プログレッシブなUIフレームワーク**。テンプレート（HTML拡張）＋**リアクティビティ**で宣言的にUIを組む。React がライブラリ寄りなのに対し、Vue は**公式のルーター(Vue Router)・状態管理(Pinia)が揃った“フレームワーク寄り”**で、学習しやすいのが持ち味。`.vue`（単一ファイルコンポーネント）が基本単位。

## 特徴
- **SFC（単一ファイルコンポーネント `.vue`）**：`<template>`/`<script>`/`<style>` を1ファイルに。
- **リアクティビティ**：`ref` / `reactive` で「値を変えると画面が追従」。
- **Composition API（`<script setup>`）** が現在の標準（旧 Options API もある）。
- **ディレクティブ**：`v-if` / `v-for` / `v-model`（双方向バインド）/ `v-bind`(`:`) / `v-on`(`@`)。
- **公式エコシステム**が揃う（Vue Router / Pinia）。

## React との違い（ざっくり）
| | React | Vue |
|---|---|---|
| 形 | JSX（JSの中にUI） | テンプレート（HTML拡張） |
| 双方向 | 手動（value+onChange） | `v-model` で簡単 |
| 公式の周辺 | 別ライブラリを選ぶ | ルーター/状態管理が公式 |
| 学習 | 自由だが組み立てが要る | レール感があり入りやすい |

## このフォルダの構成
- [vue3/](./vue3/) … **Vue 3 実務リファレンス（フラッグシップ）**。SFC〜リアクティビティ〜Composition API〜Pinia〜ルーティング〜テスト〜罠まで、項目=1ファイル。

> 関連: Vueの上に乗るフルスタック版は [../nuxt/](../nuxt/)（SSR・ルーティング内蔵）。土台は [../javascript/](../javascript/) / [../typescript/](../typescript/)。
