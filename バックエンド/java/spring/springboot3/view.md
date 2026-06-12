# ビュー（View）（Spring Boot 3）

## ひとことで言うと
ユーザーに返す「見える出力」を作る層。Spring Boot では大きく2パターンある:
1. **REST API（多数派）**: Viewは実質 **JSON**。Jackson が Java オブジェクト→JSON に変換するので、専用テンプレートは不要。
2. **サーバサイドHTML**: **Thymeleaf**（Spring標準のテンプレートエンジン）で HTML を組み立てる。

## 役割・なぜ必要か
- 「データ（Model/DTO）」を「表示形式（JSON or HTML）」に変換するのがViewの仕事。
- 今どきのSpringは **REST API＋別フロント（React等）** が多数派。サーバでHTMLを描くのは管理画面・社内ツール・軽いページなどで、そこに **Thymeleaf** を使う。

## 基本の書き方（コード）

### パターン1: REST（View＝JSON）
```java
@RestController                       // = @Controller + @ResponseBody
@RequestMapping("/api/users")
class UserApiController {
    private final UserService service;
    UserApiController(UserService s) { this.service = s; }

    @GetMapping("/{id}")
    UserDto show(@PathVariable Long id) {
        return service.findDto(id);   // ← 戻り値DTOがJacksonで自動的にJSONになる
    }
}
```
→ DTOで「見せる形」を決める。[dto.md](./dto.md)

### パターン2: サーバサイドHTML（Thymeleaf）
```java
@Controller                           // ← @RestControllerではない（HTMLを返す）
class UserPageController {
    private final UserService service;
    UserPageController(UserService s) { this.service = s; }

    @GetMapping("/users")
    String list(Model model) {
        model.addAttribute("users", service.findAll()); // ViewにデータをModelで渡す
        return "users";               // templates/users.html を描画
    }
}
```
```html
<!-- src/main/resources/templates/users.html -->
<table>
  <tr th:each="u : ${users}">       <!-- ループ -->
    <td th:text="${u.email}">sample@example.com</td>  <!-- 値の埋め込み（自動エスケープ） -->
  </tr>
</table>
```
- `th:text` は **自動でHTMLエスケープ**（XSS対策）。`th:utext` は生出力＝危険。
- 依存は `spring-boot-starter-thymeleaf`。テンプレートは `src/main/resources/templates/`。

## 【最重要】Model と View の関係（一番つまずく所）

> 用語は**出てくる順に1つずつ**積み上げる。整理用の番号(①②③)は**最後だけ**に付ける（先に番号を使わない）。

### ステップ1：ControllerとViewは別の担当
- **Controller** … データを用意する人（DBから取る等）。
- **View** … そのデータを HTML に並べて画面にする人。

担当が別だから、問題が1つ起きる:
```
 Controller「ユーザー一覧のデータ、用意できた」
 View      「で、そのデータ どうやって受け取るの？」
```
→ この**受け渡しの仕組み**が要る。それが次の `Model`。

（Springの Model クラスというのが別にあるわけです。これがややこしい）
（そのデータを View に渡す運搬コンテナ（Map）です）

### ステップ2：受け渡しの“メモ帳” ＝ `Model`
Spring は Controller に `Model` という入れ物を1つ渡してくる。
正体は **「名前 → 値」を書き込むメモ帳（Map / 辞書）** ：
```java
String list(Model model) {                 // Spring が渡してくれる入れ物
    var users = userService.findAll();      // 用意したデータ
    model.addAttribute("users", users);     // メモに「users = このデータ」と1行書く
    return "users";                         // 表示する画面の名前
}
```
このときの `model` の中身:
```
 model（名前→値のメモ帳）
 ┌────────────────────────────┐
 │ "users" → [ユーザーのデータ] │   ← addAttribute で書いた1行
 └────────────────────────────┘
```
View はそのメモ帳を **名前で読む** だけ:
```html
<tr th:each="u : ${users}">   <!-- model から "users" を取り出す -->
  <td th:text="${u.email}">
</tr>
```
➡ **`Model` ＝ Controllerが書いて、Viewが読む“共有メモ帳”**。これが正体。

### ステップ3：メモ帳に書いた“中身”は何者か ＝ エンティティ
さっき `"users"` に入れたデータの中身は `User` クラス。
これは **DBの1行に対応するデータ**（`@Entity`）＝いわゆる **エンティティ**。

ここで罠が1つ:
```
 「データ(User)」のことも、ふつう “モデル” と呼ぶ（データモデル）。
 でも ステップ2の Model（メモ帳）とは 別物：
     ・メモ帳の Model       … 運ぶ「入れ物」
     ・データの model(User)  … 運ばれる「中身」
```
**「Model」という言葉が、“入れ物” と “中身” の両方に使われている**。混乱の正体はこれ。

### ステップ4：メモ帳＋画面名を1個に ＝ `ModelAndView`
ステップ2では「メモ帳(`model`)」と「画面名(`return "users"`)」を**別々**に扱った。
これを **1個にまとめて返す**書き方が `ModelAndView`。中身は同じ:
```java
ModelAndView list() {
    ModelAndView mav = new ModelAndView("users");   // 画面名
    mav.addObject("users", userService.findAll());  // メモ帳に書くのと同じ
    return mav;                                      // 「メモ帳 ＋ 画面名」を1個で返す
}
```
➡ 新概念ではなく、**メモ帳と画面名をホチキスで留めただけ**。

### ここで初めて整理（番号をつける）
全部出そろったので、ようやく番号で整理:

| 番号 | 名前 | 何者か |
|---|---|---|
| ① | データ（`User` ＝ エンティティ） | 運ばれる**中身** |
| ② | `Model`（メモ帳/Map） | ①を運ぶ**入れ物**。Controllerが書く・Viewが読む |
| ③ | `ModelAndView` | ②のメモ帳 ＋ 画面名 を1個にしただけ |

```
 流れ: ①データを用意 → ②メモ帳に「名前 = ①」と書く → 画面名と一緒に View へ
                                                    → View がメモ帳から①を読んで表示
```
> **元の疑問「Viewの中にModelがある？」→ ありません。**
> ②メモ帳が①を運び、View がメモ帳を読んで①を表示するだけ。③はそのメモ帳に画面名をくっつけただけ。

---

## 実務での使い方・定番パターン
- **API中心の設計なら View は基本JSON**。表示形は DTO で制御し、`@RestController` で返す。
- **サーバHTMLが要るとき（管理画面・メールHTML・軽量ページ）に Thymeleaf**。
- 大規模な動的UIは Thymeleaf で頑張りすぎず、フロント（React/Vue）に寄せる。

## ハマりどころ / アンチパターン
- **`@RestController` で HTML を返そうとする**：それは `@Controller`＋Thymeleaf。`@RestController` は戻り値をそのままボディ（JSON）にする。
- **`th:utext` でユーザー入力を出す**：XSS。原則 `th:text`（自動エスケープ）。
- **Entity をそのまま View/JSON に渡す**：内部漏れ・`LazyInitializationException`。DTOへ。→ [dto.md](./dto.md) / [entity_jpa.md](./entity_jpa.md)
- テンプレートの置き場所ミス（`src/main/resources/templates/` 配下でないと解決されない）。

## 関連
[controller.md](./controller.md) / [dto.md](./dto.md) / [model.md](./model.md)
