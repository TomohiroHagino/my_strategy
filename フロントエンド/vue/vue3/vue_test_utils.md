# Vue Test Utils（Vue 3）

## ひとことで言うと
Vue 3 コンポーネントを **マウントして操作・検証する公式テストライブラリ**。`mount` / `shallowMount` で描画し、`find` で要素を掴み、`trigger` でイベントを発火し、`emitted` で発火イベントを検証する。props を渡して入力、emit と表示で出力を見る＝コンポーネントの「契約」をテストする。

## 役割・なぜ必要か
- リアクティビティ・props・emit・条件分岐が絡むと手で全状態を確認できない。回帰を自動検出する。
- 「この props でこう描画」「このクリックでこのイベントが飛ぶ」という **実行可能な仕様書** になり、リファクタの安全網になる。
- 公式ライブラリなので Vue の再描画タイミング（`nextTick`）や SFC の扱いに最適化されている。
- ランナー（Vitest）とは別レイヤー。Vue Test Utils は「マウント・操作・検証」を担当する。

## 基本の書き方（コード）
`mount`（描画）→ `find`（探す）→ `trigger`（操作）→ `expect`（検証）。
```ts
// Counter.spec.ts
import { mount } from "@vue/test-utils";
import { describe, it, expect } from "vitest";
import Counter from "./Counter.vue";

describe("Counter", () => {
  it("ボタンクリックで表示が増える", async () => {
    const wrapper = mount(Counter, { props: { start: 0 } });

    await wrapper.find("button.increment").trigger("click"); // DOM更新を await

    expect(wrapper.find(".count").text()).toBe("1");
  });

  it("初期値 props を表示する", () => {
    const wrapper = mount(Counter, { props: { start: 5 } });
    expect(wrapper.text()).toContain("5");
  });

  it("increment イベントを emit する", async () => {
    const wrapper = mount(Counter);
    await wrapper.find("button.increment").trigger("click");
    expect(wrapper.emitted("increment")).toHaveLength(1); // emit を検証
  });
});
```

## 実務での使い方・定番パターン
- **`mount` と `shallowMount` を使い分け**：`mount` は子まで実描画して結合を見る／`shallowMount` は子をスタブ化して対象だけ隔離（重い子・外部依存を切る）。
- **props で入力、`emitted()` で出力を検証**：コンポーネントの入出力（契約）をテストするのが基本。→ [props_emit.md](./props_emit.md)
- **非同期更新は待つ**：`onMounted` の `fetch` などは `flushPromises` で解決、DOM 反映は `nextTick` で待つ。
```ts
import { mount, flushPromises } from "@vue/test-utils";

it("非同期取得後にリストを描画", async () => {
  const wrapper = mount(UserList);
  await flushPromises(); // 保留中のPromise（fetch等）を解決
  expect(wrapper.findAll("li")).toHaveLength(3);
});
```
- **依存の差し替え**：`global.plugins`（Pinia / Router）や `global.stubs`（子コンポーネント）で環境を注入する。
- **外部APIはモック**：`vi.mock` / MSW で境界を差し替え、内部ロジックは本物を動かす。
- **composables は単体テスト**：UI を介さず関数として `ref`/`computed` の挙動だけ検証すると速くて安定。→ [composition_api.md](./composition_api.md)

## ハマりどころ
- **DOM更新待ち忘れ**：Vue の再描画は非同期。`trigger` や状態変更の直後に検証すると古い DOM を見て落ちる。`await ...trigger(...)` か `await nextTick()` を挟む。
- **非同期処理の取りこぼし**：`fetch` を `flushPromises` で待たないと「まだ空のリスト」を検証してしまう。
- **`shallowMount` の過信**：子を全スタブ化すると親子連携の不具合を見逃す。重要な結合は `mount` で実描画する。
- **`wrapper.vm` 直叩き**：内部状態を直接いじる／読むのはユーザー操作と乖離。`find`＋`trigger`＋表示で検証する。
- **実装詳細テスト**：メソッド名・data 構造・クラス名に依存すると、リファクタで赤くなる「もろい」テストに。入出力（props/emit/表示）を検証する。
- **テスト間の状態リーク**：Pinia・タイマー・モジュールキャッシュが残り順序依存で落ちる。`beforeEach` でストア初期化、`vi.restoreAllMocks()` で後始末。

## 関連
[testing.md](./testing.md)（全体方針・ランナー） / [playwright.md](./playwright.md) / [props_emit.md](./props_emit.md) / [composition_api.md](./composition_api.md)
