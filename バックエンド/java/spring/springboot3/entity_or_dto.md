# Modelの箱に入れるのは Entity か DTO か（Spring Boot 3）

> 「コントローラでリストにしてるモデルって、実際にはエンティティ？」への答えと、実務での正解。
> 用語の前提（①Entity / ②データを View に渡す運搬コンテナ（Map） / ③ModelAndView）は [view.md の「Model と View の関係」](./view.md#最重要model-と-view-の関係一番つまずく所) を先に。

## まず質問の答え：はい、中身は①エンティティ

```java
List<User> users = userService.findAll();
//        └ User = ①エンティティ。 だから users は「①の集まり（リスト）」
model.addAttribute("users", users);
//   └ model = ②そのデータを View に渡す運搬コンテナ（Map）       └ ①のリストを ②の箱に入れている
```

- **`List<User> users`** … 中身は **`User`＝①エンティティ**。「リストにしてるモデル」＝**①の集まり**で合っている。
- **`model`** … ②そのデータを View に渡す運搬コンテナ（Map）。別物。

➡ 状態は **「②そのデータを View に渡す運搬コンテナ（Map）に、①エンティティのリストを入れている」**。

## ただし実務の正解：箱に入れるのは Entity ではなく **DTO**

そのまま Entity を入れても“動く”が、実務では **Entity①→DTO に変換してから**箱に入れるのが定番:

```java
// 実務でよくある形
List<UserDto> users = userService.findUserDtos();  // ← ①Entityでなく DTO のリスト
model.addAttribute("users", users);                // ② の箱に DTO を入れる
```

### なぜ Entity を直接入れないのか
- **内部が漏れる**：Entityはパスワードハッシュ等のDBカラムも持つ → 画面/JSONに余計な情報が出る。
- **`LazyInitializationException`**：Entityの未ロードの関連を画面側で触ると落ちる（頻出事故）。
- **DB構造とAPI/画面が密結合**：Entityを変えると画面まで壊れる。DTOで切り離す。

→ DTOの詳細・マッピング方法は [dto.md](./dto.md)。

## 図でまとめ

```
 Controller が ②そのデータを View に渡す運搬コンテナ（Map）に入れる「中身」は2択:

   ┌─ そのまま①Entity を入れる  … 動くが 漏れ/例外/密結合 のリスク（非推奨）
   │
   └─ ①Entity → DTO に変換して入れる … 実務の正解（見せる形だけを渡す）

 どちらにしても「箱(②Model)」とは別物。箱は“運ぶ容れ物”、中身が①Entity or DTO。
```

## 一言まとめ
- 質問の答え：**`List<User>` の中身は①エンティティ**。`model`(②)はそれを運ぶ箱。
- 実務の正解：**Entityのまま入れず、DTOに変換してから箱に入れる**。
- 共通：**箱(②Model)と、その中身(①Entity / DTO)は別物**。

## 関連
[view.md](./view.md)（Model/Viewの関係）/ [dto.md](./dto.md)（DTOとは）/ [entity_jpa.md](./entity_jpa.md) / [model.md](./model.md)
