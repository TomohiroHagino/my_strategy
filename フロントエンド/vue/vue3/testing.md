# テスト（Testing）（Vue 3）

## ひとことで言うと
コンポーネントやロジックの**振る舞いを自動で検証するコード**。Vue 3 の実務標準は **Vitest**（テストランナー）＋ **Vue Test Utils**（コンポーネントをマウントして操作・検証する公式ライブラリ）。より「ユーザー視点」で書きたい場合は **Testing Library for Vue** も選択肢。

## 役割・なぜ必要か
- リアクティビティ・props・emit・条件分岐が絡むと、手で全状態を確認するのは非現実的。**回帰（デグレ）を自動検出**するために要る。
- 「この props を渡すとこう描画／このクリックでこのイベントが飛ぶ」という**実行可能な仕様書**になり、リファクタの安全網になる。
- Vite ベースの **Vitest** は設定が薄く高速。`@vue/test-utils` でDOM操作・イベント発火・出力検証ができる。

## 基本の書き方（コード）
```ts
// vitest.config.ts（jsdom環境＋Vueプラグイン）
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom', // DOMをエミュレート（happy-domでも可）
    globals: true,        // describe/it/expectをグローバルに
  },
})
```

```ts
// Counter.spec.ts（Vue Test Utils：mountして操作→検証）
import { mount } from '@vue/test-utils'
import { describe, it, expect } from 'vitest'
import Counter from './Counter.vue'

describe('Counter', () => {
  it('ボタンクリックで表示が増える', async () => {
    const wrapper = mount(Counter, { props: { start: 0 } })

    // find: セレクタで要素を取得 / trigger: イベント発火
    await wrapper.find('button.increment').trigger('click')

    // クリックでDOMが更新されるのを await で待ってから検証
    expect(wrapper.find('.count').text()).toBe('1')
  })

  it('初期値 props を表示する', () => {
    const wrapper = mount(Counter, { props: { start: 5 } })
    expect(wrapper.text()).toContain('5')
  })

  it('increment イベントを emit する', async () => {
    const wrapper = mount(Counter)
    await wrapper.find('button.increment').trigger('click')
    // emit された値を検証
    expect(wrapper.emitted('increment')).toHaveLength(1)
  })
})
```

```ts
// 非同期更新は nextTick / flushPromises で待つ
import { mount, flushPromises } from '@vue/test-utils'
import { nextTick } from 'vue'

it('非同期取得後にリストを描画', async () => {
  const wrapper = mount(UserList)        // onMounted で fetch する想定
  await flushPromises()                  // 保留中のPromise（fetch等）を解決
  await nextTick()                       // DOM反映を待つ
  expect(wrapper.findAll('li')).toHaveLength(3)
})
```

```ts
// Testing Library for Vue（ユーザー視点：役割やテキストで取得）
import { render, screen } from '@testing-library/vue'
import userEvent from '@testing-library/user-event'
import Counter from './Counter.vue'

it('ユーザー視点でクリックを検証', async () => {
  render(Counter, { props: { start: 0 } })
  await userEvent.click(screen.getByRole('button', { name: '増やす' }))
  expect(screen.getByText('1')).toBeInTheDocument()
})
```

## 実務での使い方・定番パターン
- **テストピラミッド**を意識：土台に多数の高速な**ユニットテスト**（composables・純粋関数）、中間に**コンポーネントテスト**（props/emit/描画）、頂点に少数の **E2E**（Playwright/Cypress で重要フローのみ）。
- **`mount` と `shallowMount`** を使い分ける。`mount` は子まで実描画して結合を見る／`shallowMount` は子をスタブ化し対象だけを隔離（重い子・外部依存を切る）。
- **props を渡して入力、`emitted()` で出力を検証**。コンポーネントの「契約（入出力）」をテストするのが基本。→ [components.md](./components.md)
- **composables（`useXxx`）は単体テスト**：UIを介さず関数として `ref`/`computed` の挙動だけ検証できると速くて安定。
- **Testing Library** は `getByRole` / `getByText` などユーザー視点のクエリで、実装詳細（クラス名やDOM構造）に依存しにくいテストが書ける。
- 外部API・タイマー等は `vi.mock` / `vi.useFakeTimers` でモック。境界だけ差し替え、内部は本物を動かす。

## ハマりどころ / アンチパターン
- **DOM更新待ち忘れ**：Vueの再描画は**非同期**。`trigger` や状態変更の直後に検証すると古いDOMを見て落ちる。**`await wrapper.find(...).trigger(...)`** か **`await nextTick()`** を挟む。
- **非同期処理の取りこぼし**：`onMounted` の `fetch` などは **`await flushPromises()`** で解決を待つ。待たないと「まだ空のリスト」を検証してしまう。
- **実装詳細テスト**：内部のメソッド名・data構造・クラス名に依存すると、リファクタで赤くなる「もろい」テストに。**入出力（props/emit/表示テキスト）**を検証する。
- **`shallowMount` の過信**：子を全スタブ化すると結合不具合を見逃す。重要な親子連携は `mount` で実描画する。
- **`wrapper.vm` 直叩き**：内部状態を直接いじる／読むのはユーザー操作と乖離。なるべく `find`＋`trigger`＋表示で検証。
- **テスト間の状態リーク**：モジュールキャッシュ・Pinia・タイマーが残り順序依存で落ちる。`beforeEach` でストア初期化、`vi.restoreAllMocks()` で後始末。

## 関連
[components.md](./components.md) / [composition_api.md](./composition_api.md)
