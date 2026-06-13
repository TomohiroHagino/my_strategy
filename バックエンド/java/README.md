# Java

## 一言で
静的型付け・オブジェクト指向の汎用言語。JVM（Java仮想マシン）上で動く「Write once, run anywhere」。堅牢で枯れており、エンタープライズ・大規模バックエンド・Androidの定番。

## 特徴
- **静的型＋コンパイル**: 実行前に型チェック。大規模でも壊れにくく、IDE補完・リファクタが強力。
- **JVM上で動く**: ソース→バイトコード→JVMが実行。JITで実行時に高速化。Kotlin/Scala も同じJVM。
- **強いOOP＋後方互換重視**: 長期保守に強い。古いコードが動き続ける。
- **近年モダン化（17/21 LTS）**: `record`・`sealed`・パターンマッチ・`var`・**仮想スレッド（Virtual Threads, 21）** など。
- **GC（自動メモリ管理）**。

## どういう使い方をするのか
- **エンタープライズ／大規模バックエンド**（実質 Spring）。金融・基幹系で鉄板。
- **マイクロサービス**（Spring Boot / Cloud）。
- **Android**（近年は Kotlin が主流だが土台は同じ）。
- **ビッグデータ基盤**（Hadoop / Spark / Kafka）。

## 強み
- 型安全で堅牢。大人数・長期保守に向く。
- ツール・IDE・ビルドが成熟（IntelliJ / Maven / Gradle）。
- 性能が高い（JIT）。求人・人材が豊富。

## 弱み・注意点
- 冗長（ボイラープレート）。※`record`等で改善中。
- 起動・メモリが重め。※GraalVM native image で改善。
- 学習コストとエコシステムの広さ。

## 向いている場面 / 向かない場面
- 向いている: 大規模・長期・堅牢性重視のバックエンド、マイクロサービス。
- 向かない: 使い捨てスクリプト、超軽量な小物（そこはPython等が速い）。

## エコシステム・周辺ツール
- ビルド: **Maven** / **Gradle**
- フレームワーク: **Spring / Spring Boot**（→ [spring/](./spring/)）
- ORM: Hibernate（JPA）
- テスト: JUnit 5 / Mockito / Testcontainers
- JSON: Jackson、JVM言語: Kotlin / Scala / Groovy

## ひとことメモ（自分の実感）
- （現役として触った所感を後から追記）

## このフォルダの構成
- [解説/java21.md](./解説/java21.md) … Java 21（LTS）の言語そのものの解説（文法・型・record/sealed/パターンマッチ・Virtual Threads）。
- [spring/](./spring/) … Spring / Spring Boot（フレームワーク）。フラッグシップの版別リファレンス。
