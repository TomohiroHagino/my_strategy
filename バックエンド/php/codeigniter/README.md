# CodeIgniter（PHP の軽量MVCフレームワーク）

## 一言で
**軽量・高速・学習が容易**なPHPのMVCフレームワーク。Laravelほど“全部入り・規約多め”ではなく、**シンプルで素直**。小〜中規模・レンタルサーバ運用・既存PHP資産との相性が良い。現行は **CodeIgniter 4**（PHP 8系・名前空間/Composer対応の全面刷新）。

## 特徴
- **軽量・高速・低依存**：フットプリントが小さく動かしやすい。
- **MVC**：Controller / Model / View ＋ 強力な **Query Builder**。
- **設定より明示**：Laravelの“魔法”より、何が起きているか分かりやすい寄り。
- **CI3→CI4で全面刷新**：名前空間・Composer・PSR準拠・`spark` CLI。

## どういう使い方をするのか
- 小〜中規模の業務Web・社内ツール・既存PHPサイトの刷新。
- 共有ホスティング等、軽さ・手軽さが効く環境。

## 強み / 弱み
- 強み：軽い・速い・学習コスト低・分かりやすい。
- 弱み：エコシステム/求人はLaravelに劣る。大規模・高度な要件はLaravel/Symfonyが厚い。

## このフォルダの構成
- [codeigniter4/](./codeigniter4/) … **CodeIgniter 4 実務リファレンス（フラッグシップ）**。始め方〜MVC〜Query Builder〜認証〜テスト〜罠まで、項目=1ファイル。
- （CI3 は旧世代のため作らない。CI4 のみ）

> 関連: 同じPHPの本格版は [../laravel/](../laravel/)。
