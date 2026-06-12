# 実務でハマる罠まとめ（Pitfalls）（SwiftUI）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、症状からの原因切り分けの入口として使う。

## 役割・なぜ必要か
- SwiftUI は「宣言的でラク」な一方、**状態・再描画・ライフサイクルの暗黙ルール**で事故りやすい。症状から該当箇所へ素早く飛ぶための索引。

## 状態 / バインディング
- **`@State` は `private`、View が所有する**：状態は「その View のもの」。外から初期化したい値は `@State` でなく引数 or `@Binding`/`@Observable` で渡す。→ [state.md](./state.md)
- **`$` でバインディングを渡す**：`TextField("", text: $text)` のように書き換えさせたい値は `$`。表示だけなら `$` なし。付け忘れが型エラー最頻原因。→ [state.md](./state.md)
- **値型 View に状態を渡したつもりが反映されない**：子で編集するなら子は `@Binding`、親は `@State` を `$` で渡す。値コピーでは親に戻らない。→ [state.md](./state.md)
- **iOS17 は `@Observable` を使う**：参照型の共有状態は旧 `ObservableObject`/`@Published` でなく `@Observable` マクロ＋ `@State`/`@Bindable` が新標準。→ [data_flow.md](./data_flow.md)
- **`@Environment` の入れ忘れ**：`.environment(model)` を祖先で注入し忘れると実行時に取得できず落ちる。→ [data_flow.md](./data_flow.md)

## レイアウト / 描画
- **modifier の順序が結果を変える**：`.padding().background()` と `.background().padding()` は別物。背景や枠は「広げてから塗る」順を意識。→ [modifiers.md](./modifiers.md)
- **`frame` の付け過ぎ**：固定サイズの乱用でレイアウトが破綻。まず親に任せ、必要な箇所だけ `frame`。→ [layout.md](./layout.md)
- **`some View` の意味**：`body` は単一の不透明型を返す。`if` の両分岐で型が違うと怒られる→ `Group` や `@ViewBuilder` で包む。→ [views.md](./views.md)

## リスト / 反復
- **`Identifiable` または `id:` が必須**：`ForEach`/`List` は各要素の安定したIDが要る。無いと並べ替え・削除で表示が崩れる。`id: \.self` は値が重複すると破綻。→ [lists.md](./lists.md)
- **`ForEach` の index 直渡し**：`ForEach(0..<array.count)` は要素変化に弱い。データ自体を回し `Identifiable` を使う。→ [lists.md](./lists.md)

## 非同期 / データ取得
- **非同期は `.task` で**：`onAppear` で `Task{}` を手書きするより `.task`。View 消滅時に自動キャンセルされる。→ [async_data.md](./async_data.md)
- **UI 更新は `@MainActor`（メインスレッド）**：URLSession 等のバックグラウンド処理結果で状態を更新する箇所はメインに乗せる。`@MainActor` 付与 or `await MainActor.run`。→ [async_data.md](./async_data.md)
- **`.task(id:)` の指定漏れ**：依存値が変わったら再実行したいのに `id:` を付けず一度きりで終わる。→ [async_data.md](./async_data.md)

## 画面遷移
- **旧 `NavigationView` → `NavigationStack`**：iOS16+ は `NavigationStack`。`NavigationView` は非推奨で挙動も古い。→ [navigation.md](./navigation.md)
- **`navigationDestination` の付け場所**：Stack の内側に置く。`sheet`/`alert` の `isPresented` バインディング忘れにも注意。→ [navigation.md](./navigation.md)

## 入力 / フォーム
- **フォーカスは `@FocusState`**：`@State Bool` では `.focused()` に渡せない。専用属性を使う。→ [forms_input.md](./forms_input.md)
- **`Form` は標準スタイル強制**：作り込み UI に `Form` を使うと余白・色で戦う。自由 UI は `VStack`/`ScrollView`。→ [forms_input.md](./forms_input.md)
- **`Picker` の `tag` 型不一致**：`selection` と `tag` の型を揃えないと選択が反映されない。→ [forms_input.md](./forms_input.md)

## 環境 / ツール
- **Mac + Xcode が必須**：SwiftUI のビルド・プレビュー・実機/シミュレータ実行は実質 Mac + Xcode 前提。Linux 等では完結しない。→ [getting_started.md](./getting_started.md)

## 関連
[state.md](./state.md) / [data_flow.md](./data_flow.md) / [navigation.md](./navigation.md) / [modifiers.md](./modifiers.md) / [lists.md](./lists.md) / [async_data.md](./async_data.md) / [forms_input.md](./forms_input.md)
