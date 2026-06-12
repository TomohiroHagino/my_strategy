# Mockito（Spring Boot 3）

## ひとことで言うと
依存オブジェクトの**偽物（モック）**を作るライブラリ。テスト対象が依存する Repository や外部 API クライアントを「指定した値を返すだけのダミー」に差し替え、対象の振る舞いだけを隔離して検証する。Spring Boot のテストでサービス層の単体テストを書く中核ツール。

## 役割・なぜ必要か
- Service をテストするとき、本物の Repository を使うと DB が必要になり遅くて壊れやすい。**依存をモック化**して「この入力ならこの値を返す」と決め打ちすれば、DB なしで高速にロジックだけ検証できる。
- 外部 API・時刻・乱数といった**制御しにくい境界**を固定値に置き換え、再現性のあるテストにする。
- 「正しいメソッドが正しい引数で呼ばれたか」（`verify`）を確認でき、副作用の検証にも使う。

## 基本の書き方（コード）
```java
import static org.mockito.Mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import org.junit.jupiter.api.extension.ExtendWith;

@ExtendWith(MockitoExtension.class)   // @Mock / @InjectMocks を有効化
class OrderServiceTest {

    @Mock OrderRepository repo;            // 偽物の依存
    @InjectMocks OrderService service;     // repo を注入した実体（テスト対象）

    @Test
    void 在庫があれば注文できる() {
        // スタブ：この引数で呼ばれたらこの値を返す
        when(repo.stockOf(1L)).thenReturn(5);

        boolean ok = service.place(1L);

        assertTrue(ok);
        // 検証：save が1回・指定引数で呼ばれたか
        verify(repo, times(1)).save(any(Order.class));
    }
}
```
```java
// 手動生成（アノテーション無しでも書ける）
OrderRepository repo = mock(OrderRepository.class);
when(repo.findById(1L)).thenReturn(Optional.of(new Order()));

// 例外を投げさせる
when(repo.stockOf(99L)).thenThrow(new IllegalStateException("not found"));

// 戻り値が void のメソッドで例外
doThrow(new RuntimeException()).when(repo).delete(any());
```
```java
// ArgumentCaptor：実際に渡された引数を捕まえて中身を検証
@Test
void 保存される注文の内容を検証() {
    ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);

    service.place(1L);

    verify(repo).save(captor.capture());
    assertEquals("NEW", captor.getValue().getStatus());   // 渡された引数の中身
}
```
```java
// spy：本物を使いつつ一部だけ差し替え（部分モック）
List<String> real = new ArrayList<>();
List<String> spy = spy(real);
when(spy.size()).thenReturn(100);   // size だけ偽装、他は本物
spy.add("x");                        // add は本物が動く
```

## 実務での使い方・定番パターン
- **`@Mock` + `@InjectMocks` が基本形**：依存を `@Mock`、テスト対象を `@InjectMocks` にすると、Mockito がコンストラクタ／フィールドにモックを自動注入する。Spring を起動しないので最速。
- **Spring 連携は `@MockBean`**：`@WebMvcTest`/`@SpringBootTest` のようにコンテキストを立てるテストでは、`@Mock` ではなく **`@MockBean`** を使い、コンテナ内の Bean をモックに差し替える（→ [testing.md](./testing.md)）。
- **`verify` は境界だけ**：外部副作用（保存・送信）の確認に使う。内部の呼び出し順まで縛るともろくなる。
- **`ArgumentCaptor` で引数の中身を検証**：DTO 変換やマッピングが正しいかを、保存直前の値を捕まえて確認する。
- **`any()`・`eq()` の使い分け**：マッチャを1つ使ったら**全引数をマッチャにする**ルール。固定値は `eq(1L)` でラップする。

## ハマりどころ / アンチパターン
- **`@Mock` と `@MockBean` の混同（最頻）**：`@Mock` は Spring を立てない単体用、`@MockBean` は Spring コンテキスト内の Bean 差し替え用。`@WebMvcTest` で `@Mock` を使っても注入されず NPE になる。
- **マッチャの混在**：`verify(repo).save(any(), eq("x"))` のように生値とマッチャを混ぜると `InvalidUseOfMatchersException`。全部マッチャ（`any()` と `eq("x")`）にする。
- **`when().thenReturn()` の過剰スタブ**：呼ばれないスタブを書くと `MockitoExtension` が `UnnecessaryStubbingException` を出す。使わないスタブは消すか `lenient()`。
- **final / static メソッドのモック**：標準では不可（mockito-inline で final は可、static は `mockStatic`）。設計で避けるのが基本。
- **`verify` で内部実装を縛りすぎる**：呼び出し順・回数を細かく検証するとリファクタで赤くなる。**結果**を `assertThat`（[AssertJ](./assertj.md)）で見るのを優先し、`verify` は副作用の要所だけ。
- **`spy` で本物が動く事故**：`when(spy.foo())` は foo を実際に呼んでしまう。`spy` のスタブは `doReturn(x).when(spy).foo()` 形式が安全。

## 関連
[testing.md](./testing.md) / [junit5.md](./junit5.md) / [service.md](./service.md)
