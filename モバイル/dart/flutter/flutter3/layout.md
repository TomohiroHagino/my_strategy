# レイアウト（Row / Column / Container / 制約）（Flutter 3）

## ひとことで言うと
Widget を「**どう並べ・どう配置し・どんなサイズにするか**」を決める仕組み。横/縦に並べる `Row`/`Column`、余白や装飾の `Container`、重ねる `Stack` が土台。

## 役割・なぜ必要か
- 画面は「部品をどう置くか」で決まる。Flutter のレイアウトは **制約（constraints）モデル**：**親が子に「使ってよいサイズの範囲」を渡し、子が自分のサイズを返し、親がそれを配置する**（上→下に制約、下→上にサイズ、最後に親が位置決め）。
- この一方向の流れを理解していないと「なぜ広がらない/なぜはみ出す」が読めない。レイアウトのバグはほぼ**制約の取り違え**。
- `Row`/`Column` は「主軸（並ぶ方向）」と「交差軸」を持ち、`Expanded`/`Flexible` で残りスペースの分配を制御する。

## 基本の書き方（コード）
```dart
import 'package:flutter/material.dart';

class Demo extends StatelessWidget {
  const Demo({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(                          // 縦に並べる（主軸=縦）
      crossAxisAlignment: CrossAxisAlignment.stretch, // 交差軸=横いっぱい
      children: [
        // Container：余白(padding)・外側余白(margin)・装飾(decoration)
        Container(
          padding: const EdgeInsets.all(16),
          margin: const EdgeInsets.only(bottom: 8),
          decoration: BoxDecoration(
            color: Colors.blue.shade50,
            borderRadius: BorderRadius.circular(12),
          ),
          child: const Text('カード風'),
        ),

        // Row：横に並べる。Expanded で残り幅を比率配分
        Row(
          children: [
            const Icon(Icons.star),
            const SizedBox(width: 8),         // 固定の隙間
            Expanded(child: Text('伸びるテキスト' * 3)), // 残り幅を全部もらう
            const Text('右'),
          ],
        ),

        // Stack：子を重ねる（左上基準、Positioned で位置指定）
        SizedBox(
          height: 80,
          child: Stack(
            children: [
              Container(color: Colors.amber),
              const Positioned(right: 8, bottom: 8, child: Text('右下')),
            ],
          ),
        ),
      ],
    );
  }
}
```

## 実務での使い方・定番パターン
- **並べる＝`Row`/`Column`、余白/装飾＝`Container`、中央寄せ＝`Center`、固定サイズ/隙間＝`SizedBox`、内側余白だけ＝`Padding`**。役割で使い分けると読みやすい。
- **`Expanded` と `Flexible`**：`Expanded` は「残りを必ず埋める（flex 比で分配）」、`Flexible` は「最大その範囲まで、必要なら縮む」。複数 `Expanded(flex: 2)` 等で比率配分。
- **可変テキスト/長い行は `Expanded` か `Flexible` で包む**。これが overflow を防ぐ定番。
- **`Container` を装飾なしのサイズ調整に使うな**：余白だけなら `Padding`、隙間だけなら `SizedBox` の方が意図が明確（KISS）。
- **重ね表示（バッジ・タグ・オーバーレイ）は `Stack` + `Positioned`**。

## ハマりどころ / アンチパターン
- **"RenderFlex overflowed by N pixels"（はみ出し）**：`Row`/`Column` の子の合計が画面を超えた合図。対処は ① 伸びる子を `Expanded`/`Flexible` で包む、② スクロールさせたいなら `SingleChildScrollView`/`ListView` にする、③ テキストは `overflow: TextOverflow.ellipsis`。→ [pitfalls.md](./pitfalls.md)
- **`Expanded`/`Flexible` を `Row`/`Column` の外で使う** → 例外（Incorrect use of ParentDataWidget）。これらは **Flex（Row/Column）の直接の子**でのみ有効。
- **制約モデルの誤解**：親が「高さ無限」を渡す場所（縦スクロール内など）に「高さいっぱいに広がろうとする子」を置くと無限高でエラー。逆に親が幅0を渡せば子は潰れる。「親が範囲を決める」を常に意識する。
- **`Container` に色も大きさも child も無い** → サイズ0で消える。装飾か制約を必ず与える。
- **無限の入れ子で深いネスト**：4段を超えたら小 Widget に切り出す。→ [widgets.md](./widgets.md)

## 関連
[widgets.md](./widgets.md) / [styling_theming.md](./styling_theming.md)
