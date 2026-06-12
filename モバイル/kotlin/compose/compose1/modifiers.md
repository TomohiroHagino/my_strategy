# Modifier（修飾子）（Jetpack Compose）

## ひとことで言うと
Composable に**見た目・サイズ・余白・配置・入力（クリック等）を後付けする部品**。`Modifier.padding(...).background(...).clickable(...)` のように**チェーン（数珠つなぎ）**で重ねていく。

## 役割・なぜ必要か
- Compose の Composable 自体は「何を表示するか」だけを持ち、「**どう装飾・配置するか**」は Modifier に分離されている。だから同じ `Text` でも Modifier 次第で余白・背景・サイズ・タップ挙動が変わる。
- HTML/CSS のように属性を個別に並べるのではなく、**1本の Modifier チェーンを上から順に適用**していくモデル。順番に意味があるのが特徴。
- Composable に Modifier を**引数で受け取らせる**ことで、呼び出し側が外から装飾を注入でき、再利用しやすくなる（Compose の慣習）。

## 基本の書き方（コード）
```kotlin
@Composable
fun Card() {
    Text(
        text = "Hello",
        modifier = Modifier
            .padding(16.dp)          // まず外側に余白
            .background(Color.Yellow) // 余白の内側を黄色で塗る
            .padding(8.dp)            // さらに内側に余白
            .clickable { /* tap */ }  // タップ可能領域
    )
}
```
**慣習：Composable は `modifier: Modifier = Modifier` を第1オプション引数として受け取る。**
```kotlin
@Composable
fun Label(
    text: String,
    modifier: Modifier = Modifier,   // 既定は空。外から装飾を注入できる
) {
    Text(text = text, modifier = modifier.padding(4.dp))
}

// 呼び出し側が外から付け足す
Label("Tag", modifier = Modifier.background(Color.LightGray))
```

## 実務での使い方・定番パターン
- **順序を意識して組む**：`padding` → `background` の順なら「余白の内側が塗られる」、`background` → `padding` の順なら「塗ってから内側に余白」。意図に合わせて並べる。
```kotlin
// A: 余白の外は塗られない（背景が小さく見える）
Modifier.padding(16.dp).background(Color.Cyan)

// B: 全体を塗ってから内側に余白（背景が大きく見える）
Modifier.background(Color.Cyan).padding(16.dp)
```
- **`size` と他の Modifier の順**も結果が変わる。`size` の前の `padding` はサイズの外側、後の `padding` は内側に効く。
```kotlin
Modifier.size(100.dp).padding(8.dp)   // 100dp 枠の内側に 8dp 余白（中身は 84dp）
Modifier.padding(8.dp).size(100.dp)   // 8dp 余白の内側に 100dp 枠（全体 116dp）
```
- **受け取った modifier は最初に適用**するのが定石：`modifier.then(...)` 相当で、内部の装飾より先に外部指定を効かせる（上の `Label` の書き方）。
- `fillMaxWidth()` / `weight()` / `align()` はレイアウト（Column/Row/Box）と組で使う。→ [layout.md](./layout.md)
- `clickable` は**タップ範囲＝その時点の Modifier が確定させた領域**。`padding` の前後でタップできる範囲が変わる点に注意。

## ハマりどころ / アンチパターン
- **Modifier の順序を軽視する**：最大のハマりどころ。`padding`→`background` と `background`→`padding` は別物。「上から順に効く」を常に意識する。並べ替えで見た目が直ることが多い。
- **`size` と `padding` の順違いでサイズがズレる**：`size` の前後どちらに `padding` を置くかで実寸が変わる。意図したサイズにならないときはまずこの順を疑う。
- **`clickable` のタップ範囲が思ったより狭い/広い**：`clickable` を `padding` の前に置くと余白部分が反応しない。タップ領域を広げたいなら `padding` の後に `clickable` を置く。
- **Modifier を引数で受け取らない／受け取っても使わない**：再利用性が落ちる。`modifier: Modifier = Modifier` を受け、内部装飾より先に適用する慣習を守る。
- **Modifier を `remember` せず毎回生成して気にしすぎる**：基本は気にしなくてよいが、重い処理を伴う場合のみ最適化を検討。まずは可読性優先（KISS）。
- **複数の競合する Modifier を重ねる**：`fillMaxWidth()` の後に `width(100.dp)` など矛盾する指定は後勝ち・想定外になりやすい。意図を1本に絞る。

## 関連
[layout.md](./layout.md) / [composables.md](./composables.md)
