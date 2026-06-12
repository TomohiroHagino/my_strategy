# 品質ゲートとシフトレフト（DevOps）

## ひとことで言うと
**パイプラインの途中に「ここを通らなければ先へ進めない」関門(品質ゲート)を置き、lint/型/テスト/カバレッジ/脆弱性スキャンを自動で検査する。しかも、できるだけ早い段階(左)で検査する＝シフトレフト。**

## 役割・なぜ必要か
- 自動でデプロイが流れる以上、**「壊れたものを通さない仕組み」が無いと事故も自動で量産される**（→ [ci_cd.md](./ci_cd.md)）。
- 人のレビューだけに頼ると見落とす・属人化する。機械的に判定できるものは機械で落とす。
- **早く見つけるほど安い**：開発者の手元で気づく＜PRで気づく＜本番で気づく、の順に修正コストは跳ね上がる。だから検査を左に寄せる(シフトレフト)。
- 変更失敗率を下げ、デプロイ頻度を上げる土台 → [dora_metrics.md](./dora_metrics.md)。

## 基本（手順・考え方・コード）

### 品質ゲートの中身（左から重く）
```
 開発者の手元      PR（CI）              マージ後/デプロイ前
 ─────────────   ─────────────────    ──────────────────
 lint --fix       lint / 型チェック      コンテナイメージスキャン
 ユニットtest     全test + coverage 80%  IaCスキャン(tfsec等)
 (pre-commit)     SAST(コード脆弱性)      DAST(動いてる物を攻撃検査)
                  SCA(依存脆弱性)         署名/プロベナンス
```

### 検査の種類（セキュリティ）
| 種類 | 何を見るか | いつ | 例 |
|---|---|---|---|
| **lint / 型** | 書式・明らかなバグ・型不整合 | 手元〜PR | ESLint, ruff, tsc |
| **SAST** | 自分のコードの脆弱性（静的解析） | PR | CodeQL, Semgrep |
| **SCA** | 依存ライブラリの既知脆弱性 | PR/定期 | `npm audit`, Dependabot, Trivy |
| **コンテナスキャン** | イメージのOSパッケージ脆弱性 | build後 | Trivy, Grype |
| **IaCスキャン** | Terraform等の設定ミス（公開バケット等） | PR | tfsec, Checkov |
| **DAST** | 動いているアプリを外から攻撃検査 | staging | OWASP ZAP |
| **secretスキャン** | 鍵/トークンの混入 | 全commit | gitleaks, truffleHog |

### ゲートの実装（CIで「閾値未満なら失敗」）
GitHub Actions の例（カバレッジ80%ゲート。全体は [ci_tools.md](./ci_tools.md)）:
```yaml
      - run: npm test -- --coverage
      - name: Coverage gate
        run: npx nyc check-coverage --lines 80   # 80%未満なら exit≠0 で失敗
      - name: Dependency vuln gate
        run: npm audit --audit-level=high          # high以上の脆弱性で失敗
      - name: Image scan gate
        run: trivy image --exit-code 1 --severity CRITICAL myapp:${{ github.sha }}
```
> 「失敗したらjobが落ちて、後続のdeployに進めない」のが品質ゲート。`--exit-code`/閾値で**機械的にブロック**する。

### 重大度でブロックか警告かを分ける
```
 CRITICAL/High → ブロック（マージ/デプロイ不可）
 Medium        → 警告（issue化・期限つきで対応）
 Low           → 記録のみ
```
> 全部ブロックにすると開発が止まる。**重大度で扱いを変える**（→ [pitfalls.md](./pitfalls.md) のgate飛ばしを防ぐ）。

### シフトレフト＝検査を前倒しする具体策
```
 pre-commit hook  : commit時にlint/secretスキャンを手元で
 PRテンプレ＋CI    : PR時点で型/test/SAST/SCAを必須化（ブランチ保護）
 Dependabot等     : 依存の脆弱性を定期PRで自動通知
```

## 実務での使い方・定番パターン
- **ブランチ保護で必須化**：「指定したチェックが緑でないとマージ不可」をリポジトリ設定で強制 → [ci_tools.md](./ci_tools.md)。
- **段階導入**：いきなり全ゲートをブロックにせず、まず警告→数字を見て→ブロックに昇格。既存コードのカバレッジ80%は新規差分から適用すると現実的。
- **テストはTDDで前もって書く**（共通ルールの testing.md / tdd-guide）。ゲートに通すためでなく、設計のために書く。
- **IaC/コンテナもスキャン対象**：アプリだけでなく [../インフラ/terraform/](../インフラ/terraform/) と [../インフラ/docker/](../インフラ/docker/) の成果物も検査する。
- **secretスキャンは全員・全commit**。漏れたら即ローテート → [pitfalls.md](./pitfalls.md)。

## ハマりどころ / アンチパターン
- **ゲートをすぐ無効化/スキップする文化**：「赤いけど急ぎだからマージ」が常態化すると、ゲートが無いのと同じ。緊急時の迂回は記録と事後レビュー必須。
- **遅いゲートで開発が詰まる**：重いDAST/E2EをすべてのPRで全部回すと待てない。重い検査はマージ後/夜間に回し、PRは速い検査に絞る。
- **警告の山を放置**：Medium以下を貯め続けると、本物が埋もれる。期限つきで掃除。
- **誤検知(false positive)を放置**：「どうせ嘘」と全員が無視し始める。抑制ルールを整備して信頼を保つ。
- **カバレッジ数字だけ追う**：意味のないテストで80%を満たしても品質は上がらない。挙動を検証するテストを書く。

## 関連: [ci_cd.md](./ci_cd.md) / [ci_tools.md](./ci_tools.md) / [culture.md](./culture.md) / [pitfalls.md](./pitfalls.md) / [dora_metrics.md](./dora_metrics.md)
スキャン対象は [../インフラ/docker/](../インフラ/docker/)（イメージ）/ [../インフラ/terraform/](../インフラ/terraform/)（IaC）
