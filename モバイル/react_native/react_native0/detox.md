# Detox（React Native）

## ひとことで言うと
RN 向けの**E2E（エンドツーエンド）テストフレームワーク**。実機/シミュレータ上で**本物のアプリを起動**し、`device`（アプリ操作）/ `element(by.id(...))`（要素特定）/ `expect(...).toBeVisible()`（検証）でユーザー操作を再現する。グレーボックス方式で、アプリが「アイドル状態になるまで」自動で待つため、固定 `sleep` 無しで安定したテストが書ける。

## 役割・なぜ必要か
- ユニット/コンポーネントテスト（RNTL → [rntl.md](./rntl.md)）では再現できない、**実機・実権限ダイアログ・画面遷移・ネイティブ連携を含む通しのフロー**を検証する。
- 「起動 → ログイン → 一覧 → 詳細」のような**重要ユーザーフロー**を、リリース前に自動で守る。
- グレーボックス（アプリ内部の状態を見られる）ゆえに、ネットワークやアニメの完了を**自動で待機**でき、E2E にありがちな flaky を抑えられる。
- テストピラミッドの頂点：数は少なく、重要フローに絞って運用する。

## 基本の書き方（コード）
```bash
npm install --save-dev detox jest
# iOS は applesimutils、Android はエミュレータが必要
npx detox init   # .detoxrc.js と e2e/ を生成
```
```jsx
// アプリ側：要素に testID を付ける（by.id で掴むため）
<TextInput testID="emailInput" placeholder="メール" />
<Button testID="loginButton" title="ログイン" onPress={onLogin} />
```
```js
// e2e/login.test.js
describe('ログインフロー', () => {
  beforeAll(async () => {
    await device.launchApp({ permissions: { notifications: 'YES' } }); // 権限も付与可
  });

  beforeEach(async () => {
    await device.reloadReactNative(); // 各テストで状態をリセット
  });

  it('正しい認証情報でホームに遷移する', async () => {
    // by.id で要素特定 → typeText / tap で操作
    await element(by.id('emailInput')).typeText('a@example.com');
    await element(by.id('passwordInput')).typeText('pass1234');
    await element(by.id('loginButton')).tap();

    // 自動待機しつつ表示を検証
    await expect(element(by.id('homeScreen'))).toBeVisible();
    await expect(element(by.text('ようこそ'))).toBeVisible();
  });

  it('スクロールして項目をタップ', async () => {
    await element(by.id('list')).scrollTo('bottom');
    await element(by.text('最後の項目')).tap();
    await expect(element(by.id('detailScreen'))).toBeVisible();
  });
});
```
```bash
npx detox build --configuration ios.sim.debug    # まずビルド
npx detox test  --configuration ios.sim.debug    # ビルド済みアプリを実行
```

## 実務での使い方・定番パターン
- **要素特定は `by.id`（testID）優先**：`by.text` は文言変更で壊れやすい。E2E では安定の `testID` を基本にする（RNTL の「テキスト優先」とは方針が逆）。
- **アクション**：`tap()` / `typeText()` / `clearText()` / `scroll()` / `scrollTo()` / `swipe()`。検証は `toBeVisible()` / `toExist()` / `toHaveText()`。
- **状態リセット**：`beforeEach` で `device.reloadReactNative()`、あるいは `device.launchApp({ delete: true })` でクリーンインストール起動。テスト間の独立性を保つ。
- **ネットワークは安定化**：実APIに依存せず**モックサーバ**を立てる。Detox の自動待機は API 完了も待つので、遅延レスポンスでも `sleep` 不要。
- **権限ダイアログ**：`launchApp({ permissions: { camera: 'YES', ... } })` で事前付与。実機固有のダイアログを越えられるのが E2E の価値。
- **CI 運用**：エミュレータ/シミュレータをCIで起動し、`detox build` →`detox test`。重要フロー（ログイン・決済・主要導線）に絞って数を抑える。
- **Maestro という選択肢**：YAML で手軽に書ける E2E。Detox はグレーボックスで待機が強い、Maestro は導入が軽い、という棲み分け。

## ハマりどころ
- **build を忘れて test**：Detox はビルド済みバイナリを動かす。`detox build` 抜きで `detox test` すると起動失敗。設定変更後は必ず再ビルド。
- **固定 `sleep` 依存**：自動待機があるのに `setTimeout` で待つと逆に不安定化。`waitFor(element).toBeVisible().withTimeout(ms)` を使う。
- **アニメ無限ループ / 進まない idle**：ローディングスピナー等の常時アニメがあると「アイドルにならない」と判定され待ち続ける。テスト時はアニメを止める／`device.disableSynchronization()` を局所的に使う。
- **testID 不足**：掴みたい要素に `testID` が無く `by.text` 頼みになって脆くなる。重要要素には `testID` を付ける。
- **テスト間の状態漏れ**：前テストのログイン状態等が残って失敗。`reloadReactNative` / クリーン起動でリセットする。
- **実APIへの依存で flaky**：本番/開発サーバに直接当てると遅延・障害で落ちる。モックサーバで決定的にする。
- **E2E を増やしすぎ**：細かいロジックまで E2E で総当たりすると遅く不安定。細部は RNTL（→ [rntl.md](./rntl.md)）、E2E は重要フローだけ。

## 関連
[rntl.md](./rntl.md) / [testing.md](./testing.md) / [navigation.md](./navigation.md) / [platform.md](./platform.md)
