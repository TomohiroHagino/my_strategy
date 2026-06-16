# 保存 / 状態管理（AsyncStorage / Redux / Zustand）（React Native）

## ひとことで言うと
アプリを閉じても消えないデータの **永続化**（端末保存）と、画面をまたいで使う **状態管理** の話。RN の定番は永続化に **`AsyncStorage`**、状態管理は Web React と同じ（Context / Redux / Zustand）。機密情報だけは **`expo-secure-store`** を使う。

## 役割・なぜ必要か
- **状態（state）** はメモリ上のデータでアプリを閉じると消える。ログイン情報・設定・下書きなどは閉じても残したい → **永続化（storage）** が要る。
- **`AsyncStorage`**：キー・バリューの **永続ストア**（端末のローカルに保存）。Web の `localStorage` に近いが **非同期** かつ **文字列のみ**。トークンの有無・テーマ設定・オンボーディング既読フラグなど軽量データ向け。
- **状態管理ライブラリ**：複数画面で共有する状態を整理するため。RN でも React と同じ選択肢（小規模なら Context、規模が出れば Redux Toolkit / Zustand）。
- **`expo-secure-store`**：OS の安全領域（iOS Keychain / Android Keystore）に暗号化して保存。**認証トークン・パスワード等の機密はここへ**。

## 基本の書き方（コード）
```bash
npx expo install @react-native-async-storage/async-storage expo-secure-store
```
```tsx
// AsyncStorage：非同期・文字列のみ。オブジェクトは JSON 化して出し入れする
import AsyncStorage from '@react-native-async-storage/async-storage';

type Settings = { theme: 'light' | 'dark'; notify: boolean };

export async function saveSettings(s: Settings): Promise<void> {
  await AsyncStorage.setItem('settings', JSON.stringify(s)); // オブジェクト→文字列
}
export async function loadSettings(): Promise<Settings | null> {
  const raw = await AsyncStorage.getItem('settings');        // null の可能性あり
  return raw ? (JSON.parse(raw) as Settings) : null;         // 文字列→オブジェクト
}
```
```tsx
// 機密（トークン等）は SecureStore へ。AsyncStorage には入れない
import * as SecureStore from 'expo-secure-store';
await SecureStore.setItemAsync('authToken', token);
const token = await SecureStore.getItemAsync('authToken');
```
```tsx
// グローバル状態：Zustand（軽量で RN と相性が良い）
import { create } from 'zustand';

type AuthState = { token: string | null; setToken: (t: string | null) => void };
export const useAuth = create<AuthState>((set) => ({
  token: null,
  setToken: (token) => set({ token }), // 新しい状態を返す（直接書き換えない）
}));
```

## 実務での使い方・定番パターン
- **役割で保存先を分ける**：軽量な非機密 → `AsyncStorage`、機密 → `SecureStore`。大量データや高速読み書きが要るなら `react-native-mmkv` も定番。
- **永続化はラッパー関数に集約**：`saveSettings` / `loadSettings` のように薄い関数でくるみ、`JSON.stringify/parse` の対称性とエラー処理を1か所に。
- **状態管理は規模で選ぶ**：少数の共有値なら Context、横断的で更新も多いなら **Zustand**（ボイラープレート少）か **Redux Toolkit**（規約・DevTools 充実）。
- **サーバ状態は別扱い**：API から取るデータは **TanStack Query** などのキャッシュ層に任せ、グローバルストアに丸写ししない。クライアント状態（UI・認証フラグ）とサーバ状態を混ぜない。
- **永続化と状態の橋渡し**：起動時に `AsyncStorage`/`SecureStore` から読み出してストアへ流し込む（hydrate）。Zustand なら `persist` ミドルウェア＋AsyncStorage アダプタで自動化できる。
- **基礎は React と同じ**：`useState`/`useEffect`/Context の使い方は Web と共通。差分はストレージ API が非同期な点。→ [components_state.md](./components_state.md)

## ハマりどころ / アンチパターン
- **`AsyncStorage` は非同期**：`localStorage` 感覚で同期に書くと値が取れない。必ず `await`/`then`。読み込み中の **ローディング状態** を用意する。
- **文字列しか入らない**：オブジェクト/配列/数値をそのまま渡すと壊れる。**`JSON.stringify` で入れ、`JSON.parse` で出す**。`getItem` は **`null` を返し得る** ので分岐必須。
- **機密を `AsyncStorage` に入れる**：暗号化されず平文に近い形で残り、端末解析で漏れる。**トークン・パスワードは必ず `SecureStore`**。
- **グローバル状態の入れすぎ**：何でもグローバルに置くと依存が増え再レンダリングも増える。ローカルで足りる state はローカルに。共有が本当に必要な値だけストアへ。
- **直接ミューテート**：状態オブジェクトを書き換えると再描画が走らない／バグる。常に **新しいオブジェクトを返す**（イミュータブル更新）。
- **hydrate 前の参照**：起動直後、永続値の読み込み完了前に画面が参照して一瞬「未ログイン」になる。読み込み完了フラグでガードする。
- **キーの衝突・無管理**：文字列キーを散らすと衝突・消し忘れが起きる。キーは定数で一元管理する。

## 関連
[components_state.md](./components_state.md) / [networking.md](./networking.md) / [native_modules.md](./native_modules.md)
