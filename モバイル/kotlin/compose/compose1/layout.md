# レイアウト（Jetpack Compose）

## ひとことで言うと
部品を**縦に並べる `Column` / 横に並べる `Row` / 重ねる `Box`** と、サイズ・余白・配置を指定する **`Modifier`** の組み合わせ。Compose の画面はこの3つ＋Modifierでほぼ組める。

## 役割・なぜ必要か
- XML の `LinearLayout` / `FrameLayout` 等に相当する役割を、**`@Composable` 関数**として提供する。並べる・重ねるをコードで宣言する。
- `Column`/`Row`/`Box` は「子をどう配置するか」だけを担い、**サイズ・余白・背景・クリックなどの装飾は `Modifier`** が担う。役割が分かれているので組み合わせが効く。
- `Spacer` で隙間、`Arrangement` で主軸方向の詰め方、`Alignment` で交差軸の揃え、`weight` で残り領域の比率配分、と語彙が揃っている。→ [modifiers.md](./modifiers.md)

## 基本の書き方（コード）
```kotlin
@Composable
fun ProfileRow() {
    Row(
        modifier = Modifier
            .fillMaxWidth()         // 横いっぱい
            .padding(16.dp),        // 外側に余白
        verticalAlignment = Alignment.CenterVertically,  // 縦方向の揃え
        horizontalArrangement = Arrangement.spacedBy(8.dp) // 子の間隔
    ) {
        Text("名前")
        Spacer(Modifier.weight(1f))  // 残り幅を食って右に押し出す
        Text("編集")
    }
}
```

```kotlin
@Composable
fun Badge() {
    // Box は子を「重ねる」。後に書いた子ほど前面
    Box(contentAlignment = Alignment.TopEnd) {
        Image(/* ... */)                 // 下
        Text("3", Modifier.background(Color.Red)) // 上（右上に重なる）
    }
}

@Composable
fun SplitRow() {
    Row {
        // weight で残り幅を 1:2 に配分
        Text("左", Modifier.weight(1f))
        Text("右", Modifier.weight(2f))
    }
}
```

## 実務での使い方・定番パターン
- **`Column`＝縦、`Row`＝横、`Box`＝重ね**を基本語彙にする。多くの画面はこの入れ子で組める。リストは `Column` ではなく `LazyColumn`。→ [lists.md](./lists.md)
- **`Modifier` は呼び出し側から引数で受け取る**のが慣習。`@Composable fun X(modifier: Modifier = Modifier)` と書き、ルート要素に渡す。これで親がサイズ・余白を自由に指定できる＝再利用性が上がる。→ [modifiers.md](./modifiers.md)
- 隙間は **`Spacer(Modifier.height(8.dp))`** か、`Arrangement.spacedBy(8.dp)` でまとめて指定。後者の方がスッキリする。
- 残り領域の比率配分は **`weight`**。「片方は内容ぶん、もう片方は残り全部」は `Modifier.weight(1f)` を片側に付ける定番。
- 揃えは `Column` なら `horizontalAlignment`、`Row` なら `verticalAlignment`、`Box` なら `contentAlignment`／子の `Modifier.align(...)`。

## ハマりどころ / アンチパターン
- **`Modifier` を引数で受け渡さない**設計はアンチパターン。部品内で `Modifier` を固定すると親が余白やサイズを調整できず、使い回しが効かなくなる。`modifier: Modifier = Modifier` を必ず引数に。
- **`weight` を `Box` で使おうとする**：`weight` は `Row`/`Column` のスコープ内専用（`RowScope`/`ColumnScope`）。`Box` では使えない。
- **`Box` の重ね順を取り違える**：後に書いた子ほど前面（z順は記述順）。背景を先、前面要素を後に書く。
- `Modifier` の**順序で結果が変わる**（`padding` → `background` と `background` → `padding` は別物）。サイズ・余白系は順序に意味がある。→ [modifiers.md](./modifiers.md)
- `Column` に大量要素を直接並べると**全部一度に構築**されて重い。スクロールする長いリストは `LazyColumn` を使う。→ [lists.md](./lists.md)

## 関連
[modifiers.md](./modifiers.md)
