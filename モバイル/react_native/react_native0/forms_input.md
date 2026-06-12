# 入力（TextInput）（React Native）

## ひとことで言うと
ユーザーが文字を打ち込むためのコアコンポーネント **`<TextInput>`**。Webの `<input>` / `<textarea>` に相当するが、**値の取得が `onChange` ではなく `onChangeText`** という最大の差分がある。制御コンポーネント（`value` ＋ `onChangeText`）で扱うのが基本。

## 役割・なぜ必要か
- ログイン・検索・コメントなど、**ユーザー入力を受け取る唯一の標準部品**。
- Reactの「制御コンポーネント」の考え方はWebと同じ。入力値を**state**で持ち、`value`で表示、変更を`onChangeText`でstateへ反映する一方向ループにする。→ [core_components.md](./core_components.md)
- モバイル特有の事情（ソフトキーボードの種類、キーボードが入力欄を隠す問題）に対応するためのprops（`keyboardType` など）やラッパー（`KeyboardAvoidingView`）が用意されている。

## 基本の書き方（コード）
```tsx
import { useState } from 'react';
import { View, TextInput, Text, StyleSheet } from 'react-native';

export function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <View>
      <TextInput
        style={styles.input}
        value={email}                 // ① state を表示（制御）
        onChangeText={setEmail}       // ② 変更を state へ（onChange ではない！）
        placeholder="メールアドレス"
        keyboardType="email-address"  // メール用キーボード
        autoCapitalize="none"         // 先頭大文字化を無効（メール/IDで必須）
        autoCorrect={false}
      />
      <TextInput
        style={styles.input}
        value={password}
        onChangeText={setPassword}
        placeholder="パスワード"
        secureTextEntry              // ● 表示（マスク）
      />
      <Text>入力中: {email}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  input: {
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 8,
    padding: 12,
    marginBottom: 12,
  },
});
```

## 実務での使い方・定番パターン
- **`keyboardType`** で入力に合ったキーボードを出す: `email-address` / `numeric` / `phone-pad` / `decimal-pad`。数値入力でも返ってくる値は**文字列**なので、保存前に `Number()` 変換。
- **`secureTextEntry`** でパスワードをマスク。目アイコンで表示切替するなら `secureTextEntry={!visible}` とstateで制御。
- **フォームが増えたら `react-hook-form`** を使う。`Controller` で `TextInput` をラップし、バリデーション・エラーメッセージ・送信管理をまとめて任せる。useStateを項目数だけ書く手間が消える。
```tsx
import { useForm, Controller } from 'react-hook-form';
const { control, handleSubmit } = useForm({ defaultValues: { email: '' } });
<Controller
  control={control}
  name="email"
  rules={{ required: '必須です' }}
  render={({ field: { value, onChange } }) => (
    <TextInput value={value} onChangeText={onChange} />
  )}
/>;
```
- **キーボード回避**は **`KeyboardAvoidingView`** で。下部の入力欄がソフトキーボードに隠れるのを防ぐ。iOSは `behavior="padding"`、Androidは挙動が違うので `Platform.select` で出し分けるのが定番。
```tsx
import { KeyboardAvoidingView, Platform } from 'react-native';
<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
  style={{ flex: 1 }}
>
  {/* ...フォーム... */}
</KeyboardAvoidingView>;
```
- **`returnKeyType`** と **`onSubmitEditing`** で「次へ」「完了」キーや次の入力欄へのフォーカス移動を制御すると体験が上がる。

## ハマりどころ / アンチパターン
- **`onChange` を使ってしまう** → 値が取れない／取りづらい。RNの`onChange`はイベントオブジェクトを渡す。**文字列が欲しいなら必ず `onChangeText`**。これが最頻ハマりどころ。
- **キーボードが入力欄を隠す**。`KeyboardAvoidingView` でラップしていない、または `ScrollView` 内で `keyboardShouldPersistTaps="handled"` を付けていないと、入力欄が見えない・タップが効かない。
- **`value` だけ渡して `onChangeText` を忘れる** → 入力しても文字が増えない「打てない入力欄」になる（Webの制御コンポーネントと同じ罠）。
- **`value` を渡さない非制御運用**だと、state とUIがずれてバリデーションやリセットが効かなくなる。原則は制御で。
- **`autoCapitalize` / `autoCorrect` を放置** → メールやユーザーIDで先頭が勝手に大文字化・自動修正され、ログイン失敗の温床。`none`・`false` を明示。
- **数値を文字列のまま計算**して `NaN`。`keyboardType="numeric"` でも値は文字列。
- **キーボードを閉じられない**。空白タップで閉じたいなら `Keyboard.dismiss()` や `ScrollView` の `keyboardDismissMode` を使う。

## 関連: [core_components.md](./core_components.md)
