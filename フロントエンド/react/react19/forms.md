# フォーム（制御コンポーネント）（React）

## ひとことで言うと
入力欄(`<input>` など)の値を **state で持ち、`value` と `onChange` で同期させる**書き方が**制御コンポーネント**。画面の「真実の値」を React 側(state)に一本化する。

## 役割・なぜ必要か
- 素のHTMLでは入力値はDOM側が持つ。Reactは「**UI = f(state)**」が基本なので、入力値も state に寄せると **バリデーション・整形・送信・初期化**を全部 state 操作で扱える。
- `value={state}` で「stateが画面の正」、`onChange` で「ユーザー入力をstateへ反映」。この往復で**常に一致**する。
- 送信は `<form onSubmit>` で受け、`e.preventDefault()` で**ブラウザのページ再読み込みを止める**(SPAでは必須)。

## 基本の書き方（コード）
```tsx
import { useState, FormEvent } from "react";

export function SignupForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();            // ページ再読み込みを止める
    console.log({ name, email });  // ここでAPI送信など
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={name}                          // stateが画面の正
        onChange={(e) => setName(e.target.value)} // 入力をstateへ
        placeholder="名前"
      />
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="メール"
      />
      <button type="submit">登録</button>
    </form>
  );
}
```

## 実務での使い方・定番パターン
- **非制御コンポーネント**：state を介さず `useRef` でDOMから直接値を読む。軽い・再レンダリングしない反面、整形やリアルタイム検証はやりにくい。
  ```tsx
  import { useRef, FormEvent } from "react";

  export function QuickForm() {
    const nameRef = useRef<HTMLInputElement>(null);
    const onSubmit = (e: FormEvent) => {
      e.preventDefault();
      console.log(nameRef.current?.value); // 送信時にまとめて読む
    };
    return (
      <form onSubmit={onSubmit}>
        <input ref={nameRef} defaultValue="" /> {/* value でなく defaultValue */}
        <button>送信</button>
      </form>
    );
  }
  ```
- **実務の定番は React Hook Form + zod**。フォームは非制御ベースで高速、zod でスキーマ検証し型も導出できる。手書きの `useState` 地獄を避けられる。
  ```tsx
  import { useForm } from "react-hook-form";
  import { zodResolver } from "@hookform/resolvers/zod";
  import { z } from "zod";

  const schema = z.object({
    email: z.string().email("メール形式が不正です"),
    age: z.coerce.number().min(0),
  });
  type FormValues = z.infer<typeof schema>; // 型をスキーマから導出

  export function ProfileForm() {
    const { register, handleSubmit, formState: { errors } } =
      useForm<FormValues>({ resolver: zodResolver(schema) });

    const onSubmit = (data: FormValues) => console.log(data);

    return (
      <form onSubmit={handleSubmit(onSubmit)}>
        <input {...register("email")} />
        {errors.email && <p>{errors.email.message}</p>}
        <input {...register("age")} />
        <button>保存</button>
      </form>
    );
  }
  ```
- 入力数が多い／検証が複雑なら最初から RHF+zod を選ぶと後がラク。素の `useState` は1〜2個の単純フォーム向け。

## ハマりどころ / アンチパターン
- **`value` だけ書いて `onChange` を書かない** → 値が state に固定され**入力できない**(read-only 扱い)。Reactが警告を出す。読み取り専用なら `readOnly` を明示。
- **制御と非制御の混在**：「`value` が `undefined`/`null` → 値あり」に途中で変わると *"changing an uncontrolled input to be controlled"* 警告。初期値は `""` など**必ず定義**する。
- 制御なら `value`＋`onChange`、非制御なら `defaultValue`＋`ref`。**どちらか一方に統一**する。
- `onSubmit` で `e.preventDefault()` を忘れると**画面がリロード**され state が吹き飛ぶ。
- チェックボックスは `value` でなく `checked` を state に同期する。

## 関連
[state.md](./state.md) / [refs.md](./refs.md)
