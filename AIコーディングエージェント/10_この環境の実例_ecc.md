# この環境の実例（ECC 設定）── 詳細版

## ひとことで言うと
**この PC に入っている Claude Code 設定そのものの全解説**。中身は **ECC（Everything Claude Code）という"全部入り"プラグイン**で、01〜09で説明した機能（コマンド・サブエージェント・フック・ルール）が**実際にどう構成されているか**の生きた見本。特に **rules/（言語別の規約集）を全12言語ぶん**、どのツール・どのテストFW・どの慣用が設定されているかまで具体的に書き出す。

## なぜ必要か / どこで効くか
- 01〜09 は「**何ができるか**」の一般論。ここは「**この環境では実際どうなっているか**」の答え合わせ。
- 自分の環境を**読み解ける**ようになると、挙動の理由（なぜ整形が走る・なぜ push 前に確認が出る・なぜ Go だと `%w` でラップしろと言われる）が腑に落ち、**壊さず育てられる**。
- ＝ 既製の優れた設定を**スケルトンとして借り、その構造の中で理解を深める**やり方（このリポジトリの「骨格を借りて学ぶ」方針そのもの）。

> ECC = 公式の機能（プラグイン／フック／サブエージェント／ルール）を**実戦的にまとめたプラグイン**。`~/.claude/` にユーザー全体設定として入っている（→ [05_設定とメモリ.md](./05_設定とメモリ.md) の置き場所そのもの）。

## 実際の構成（この環境で観測した中身）
プラグイン **`ecc` v1.10.0**。`~/.claude/` 直下は概ねこうなっている。
```
~/.claude/
├── settings.json         … フック登録の本体（後述）／ tui 設定
├── plugin.json           … プラグイン定義（name: ecc, version: 1.10.0）
├── AGENTS.md / README.md … プラグインの解説
├── agents/   (47個)      … サブエージェント定義（→ 03）
├── commands/ (79個)      … スラッシュコマンド（→ 02）
├── skills/   (19種/32ファイル) … Skill 群（→ 02）
├── rules/                … コーディング規約集（common＋12言語＋web＋zh）★本章の主役
├── scripts/hooks/  (30本) … フックの実体スクリプト（→ 04）
├── session-data/ (78)    … セッション保存（Stop/SessionEnd フックの成果物）
├── bash-commands.log     … 実行コマンド記録（PostToolUse フック・約5,200行）
├── cost-tracker.log      … コスト記録（フック・約5,200行）
├── history.jsonl         … 履歴（約3,900行）
└── ecc/, plugins/, mcp-configs/, telemetry/, metrics/, backups/ …
```
- 数は**この環境で実際に入っている数**（選択インストールのため、プラグイン公称値とは一致しないことがある）。

---

## rules/ の仕組み（最重要）── path で自動発動する「言語別の基準」
`rules/` は会話の前提として読まれる規約集。2つの仕掛けで効く。

### ① path スコープ：触っているファイルで自動的に効く
各言語ルールの先頭には **frontmatter で `paths:`（対象グロブ）** が書かれている。例：
```yaml
# rules/golang/coding-style.md の冒頭
---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
```
→ **`.go` を触っているときだけ Go の規約が効く**。Python を触れば Python の規約に切り替わる。CLAUDE.md のように「常時全部」ではなく、**ファイル種別で自動的に出し分け**られているのがポイント。

### ② common → 言語の上書き（「特定 > 一般」）
```
rules/common/*    … 言語非依存の既定（全プロジェクト共通）
     ▲ 上書き
rules/{言語}/*    … その言語の慣用で common を上書き
```
- 各言語ファイルは必ず `> This file extends common/xxx.md` で始まり、**common を土台に言語固有を足す/覆す**構造。
- 例：common は「不変性を既定」とするが、それを **Go はポインタ慣用、Rust は所有権**…と言語側が具体化する。

---

## rules/common/ ── 全10ファイル（言語非依存の土台）
| ファイル | 何を定めるか |
|---|---|
| `coding-style.md` | 不変性既定・KISS/DRY/YAGNI・小さいファイル(200-400行/800上限)・命名・エラー処理 |
| `testing.md` | **カバレッジ80%最低**・unit/integration/E2E・TDD(RED→GREEN→REFACTOR)・AAA |
| `security.md` | コミット前チェック（秘密・入力検証・SQLi/XSS/CSRF・レート制限）・秘密管理 |
| `code-review.md` | レビュー必須トリガ・重大度(CRITICAL/HIGH/MEDIUM/LOW)・言語別 reviewer 割当 |
| `git-workflow.md` | コミット形式(`type: desc`)・PR は全履歴と `diff base...HEAD` で要約 |
| `development-workflow.md` | 研究→計画→TDD→レビュー→コミットの開発パイプライン（既存実装の再利用優先） |
| `agents.md` | サブエージェント一覧・即時起動条件・**並列実行を常に使う**・多視点レビュー |
| `performance.md` | モデル選択戦略（Haiku/Sonnet/Opus）・コンテキスト窓管理・拡張思考＋plan |
| `patterns.md` | スケルトン流用・Repository パターン・API レスポンス封筒 |
| `hooks.md` | フック種別（Pre/Post/Stop）・自動承認の注意・TodoWrite 活用 |

---

## rules/{言語}/ ── 全12言語の設定（各5ファイル：coding-style / hooks / patterns / security / testing）
**「整形 / Lint・静的解析 / 型・ビルド / テストFW / カバレッジ / 特記の慣用」** を1言語1表で。**hooks 列が実際に PostToolUse で走り得るツール**。

### Go（`**/*.go` `go.mod` `go.sum`）
| 項目 | 設定 |
|---|---|
| 整形 | **gofmt / goimports**（必須・議論しない） |
| Lint・静的解析 | **go vet** / **staticcheck** |
| テスト | `go test` ＋**テーブル駆動**・**`-race` 必須** |
| カバレッジ | `go test -cover ./...` |
| 慣用 | 「**インターフェースを受け取り構造体を返す**」/ 小さいIF(1-3メソッド)/ エラーは `fmt.Errorf("...: %w", err)` でラップ |

### Python（`**/*.py` `**/*.pyi`）
| 項目 | 設定 |
|---|---|
| 整形 | **black** / **isort**（import 整列） |
| Lint | **ruff** |
| 型 | **mypy / pyright** |
| テスト | **pytest**（`@pytest.mark.unit/integration`） |
| カバレッジ | `pytest --cov=src --cov-report=term-missing` |
| 慣用 | PEP 8 / 全シグネチャに型注釈 / `@dataclass(frozen=True)`・`NamedTuple` / **`print()` は警告**（logging へ） |

### TypeScript / JavaScript（`**/*.ts` `tsx` `js` `jsx`）
| 項目 | 設定 |
|---|---|
| 整形 | **Prettier** |
| 型 | **tsc**（編集後に型チェック） |
| テスト | E2E は **Playwright**（`e2e-runner` 連携）・unit は共通80% |
| 慣用 | 公開APIに明示型 / `interface` vs `type` 使い分け / **`any` 禁止→`unknown` で絞る** / **Zod** で入力検証 / spread で不変更新 / `React.FC` 不使用 / **`console.log` 禁止（Stop フックで全変更ファイルを監査）** |

### Rust（`**/*.rs` `Cargo.toml`）
| 項目 | 設定 |
|---|---|
| 整形 | **rustfmt**（`cargo fmt`・100桁） |
| Lint | **clippy**（`cargo clippy -- -D warnings`＝警告をエラー扱い） |
| ビルド | `cargo check`（build より速い） |
| テスト | `#[test]`/`#[cfg(test)]` ＋ **rstest** / **proptest** / **mockall** / `#[tokio::test]` |
| カバレッジ | **cargo-llvm-cov**（`--fail-under-lines 80`） |
| 慣用 | 既定不変(`let`)/ 借用優先 / `Result`＋`?`・**`unwrap()` 禁止** / lib は **thiserror**・app は **anyhow** / ドメインでモジュール分割 |

### Java（`**/*.java` `pom.xml` `build.gradle(.kts)`）
| 項目 | 設定 |
|---|---|
| 整形 | **google-java-format** / **Checkstyle** |
| ビルド | `./mvnw compile` / `./gradlew compileJava` |
| テスト | **JUnit 5** ＋ **AssertJ** ＋ **Mockito** ＋ **Testcontainers** |
| カバレッジ | **JaCoCo** 80%+ |
| 慣用 | `record`/ `final` 既定 / 防御的コピー(`List.copyOf`) / sealed・pattern matching・switch式 / finder は `Optional`（`get()` 禁止）/ ドメイン例外は `RuntimeException` |

### Kotlin（`**/*.kt` `kts` `build.gradle.kts`）
| 項目 | 設定 |
|---|---|
| 整形 | **ktfmt / ktlint**（`kotlin.code.style=official`） |
| 静的解析 | **detekt** |
| ビルド | `./gradlew build` |
| テスト | **kotlin.test / JUnit** ＋ **Turbine**（Flow）＋ **kotlinx-coroutines-test**（`runTest`）/ モックより手書き fake |
| 慣用 | `val` 優先 / `data class` / **`!!` 禁止**（`?.`,`?:`,`requireNotNull`）/ sealed＋網羅 `when`（`else` なし）/ scope 関数使い分け / **`CancellationException` は catch せず再送出** |

### Swift（`**/*.swift` `Package.swift`）
| 項目 | 設定 |
|---|---|
| 整形 | **SwiftFormat**（Xcode16+ は `swift-format` も） |
| Lint | **SwiftLint** |
| ビルド | `swift build`（型チェック） |
| テスト | **Swift Testing**（`import Testing` / `@Test` / `#expect`） |
| カバレッジ | `swift test --enable-code-coverage` |
| 慣用 | `let` 優先 / `struct` 値型既定 / **typed throws（Swift6）** / Swift6 strict concurrency（`Sendable`・actor・構造化並行）/ `print()` 警告 |

### PHP（`**/*.php` `composer.json` `phpstan.neon` `psalm.xml`）
| 項目 | 設定 |
|---|---|
| 整形 | **PHP-CS-Fixer / Laravel Pint**（PSR-12） |
| 静的解析 | **PHPStan / Psalm** |
| テスト | **PHPUnit**（Pest があれば Pest） |
| カバレッジ | `phpunit --coverage-text` / `pest --coverage`（pcov/Xdebug・CIで閾値） |
| 慣用 | `declare(strict_types=1)` / `readonly` DTO・値オブジェクト / 例外で失敗を表す / 入力は検証済み DTO に変換 / **`var_dump`/`dd`/`dump`/`die()` 残置を警告**・生SQL/CSRF無効化を警告 |

### Dart / Flutter（`**/*.dart` `pubspec.yaml` `analysis_options.yaml`）
| 項目 | 設定 |
|---|---|
| 整形 | **dart format**（80桁・末尾カンマ・CIは `--set-exit-if-changed`） |
| 静的解析 | **dart analyze**（`--fatal-infos`） |
| テスト | **flutter_test / dart:test** ＋ mockito/mocktail ＋ **bloc_test** ＋ fake_async ＋ **integration_test** ＋ **golden** |
| カバレッジ | `flutter test --coverage`（lcov）80%+・全状態遷移をテスト |
| 慣用 | `final`/`const` / **`!` 回避**・`late` 慎重 / **sealed＋網羅 switch** / `package:` import（相対禁止）/ codegen(`.g.dart`)は手編集禁止 |

### C++（`**/*.cpp` `hpp` `cc` `CMakeLists.txt` 等）
| 項目 | 設定 |
|---|---|
| 整形 | **clang-format**（`-i`） |
| 静的解析 | **clang-tidy** / **cppcheck** |
| ビルド | **cmake --build** |
| テスト | **GoogleTest**（gtest/gmock）＋ **CTest** |
| カバレッジ | **lcov**（`--coverage`）・CIは **sanitizer（ASan/UBSan）**併用 |
| 慣用 | Modern C++17/20/23 / **RAII 徹底（手動 new/delete 禁止）** / `unique_ptr`・`make_unique` / 構造化束縛 |

### C#（`**/*.cs` `csproj` `sln` 等）
| 項目 | 設定 |
|---|---|
| 整形 | **dotnet format**（＋アナライザ修正・**nullable 参照型 ON**） |
| ビルド | **dotnet build** |
| テスト | **xUnit** ＋ **FluentAssertions** ＋ **Moq/NSubstitute** ＋ **Testcontainers**（API は `WebApplicationFactory`） |
| カバレッジ | `dotnet test`（80%+・ドメイン/検証/認証/失敗系に集中） |
| 慣用 | `record`/`record struct`・`init` setter / `async`/`await`＋`CancellationToken`（`.Result`/`.Wait()` 禁止）/ `appsettings*.json` の秘密混入を警告 |

### Perl（`**/*.pl` `pm` `t` `psgi` `cgi`）
| 項目 | 設定 |
|---|---|
| 整形 | **perltidy**（`-i=4 -l=100 -ce -bar`） |
| Lint | **perlcritic**（severity 3・`core/pbp/security`） |
| テスト | **Test2::V0**（Test::More 非推奨）・`prove -l` |
| カバレッジ | **Devel::Cover**（`cover -test`）80%+ |
| 慣用 | `use v5.36`（strict/warnings/署名）/ サブルーチン署名（`@_` 手動展開禁止）/ `say` / **Moo `is=>'ro'`＋`Types::Standard`**・`print` を非スクリプト `.pm` で警告 |

> 注：hooks 列のツールは「**設定上 PostToolUse で走り得る**」もの。実際に発火するかは `settings.json` の登録と `check-hook-enabled`（個別ON/OFF）による。各言語ファイルは末尾で対応する **Skill**（`golang-patterns`・`rust-testing`・`java-coding-standards` 等）も指している。

---

## rules/web/ ── フロント特化（7ファイル）
`coding-style`（CSS変数・semantic HTML・compositor向けアニメ）/ `design-quality`（**アンチテンプレ方針**・必須品質10項目）/ `performance`（Core Web Vitals 目標・バンドル予算）/ `security`（CSP・XSS・SRI・各種ヘッダ）/ `patterns`（compound components・状態管理の分離・URL as state）/ `hooks`（prettier/eslint/tsc/stylelint）/ `testing`（視覚回帰・a11y・Lighthouse・レスポンシブ）。

## rules/zh/ ── common の中国語訳（11ファイル）
`common/` 10本の中国語版＋README。内容は英語版と一致（多言語チームでも同じ基準を共有できる）。

---

## agents/ ── 全47サブエージェント（→ [03_サブエージェント.md](./03_サブエージェント.md)）
| 分類 | エージェント |
|---|---|
| 言語別レビュー | `cpp-reviewer` `csharp-reviewer` `go-reviewer` `java-reviewer` `kotlin-reviewer` `python-reviewer` `rust-reviewer` `typescript-reviewer` `flutter-reviewer` `database-reviewer` `healthcare-reviewer` ＋汎用 `code-reviewer` |
| ビルド/エラー修正 | `build-error-resolver` `cpp-build-resolver` `dart-build-resolver` `go-build-resolver` `java-build-resolver` `kotlin-build-resolver` `rust-build-resolver` `pytorch-build-resolver` |
| 設計/計画/探索 | `architect` `code-architect` `planner` `code-explorer` |
| 品質/分析 | `code-simplifier` `comment-analyzer` `type-design-analyzer` `silent-failure-hunter` `pr-test-analyzer` `refactor-cleaner` `performance-optimizer` `security-reviewer` |
| テスト | `tdd-guide` `e2e-runner` |
| ドキュメント | `doc-updater` `docs-lookup` |
| GAN ハーネス | `gan-planner` `gan-generator` `gan-evaluator`（生成↔評価を回す） |
| OSS 公開 | `opensource-forker` `opensource-sanitizer` `opensource-packager` |
| ハーネス運用 | `harness-optimizer` `loop-operator` `conversation-analyzer` |
| 専門/その他 | `chief-of-staff`（通信トリアージ） `seo-specialist` |

## commands/ ── 全79スラッシュコマンド（→ [02_スラッシュコマンドとスキル.md](./02_スラッシュコマンドとスキル.md)）
| 分類 | コマンド |
|---|---|
| 言語別 build/review/test | `cpp-build/review/test` `go-build/review/test` `kotlin-build/review/test` `rust-build/review/test` `flutter-build/review/test` `python-review` `gradle-build` |
| レビュー/PR | `code-review` `review-pr` |
| 計画/開発 | `plan` `feature-dev` `prp-prd` `prp-plan` `prp-implement` `prp-pr` `prp-commit` |
| TDD/品質ゲート | `tdd` `test-coverage` `quality-gate` `verify` `build-fix` |
| マルチモデル協調 | `multi-plan/backend/frontend/execute/workflow` `model-route` |
| 学習/instinct | `learn` `learn-eval` `evolve` `instinct-export/import/status` `projects` `promote` `prune` |
| セッション/文脈 | `checkpoint` `save-session` `resume-session` `sessions` `context-budget` |
| hookify | `hookify` `hookify-configure` `hookify-list` `hookify-help` |
| ループ/オーケスト | `loop-start` `loop-status` `orchestrate` `santa-loop` `devfleet` `claw` |
| ドキュメント/メタ | `docs` `update-docs` `update-codemaps` `skill-create` `skill-health` `rules-distill` `harness-audit` `agent-sort` |
| 環境/雑多 | `pm2` `setup-pm` `aside` `prompt-optimize` `jira` `eval` `e2e` `refactor-clean` `gan-build` `gan-design` |

## skills/ ── 19種（→ [02_スラッシュコマンドとスキル.md](./02_スラッシュコマンドとスキル.md)）
`tdd-workflow` `verification-loop` `e2e-testing` `council`（多声議論）`eval-harness` `hookify-rules` `iterative-retrieval` `strategic-compact` `continuous-learning(-v2)` `agent-introspection-debugging` `ai-regression-testing` `code-tour` `skill-stocktake` `plankton-code-quality` `configure-ecc` `agent-sort` `omarchy`（Linux デスクトップ設定）`learned`。
> ※ これに加え、`plugins/` 経由でさらに多くの Skill/コマンド（`deep-research` 等）が利用可能。skills/ 直下の"自前 Skill"は 19種。

---

## hooks ── 実際に"強制"されている自動化（→ [04_フック.md](./04_フック.md)）
`settings.json` に多数登録され、実体は `scripts/hooks/*.js`（30本）。観測された発火種別と件数：
```
PreToolUse 11 / PostToolUse 10 / Stop 6 / PreCompact 1 / SessionStart 1 / PostToolUseFailure 1 / SessionEnd 1
```
代表的なスクリプトと役割（＝この環境が"勝手にやってくれている"こと）：
| タイミング | スクリプト（例） | 効果 |
|---|---|---|
| **PreToolUse** | `block-no-verify` | `git commit --no-verify`（フック迂回）を**ブロック** |
| | `pre-bash-commit-quality` | コミット前に品質ゲート |
| | `pre-bash-git-push-reminder` | push 前にリマインド |
| | `pre-bash-dev-server-block` | フォアグラウンドの dev サーバ起動を抑止（背景実行へ誘導） |
| | `pre-bash-tmux-reminder` / `auto-tmux-dev` | tmux 運用へ誘導 |
| | `config-protection` | 設定ファイルの保護 |
| **PostToolUse** | `post-edit-format` | 編集後に**自動整形**（言語別 rules のツール） |
| | `post-edit-typecheck` | 編集後に**型チェック**（tsc/mypy 等） |
| | `check-console-log` / `post-edit-console-warn` | `console.log` 混入を検出 |
| | `design-quality-check` | フロントの品質チェック（web rules 連動） |
| | `pre-write-doc-warn` / `doc-file-warning` | ドキュメント作成時の警告 |
| | `post-bash-command-log` / `cost-tracker` | 実行コマンド・コストを記録（`*.log`） |
| | `post-bash-build-complete` / `post-bash-pr-created` / `desktop-notify` | 完了/PR作成の通知 |
| **PostToolUseFailure** | `run-with-flags` | 失敗時の処理 |
| **SessionStart** | `session-start-bootstrap` | 起動時に**前回セッションの要約を注入**（このセッション冒頭で実際に起きたあれ） |
| **Stop / SessionEnd** | `session-end` / `evaluate-session` / `governance-capture` / `quality-gate` | 終了時に**セッション保存・自己評価・品質ゲート**（`session-data/` に蓄積） |
| **PreCompact** | `pre-compact` | 圧縮前の処理 |
| 随時 | `mcp-health-check` | MCP サーバの死活確認 |

- つまり 04 で説明した「**お願いでなくハーネスで強制**」が、この環境では **コミット品質・整形・型・console.log監査・ログ・セッション保存** として現に効いている。
- `bash-commands.log` / `cost-tracker.log` が各約5,200行たまっているのは、PostToolUse フックが毎コマンド記録している証拠。
- 個別の ON/OFF は `check-hook-enabled` の仕組みで制御できる（重い/邪魔なものだけ無効化）。

## このリポジトリ自体との関係
- この学習リストの執筆・整理も、上記フック群（整形・コミット品質ゲート・セッション保存）が効いた状態で行われている。
- 「前回の続きから書ける」「コミット時に品質チェックが入る」のは、**プロンプトの賢さではなくこの設定のおかげ**。機能と現物が一対一で対応している好例。
- このリポジトリは主に日本語のドキュメントなので、上記の言語別 rules（Go/Python…）が**直接効く場面は少ない**が、`doc-file-warning` などドキュメント系フックは効いている。

## 強み / 弱み・限界
**強み**：
- **ベストプラクティスが初期装備**：12言語ぶんのレビュー・テスト・整形・記録が最初から効く。
- 概念（01〜09）の**生きた参照実装**として読める。
- 言語別 rules / reviewer / build-resolver が揃い、**多言語に一貫した品質**。

**弱み・限界**：
- **全部入りゆえ重い・把握しづらい**：何が効いているか追うのに本章のような地図が要る。
- **借り物設定はブラックボックス化しやすい**：理由を理解せず使うと、誤ブロック時に直せない。
- フックが多い＝**動作が遅くなる/誤発火する**余地。`check-hook-enabled` で個別 ON/OFF できる設計。
- ルールは `paths:` で発動するので、**対象外の拡張子では効かない**（例：このリポの `.md` には言語別ルールは効かない）。

## ハマりどころ / アンチパターン
- **「なぜか整形/確認/型チェックが走る」を不思議がる** → それは ECC のフック＋言語別 rules。`scripts/hooks/` と `rules/{言語}/` を見れば理由が分かる。
- **言語の慣用に逆らう実装をして弾かれる** → Go で `%w` ラップ無し、Rust で `unwrap()`、Kotlin で `!!`… は**各言語ルールが明示的に禁止/警告**している。ルールを読む。
- **設定を読まずに上書き** → `config-protection` フックが守っているが、構造を理解してから触る。
- **フックを邪魔だと全切り** → 事故防止機能まで失う。重い/誤発火するものだけ個別に無効化。
- **公称スペックを鵜呑み** → 実際に入っている数・内容は**選択インストールで変わる**。現物（`ls ~/.claude/...`）で確認する。
- **秘密情報の扱い** → `settings.json`/ログに認証情報を残さない。カスタムゲートウェイは `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` を環境変数で（→ [05_設定とメモリ.md](./05_設定とメモリ.md)）。

## 関連
- 置き場所と優先順位の一般論 → [05_設定とメモリ.md](./05_設定とメモリ.md)
- フックの仕組み → [04_フック.md](./04_フック.md)
- サブエージェント → [03_サブエージェント.md](./03_サブエージェント.md)
- コマンド/Skill → [02_スラッシュコマンドとスキル.md](./02_スラッシュコマンドとスキル.md)
- 全体原則 → [09_ベストプラクティス.md](./09_ベストプラクティス.md)
- 各言語の中身そのもの（文法・慣用） → [../バックエンド/](../バックエンド/) ／ [../フロントエンド/](../フロントエンド/) ／ [../言語共通概念/](../言語共通概念/)
- 実体は `~/.claude/`（`rules/` `scripts/hooks/` `README.md` `AGENTS.md` を読むと詳細が分かる）
