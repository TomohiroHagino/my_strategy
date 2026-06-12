# Dart

## 一言で
Google の**静的型付け・クライアント最適化**言語。主用途は **Flutter**。開発時は **JIT（ホットリロード）**、本番は **AOT（ネイティブにコンパイル）** という二刀流で、UI開発の快適さとネイティブ性能を両立する。

## 特徴
- **型安全＋健全な null safety**（`String?` のように nullable を型で区別）。
- **AOT / JIT**：開発はホットリロード、本番はネイティブバイナリ。
- **UI記述に向く文法**（カスケード `..`、名前付き引数、コレクションif/for）。
- **非同期**：`Future` / `async`・`await` / `Stream`。
- **isolate**：メモリ共有しない並行処理（スレッドに近いが安全）。

## どういう使い方をするのか
- **Flutter アプリ**（モバイル/Web/Desktop）— ほぼこれ。
- サーバ（`dart:io`・shelf 等）や CLI も書けるが主流ではない。

## 強み / 弱み
- 強み：型安全・null safety・ホットリロード・Flutterと一体で生産性が高い。
- 弱み：Flutter以外の用途・エコシステムは限定的。Web/サーバでは他言語が主流。

## エコシステム・周辺
- パッケージ: `pub`（pub.dev）、整形: `dart format`、静的解析: `dart analyze`
- フレームワーク: **Flutter**（→ [flutter/](./flutter/)）
- テスト: `package:test`

## このフォルダの構成
- [flutter/](./flutter/) … **Flutter**（クロスプラットフォームUIフレームワーク）。フラッグシップの版別リファレンス。
