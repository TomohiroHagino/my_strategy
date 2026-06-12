# 例外ハンドリング（@RestControllerAdvice）（Spring Boot 3）

## ひとことで言うと
アプリ全体で発生した例外を**一箇所でまとめて捕まえ、適切な HTTP ステータスとレスポンスボディに変換する仕組み**。`@RestControllerAdvice` ＋ `@ExceptionHandler` でグローバルに定義する。Boot3 では **`ProblemDetail`（RFC 7807）** が標準のエラー表現。

## 役割・なぜ必要か
- 例外処理を各コントローラに散らすと、try/catch だらけで重複し、レスポンス形式もバラつく。**横断的に1箇所へ集約**することで、エラー時の出力フォーマットを統一できる。
- 「業務例外 → 404」「バリデーション違反 → 400」のような**例外とステータスの対応付け**をルール化できる。
- 何もしないと Spring 既定のエラー（Whitelabel / 既定 JSON）が返り、**本番でスタックトレースや内部情報が漏れる**リスクがある。ここを自前で制御する。
- Spring Framework 6 で `ProblemDetail` が標準化され、エラーレスポンスを **RFC 7807 準拠の機械可読な形式**（`type` / `title` / `status` / `detail` / `instance`）で返せるようになった。

## 基本の書き方（コード）
```java
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.bind.MethodArgumentNotValidException;

@RestControllerAdvice   // 全 @RestController に対するグローバル例外処理
public class GlobalExceptionHandler {

    // 自作の業務例外 → 404
    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource Not Found");
        pd.setProperty("timestamp", java.time.Instant.now());
        return pd;   // RFC 7807 形式の JSON（application/problem+json）で返る
    }

    // 想定外の例外 → 500（内部情報は出さない）
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex) {
        log.error("予期しないエラー", ex);   // 詳細はログにだけ残す
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "サーバー内部でエラーが発生しました");
    }
}
```

```java
// その場で投げるなら ResponseStatusException（軽量。ハンドラ不要）
import org.springframework.web.server.ResponseStatusException;

public User find(Long id) {
    return repo.findById(id).orElseThrow(() ->
        new ResponseStatusException(HttpStatus.NOT_FOUND, "user not found: " + id));
}
```

```java
// 自作業務例外（メッセージと意味を持たせる）
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
```

## 実務での使い方・定番パターン
- **バリデーション違反の整形**：`@Valid` 失敗時の `MethodArgumentNotValidException` を捕捉し、フィールド名→メッセージの形に整える（→ [validation.md](./validation.md)）。
  ```java
  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
      ProblemDetail pd = ProblemDetail.forStatusAndDetail(
          HttpStatus.BAD_REQUEST, "入力値が不正です");
      pd.setTitle("Validation Failed");
      Map<String, String> errors = new LinkedHashMap<>();
      for (FieldError fe : ex.getBindingResult().getFieldErrors()) {
          errors.put(fe.getField(), fe.getDefaultMessage());
      }
      pd.setProperty("errors", errors);   // {"email":"形式が不正", ...}
      return pd;
  }
  ```
- **`@ResponseStatus` でステータス固定**：例外クラス自体に付ければ、ハンドラ無しでもステータスが決まる。
  ```java
  @ResponseStatus(HttpStatus.CONFLICT)
  public class DuplicateEmailException extends RuntimeException { ... }
  ```
- **`ResponseEntityExceptionHandler` の継承**：Spring 既定例外（404・415 等）も自前形式に揃えたい場合、これを継承して該当メソッドを override する。
- **`BindingResult` を引数に取る方法**：コントローラ引数で `@Valid` の直後に `BindingResult` を置くと**例外が投げられず**、メソッド内で `result.hasErrors()` を自分で見られる。ただし基本は例外＋集約ハンドラの方が綺麗。
- **例外の階層化**：`BusinessException`（基底）→ `NotFound` / `Forbidden` のように分け、ハンドラで基底を1つ捕まえる設計にすると拡張しやすい。

## ハマりどころ / アンチパターン
- **例外の握りつぶし**：`catch (Exception e) {}` で何もしない／ログも残さないのは最悪。原因不明の不具合の温床。**必ずログに残すか再送出**する。
- **本番でスタックトレース/内部情報を返す**：例外メッセージや SQL、クラス名をそのままレスポンスに入れると情報漏洩。**詳細はログ、レスポンスは利用者向けの汎用文言**に分ける。
- **`@ExceptionHandler` を `@Controller`（advice でない方）に書く**：そのコントローラ内だけ有効でグローバルにならない。全体で使うなら `@RestControllerAdvice`。
- **`@ControllerAdvice` と `@RestControllerAdvice` の取り違え**：REST API でボディを返すなら `@RestControllerAdvice`（`@ResponseBody` 相当が付く）。`@ControllerAdvice` だとビュー名解決になりがち。
- **より具体的な例外を後回しにする**：`Exception.class` のハンドラが先に一致しないよう、具体的な例外型のハンドラも定義する（Spring は最も近い型を選ぶが、捕捉漏れに注意）。
- **チェック例外を多用してシグネチャを汚す**：Spring では `RuntimeException` 系で投げ、ハンドラで集約するのが一般的。

## 関連: [validation.md](./validation.md) / [controller.md](./controller.md)
