# リスト（ListView / ListView.builder）（Flutter 3）

## ひとことで言うと
縦（横）方向に並ぶ項目をスクロール表示するための Widget 群。要素が少なければ **`ListView`**、多ければ **`ListView.builder`** で「見えている分だけ」遅延生成するのが基本。

## 役割・なぜ必要か
- モバイルUIは「縦に並べてスクロール」が王道。リスト系 Widget はその定番をまとめて提供する。
- 画面に入る項目数は限られるので、**全要素を一度に作るとメモリ・初期表示が重い**。`.builder` は表示範囲だけ `build` し、スクロールに応じて作る／捨てるため大量データでも軽い。
- `ListTile` で「アイコン＋タイトル＋副題＋末尾」の定型行を、`GridView` で格子状を、それぞれ手早く作れる。

## 基本の書き方（コード）
```dart
// 1) 少数・固定 → ListView（children を直接渡す）
ListView(
  children: const [
    ListTile(leading: Icon(Icons.home), title: Text('ホーム')),
    ListTile(leading: Icon(Icons.settings), title: Text('設定')),
    Divider(),
    ListTile(title: Text('その他')),
  ],
);

// 2) 大量・遅延生成 → ListView.builder（これが実務の主役）
final items = List.generate(1000, (i) => 'item $i');

ListView.builder(
  itemCount: items.length,            // 件数（省略すると無限）
  itemBuilder: (context, index) {     // 見えている index だけ呼ばれる
    final item = items[index];
    return ListTile(
      key: ValueKey(item),            // 並び替え/挿入削除があるなら付ける
      title: Text(item),
      onTap: () => debugPrint('tapped $index'),
    );
  },
);

// 区切り線つきは ListView.separated
ListView.separated(
  itemCount: items.length,
  itemBuilder: (context, i) => ListTile(title: Text(items[i])),
  separatorBuilder: (context, i) => const Divider(height: 1),
);

// 3) 格子状 → GridView.builder
GridView.builder(
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 3,            // 横に3列
    mainAxisSpacing: 8,
    crossAxisSpacing: 8,
  ),
  itemCount: items.length,
  itemBuilder: (context, i) => Card(child: Center(child: Text(items[i]))),
);

// 4) スクロールさせたいが「リスト」ではない任意レイアウト → SingleChildScrollView
SingleChildScrollView(
  child: Column(
    children: const [
      Text('長い説明文…'),
      SizedBox(height: 800),
      Text('下の方のコンテンツ'),
    ],
  ),
);
```

## 実務での使い方・定番パターン
- **件数が読めない／サーバ由来のデータは原則 `.builder`**。`itemCount` に取得済みリストの `length` を渡す。
- **`ListTile`** は行UIの定番。`leading`（先頭）/`title`/`subtitle`/`trailing`（末尾）/`onTap` を埋めるだけで Material 準拠の行になる。
- **`ListView.separated`** で項目間の区切り線を宣言的に。自前で `Divider` を混ぜるより意図が明確。
- **`GridView.builder`** はギャラリーやカード一覧に。列数は `SliverGridDelegateWithFixedCrossAxisCount`、項目幅基準なら `...WithMaxCrossAxisExtent`。
- 横スクロールにしたい時は `scrollDirection: Axis.horizontal`。
- 「ヘッダ＋リスト＋フッタ」など複合スクロールは `CustomScrollView` + `Sliver*`（`SliverList` / `SliverGrid`）へ。
- 引っ張って更新は `RefreshIndicator` でリストを包む。

## ハマりどころ / アンチパターン
- **大量データを `ListView(children: [...])` で作る**＝全要素を一度に生成し、メモリ・初期描画が重くなる。**件数が多い／不定なら必ず `.builder`**。
- **`Column` の中に高さ無制限の `ListView` を直接置く**と「unbounded height」例外。`Expanded` / `Flexible` で囲む、または親に高さを与える（`SizedBox(height:)`）。逆に `SingleChildScrollView` 内のリストは `shrinkWrap: true` ＋ `physics: NeverScrollableScrollPhysics()` で“親に任せる”形にする（多用は重いので件数が少ない時だけ）。
- **`SingleChildScrollView` + 巨大 `Column` で大量項目を並べる**のは `.builder` の利点を捨てる。スクロールするリストは原則リスト系 Widget で。
- **`key` 不足**：項目の挿入・削除・並び替え時、`key` が無いと State が別要素に引き継がれて表示が崩れる。動的リストは `ValueKey` 等を付ける。
- **`itemBuilder` 内で重い処理／毎回 `Future` 発行**：スクロールのたびに走る。整形済みデータを渡す、画像は `cached_network_image` 等でキャッシュ。
- **入れ子スクロールの取り違え**：縦スクロール内に同方向リストを素朴に重ねるとスクロールが競合。複合は `CustomScrollView`/`Sliver` で一本化する。
- ネットワーク画像は `Image.network` 直貼りだと毎回再取得になりがち。キャッシュ前提で。

## 関連
[widgets.md](./widgets.md) / [layout.md](./layout.md) / [async_futures.md](./async_futures.md)
