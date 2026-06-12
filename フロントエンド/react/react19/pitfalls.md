# 実務でハマる罠まとめ（Pitfalls）（React）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、症状からの原因切り分けの入口として使う。対象 = React 18/19（TypeScript前提）。

## 役割・なぜ必要か
- React は「動くが事故る書き方」と「正しい書き方」が紙一重。症状から該当箇所へ素早く飛ぶための索引。
- 「再レンダリングされない」「無限ループ」「画面が古いまま」の多くは、下記の定番パターンが原因。

## state / レンダリング
- **state の直接変更（ミューテーション）**：`state.push(x)` / `obj.foo = 1` は参照が変わらず**再レンダリングされない**。`setState([...arr, x])` / `setState({ ...obj, foo: 1 })` と新しいオブジェクトを作る。→ [state.md](./state.md)
- **更新が前の値に依存するのに直接渡す**：`setCount(count + 1)` を連続で呼ぶと古い値基準でズレる。関数形式 `setCount((c) => c + 1)` を使う。→ [state.md](./state.md)
- **派生値を state に持つ**：他の state から計算できる値を別 state に保持すると同期ズレ。レンダリング時に計算（必要なら `useMemo`）。→ [state.md](./state.md) / [performance.md](./performance.md)

## useEffect / 副作用
- **依存配列の漏れ**：使っている値を依存に書かないと、古い値を掴んだまま（stale closure）で更新されない。→ [useeffect.md](./useeffect.md)
- **無限ループ**：effect 内で依存に入っている state/object を毎回更新、または依存にオブジェクト/関数リテラルを入れて毎回別参照に。`useCallback`/`useMemo` か依存設計を見直す。→ [useeffect.md](./useeffect.md)
- **クリーンアップ忘れ**：購読・タイマー・リスナーを return で解除しないとリーク。→ [useeffect.md](./useeffect.md)
- **本来 effect が不要なケース**：イベントで足りる処理を effect でやると複雑化。レンダー中に計算 or イベントハンドラへ。→ [useeffect.md](./useeffect.md)

## リスト / key
- **key に配列の index**：並び替え・挿入・削除で状態や入力値が**別の行に化ける**。安定した一意ID（`item.id`）を使う。→ [lists_keys.md](./lists_keys.md)
- **key 無し**：警告が出るだけでなく差分計算が非効率に。→ [lists_keys.md](./lists_keys.md)

## パフォーマンス
- **早すぎる最適化**：効果測定せず `memo`/`useMemo`/`useCallback` を全体にばら撒くと、依存比較コストと可読性低下だけ増える。まず計測、ボトルネックにだけ適用。→ [performance.md](./performance.md)
- **`useMemo`/`useCallback` の依存ミス**：依存漏れで古い値を返す／毎回リテラルを渡してメモが効かない。→ [performance.md](./performance.md)

## データ取得 / サーバ状態
- **サーバ状態を `useState` に二重保持**：`useQuery` の `data` を `useState` にコピーすると真実が二重化してズレる。`data` から直接派生。→ [data_fetching.md](./data_fetching.md)
- **`useEffect` 直書き fetch の race / 重複**：引数が変わると古い応答が後着で上書き。クリーンアップで無効化、または TanStack Query / SWR に任せる。→ [data_fetching.md](./data_fetching.md)

## フック
- **フックのルール違反**：`if`/ループ/早期 return の後でフックを呼ぶと順序が崩れて壊れる。**トップレベルで常に同じ順序**で呼ぶ。コンポーネント/カスタムフック以外から呼ばない。→ [hooks.md](./hooks.md)
- **条件分岐内のフック**：条件はフックの内側で扱い、フック呼び出し自体は無条件に。→ [hooks.md](./hooks.md)

## フォーム
- **制御 / 非制御の混在**：`value` を渡しつつ `onChange` 無し、または `value={undefined}↔値` を行き来すると「controlled/uncontrolled が切り替わった」警告＆入力不能。初期値を空文字に固定し `value`+`onChange` で統一。→ [forms.md](./forms.md)

## Context / 再描画
- **Context の値変更で配下が全再描画**：`value` に毎回新しいオブジェクト/関数を渡すと購読側が総再描画。`value` をメモ化、関心ごとに Provider を分割。→ [context.md](./context.md)
- **巨大 Context への詰め込み**：頻繁に変わる値と滅多に変わらない値を同居させると無駄な再描画。分割するか状態管理ライブラリへ。→ [context.md](./context.md)

## React 共通の挙動
- **StrictMode の二重実行**：開発時、コンポーネントの描画と effect が**意図的に2回**走る。副作用が冪等でない（重複fetch・カウント二重）と不具合が露見する＝設計の問題。本番では1回。→ [useeffect.md](./useeffect.md)
- **`key` での意図的な再マウント**：別物として作り直したい時は `key` を変えると state がリセットされる（活用も罠も両面）。→ [lists_keys.md](./lists_keys.md)

## 関連
[state.md](./state.md) / [useeffect.md](./useeffect.md) / [hooks.md](./hooks.md) / [data_fetching.md](./data_fetching.md) / [forms.md](./forms.md) / [context.md](./context.md) / [performance.md](./performance.md) / [lists_keys.md](./lists_keys.md)
