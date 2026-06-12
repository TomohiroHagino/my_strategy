# エンジニア実務大全 — 作って動かして運用するための技術リファレンス

自分が触れる **言語・フレームワークから、インフラ・運用(SRE)・DevOps まで**を、
**バックエンド / フロントエンド / モバイル / インフラ / DevOps** の5分野に分けてまとめる、実務向けの個人ナレッジベース。
（＝言語/FWだけでなく、コンテナ・IaC・障害対応・負荷検証・CI/CD など「作って動かして運用する」全体をカバー）

## 構成（全体の設計図）

```
006_engineer_learning_list/
├── README.md            ← これ（構成＋各ファイルの書式）
├── バックエンド/
│   ├── ruby/            ✅ Ruby（フォルダ化・深掘り）
│   │   ├── README.md            ← Ruby言語の概要
│   │   ├── rails/               ← Rails（版別＋周辺インフラ）
│   │   │   ├── README.md        ← Rails概要
│   │   │   ├── rails4/ rails5/ rails6/ rails7/ ← Rails 4〜7 版別リファレンス（各完成）
│   │   │   └── 周辺インフラ/     ← Redis / Sidekiq / Solid Queue / Unicorn …
│   │   ├── sinatra/             ← Sinatra
│   │   │   └── sinatra4/        ← Sinatra 4 実務リファレンス（フラッグシップ・完成）
│   │   └── hanami/              ← Hanami（モダンな代替FW・DI/ROM）
│   │       └── hanami2/         ← Hanami 2 実務リファレンス（フラッグシップ・完成）
│   ├── php/             ✅ PHP（フォルダ化・深掘り）
│   │   ├── README.md            ← PHP言語の概要
│   │   ├── laravel/             ← Laravel
│   │   │   ├── README.md        ← Laravel概要
│   │   │   └── laravel11/       ← Laravel 11 実務リファレンス（フラッグシップ・完成）
│   │   └── codeigniter/         ← CodeIgniter
│   │       ├── README.md        ← CodeIgniter概要
│   │       └── codeigniter4/    ← CodeIgniter 4 実務リファレンス（完成）
│   ├── python/          ✅ Python（フォルダ化・深掘り）
│   │   ├── README.md            ← Python言語の概要
│   │   ├── 環境管理.md          ← venv/conda/Docker の棲み分け（図解）
│   │   ├── django/              ← Django
│   │   │   ├── README.md        ← Django概要
│   │   │   └── django5/         ← Django 5 実務リファレンス（フラッグシップ・完成）
│   │   └── fastapi/             ← FastAPI
│   │       └── fastapi0/        ← FastAPI 実務リファレンス（フラッグシップ・完成 / 0.x系）
│   ├── java/            ✅ Java（フォルダ化・深掘り）
│   │   ├── README.md            ← Java言語の概要
│   │   └── spring/              ← Spring / Spring Boot
│   │       ├── README.md        ← Spring概要
│   │       └── springboot3/     ← Spring Boot 3 実務リファレンス（フラッグシップ・完成）
│   ├── node/            ✅ Node.js（フォルダ化・深掘り）
│   │   ├── README.md            ← Node.js の概要
│   │   ├── express/             ← Express
│   │   │   ├── README.md        ← Express概要
│   │   │   └── express5/        ← Express 5 実務リファレンス（フラッグシップ・完成）
│   │   └── hono/                ← Hono（マルチランタイム・Edge）
│   │       ├── README.md        ← Hono概要
│   │       └── hono4/           ← Hono v4 実務リファレンス（フラッグシップ・完成）
│   └── go/              ✅ Go（フォルダ化・深掘り）
│       ├── README.md            ← Go言語の概要
│       └── gin/                 ← Gin
│           ├── README.md        ← Gin概要
│           └── gin1/            ← Gin v1 実務リファレンス（フラッグシップ・完成）
├── フロントエンド/
│   ├── README.md          ← フロントエンド索引（言語→ビルド→FW）
│   ├── build_tools.md     ← ビルドツール（Vite/esbuild/Rollup/Turbopack）
│   ├── javascript/      ✅ JavaScript（言語の概要）
│   ├── typescript/      ✅ TypeScript（言語の概要）
│   ├── react/           ✅ React
│   │   └── react19/     ← React 18/19 実務リファレンス（フラッグシップ・完成）
│   ├── nextjs/          ✅ Next.js
│   │   ├── nextjs15/    ← Next.js 15（App Router）実務リファレンス（完成）
│   │   └── nextjs12/    ← Next.js 12（Pages Router）実務リファレンス（完成）
│   ├── vue/             ✅ Vue
│   │   └── vue3/        ← Vue 3 実務リファレンス（フラッグシップ・完成）
│   ├── nuxt/            ✅ Nuxt
│   │   └── nuxt3/       ← Nuxt 3 実務リファレンス（フラッグシップ・完成）
│   └── rust/            ✅ Rust/WASM（フロントエンド）
│       └── leptos/      ← Leptos
│           └── leptos0/ ← Leptos 0.x 実務リファレンス（フラッグシップ・完成）
├── モバイル/
│   ├── dart/            ✅ Dart
│   │   └── flutter/     ← Flutter / flutter3（＋ android_studio.md）
│   ├── react_native/    ✅ React Native / react_native0（＋ android_studio.md）
│   ├── swift/           ✅ Swift（＋ xcode.md）
│   │   └── swiftui/     ← SwiftUI / swiftui6（フラッグシップ・完成）
│   └── kotlin/          ✅ Kotlin（＋ android_studio.md）
│       └── compose/     ← Jetpack Compose / compose1（フラッグシップ・完成）
├── インフラ/            ✅ クラウド〜運用まで（7本柱）
│   ├── aws/             ← AWS（IAM/EC2/Lambda/S3/RDS/VPC/ECS/ECR… 完成）
│   ├── gcp/             ← GCP（IAM/Compute/Cloud Run/GCS/BigQuery… 完成）
│   ├── terraform/       ← IaC（state/modules… ＋ ECS構成の実例 完成）
│   ├── webサーバ/       ← nginx/Apache 詳細（仕組み/nginx/apache/リバプロ/アプリサーバ/チューニング/セキュリティ/ログ/比較 完成）
│   ├── docker/          ← コンテナ（Dockerfile/Compose/本番運用 完成）
│   ├── 障害対応/         ← SRE（切り分け/対応判断フロー/復旧/ポストモーテム 完成）
│   └── 負荷検証/         ← 性能（指標/k6/飽和点/チューニング 完成）
└── DevOps/              ✅ 上を"つなぐ"上位レイヤ（完成）
    ├── ci_cd.md / ci_tools.md       ← CI/CD・GitHub Actions の例
    ├── deploy_strategies.md         ← rolling / blue-green / canary / flag
    ├── dora_metrics.md              ← DORA 4 Keys（デプロイ頻度/リードタイム/MTTR/変更失敗率）
    ├── gitops.md                    ← GitOps（Argo CD / Flux）
    ├── observability.md             ← 可観測性（metrics/logs/traces・SLI/SLO）
    ├── automation_quality.md        ← 品質ゲート・シフトレフト
    ├── culture.md                   ← CALMS・トランクベース・blameless
    └── pitfalls.md                  ← 罠まとめ
```

> モバイルは **Flutter / React Native / SwiftUI / Jetpack Compose** の4種を整備済み。
> 各フレームワークには `getting_started.md`（深いフォルダツリー付き）と `request_flow.md`（リクエスト/処理の流れ・各層が何を返すか）を横断で用意。Rails は 4〜7、Next.js は App/Pages Router の複数版を独立管理。

## 各ファイルの書式（テンプレ）

書き方は2段階:
- **軽い概要**（言語/FWの「特徴・使い方」1ファイル）= 下のテンプレ。`ruby/README.md`・`php/README.md` が見本。新しい言語はまずこれで足す。
- **フラッグシップ深掘り**（項目=1ファイルで「○○とは」を実務粒度）= `ruby/rails/rails7/`・`php/laravel/laravel11/` が見本。本格的に書く版だけこの形にする。

```markdown
# {言語 or フレームワーク名}

## 一言で
（何者か。1〜2行）

## 特徴
- （設計思想・型・特徴的な仕組みを箇条書き）

## どういう使い方をするのか
- （典型的に何を作るか／ユースケース）

## 強み
- 

## 弱み・注意点
- 

## 向いている場面 / 向かない場面
- 向いている: 
- 向かない: 

## エコシステム・周辺ツール
- （代表的なライブラリ・ツール）

## ひとことメモ（自分の実感）
- （現役として触った所感。後から追記する欄）
```
