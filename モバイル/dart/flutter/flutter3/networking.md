# 通信（http / dio / JSON）（Flutter 3）

## ひとことで言うと
アプリからWeb API（HTTP）を叩いてデータを取得・送信する仕組み。標準的な軽量パッケージ **`http`** と、高機能な **`dio`** の2択が定番で、やり取りの中身は基本 **JSON** を **`jsonDecode` / `jsonEncode`** で Dart のオブジェクトに変換する。

## 役割・なぜ必要か
- アプリ単体は「画面の入れ物」でしかなく、**実データはほぼサーバー側**にある。一覧取得・ログイン・投稿…すべてHTTP通信が起点。
- 通信は**ネットワーク待ちがある＝非同期**。だから `Future` / `async` / `await` とセットで扱う。→ [async_futures.md](./async_futures.md)
- 受け取ったJSON（`Map<String, dynamic>`）のままだと扱いづらく型安全でもないので、**モデルクラスに `fromJson` / `toJson` を持たせて変換**するのが実務の基本形。

## 基本の書き方（コード）
```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

// 1) モデル（fromJson / toJson は手書き）
class User {
  final int id;
  final String name;
  const User({required this.id, required this.name});

  // JSON(Map) → User
  factory User.fromJson(Map<String, dynamic> json) =>
      User(id: json['id'] as int, name: json['name'] as String);

  // User → JSON(Map)
  Map<String, dynamic> toJson() => {'id': id, 'name': name};
}

// 2) GET：一覧を取得
Future<List<User>> fetchUsers() async {
  final res = await http
      .get(Uri.parse('https://api.example.com/users'))
      .timeout(const Duration(seconds: 10)); // タイムアウト必須

  if (res.statusCode != 200) {
    throw Exception('取得失敗: ${res.statusCode}');
  }
  final List list = jsonDecode(res.body) as List; // String → List/Map
  return list.map((e) => User.fromJson(e as Map<String, dynamic>)).toList();
}

// 3) POST：JSONを送る
Future<User> createUser(String name) async {
  final res = await http.post(
    Uri.parse('https://api.example.com/users'),
    headers: {'Content-Type': 'application/json'},
    body: jsonEncode({'name': name}), // Map → String
  );
  return User.fromJson(jsonDecode(res.body) as Map<String, dynamic>);
}
```

```yaml
# pubspec.yaml（依存を追加。後で flutter pub get）
dependencies:
  http: ^1.2.0
  # dio を使うなら:
  dio: ^5.4.0
```

## 実務での使い方・定番パターン
- **`http` か `dio` か**：単純なGET/POSTだけなら `http` で十分。**インターセプタ（共通ヘッダ・トークン付与・ログ）／自動リトライ／タイムアウト設定／FormData** が要るなら `dio`。
- `dio` の基本：
```dart
import 'package:dio/dio.dart';

final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',
  connectTimeout: const Duration(seconds: 10),
));

Future<List<User>> fetchUsers() async {
  final res = await dio.get('/users'); // res.data は既にデコード済み
  return (res.data as List)
      .map((e) => User.fromJson(e as Map<String, dynamic>))
      .toList();
}
```
- **コード生成（`json_serializable`）**：フィールドが多いモデルは `fromJson`/`toJson` 手書きが事故りやすい。`json_serializable` + `build_runner` で自動生成する。
```yaml
dependencies:
  json_annotation: ^4.9.0
dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.8.0
```
```dart
import 'package:json_annotation/json_annotation.dart';
part 'user.g.dart'; // 生成ファイル

@JsonSerializable()
class User {
  final int id;
  final String name;
  User({required this.id, required this.name});
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);
}
// 生成コマンド: dart run build_runner build --delete-conflicting-outputs
```
- 通信ロジックは**画面（Widget）から分離**し、`Repository` / `ApiClient` クラスに閉じ込める。UIは結果だけ受け取る形にするとテストしやすい。

## ハマりどころ / アンチパターン
- **`pubspec.yaml` に追加し忘れ／`flutter pub get` 忘れ**で `import` が解決できない。追加したら必ず取得する。
- **JSONシリアライズの取り違え**：`jsonDecode` は `String → Map/List`、`jsonEncode` は逆。`response.body`（String）を直接 `User.fromJson` に渡すと型エラー。必ず一度 `jsonDecode` する。
- **`fromJson` の型キャスト漏れ**：`json['id']` は `dynamic`。`as int` 等を付けないと実行時に `null` や型不一致でクラッシュ。null許容なら `int?` で受ける。
- **`json_serializable` で `part 'xxx.g.dart';` を書き忘れ／生成し忘れ**。ビルドエラーになる。コマンド実行を忘れない。
- **タイムアウト未設定**：圏外・サーバー無応答で**永久に待つ**。`.timeout(...)`（http）や `connectTimeout`（dio）を必ず付ける。
- **エラー処理の握り潰し**：`statusCode` を見ずに `200` 前提でパースすると、4xx/5xx のエラーJSONを正常データとして読んで誤動作。`try/catch` で `SocketException`（圏外）・`TimeoutException` も拾い、UIにエラー表示する。
- **UIスレッドで巨大JSONをパース**して画面がカクつく → 大きいデータは `compute()`（別Isolate）でデコードする。

## 関連
[async_futures.md](./async_futures.md) / [packages_pubspec.md](./packages_pubspec.md) / [state_management.md](./state_management.md)
