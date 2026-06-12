# Server Actions（変更処理）（Next.js 15）

## ひとことで言うと
`"use server"` を付けた**サーバ上だけで動く非同期関数**で、DB更新・作成・削除といった**変更処理（mutation）**をサーバ側で実行するための仕組み。フォームの `action` に直接渡せるので、API エンドポイントを別途用意しなくても「送信→サーバで処理→再検証」までを書ける。

## 役割・なぜ必要か
- 従来は「クライアントで `fetch('/api/...')` → Route Handler で受ける」と**2か所**書く必要があった。Server Actions はその往復を**1つのサーバ関数**にまとめる。
- フォーム送信の処理を**サーバに閉じ込められる**：DB接続情報やトークンがクライアントへ漏れない。
- JavaScript 無効でも `<form action={...}>` が動く（プログレッシブエンハンスメント）。
- 処理後に `revalidatePath` / `revalidateTag` を呼べば、**キャッシュ済みの画面を最新化**できる。→ [data_fetching.md](./data_fetching.md)

## 基本の書き方（コード）
```tsx
// app/todos/actions.ts
"use server"; // ← ファイル先頭。これを付けた関数だけがサーバアクションになる

import { z } from "zod";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { db } from "@/lib/db";
import { auth } from "@/lib/auth";

// 入力はスキーマで必ず検証する（フォームの値は信用しない）
const CreateTodo = z.object({
  title: z.string().min(1, "タイトルは必須").max(100),
});

export async function createTodo(prevState: unknown, formData: FormData) {
  // 1) 認可：誰でも叩ける前提で、ログイン確認を必ず入れる
  const user = await auth();
  if (!user) return { error: "ログインが必要です" };

  // 2) 入力検証
  const parsed = CreateTodo.safeParse({ title: formData.get("title") });
  if (!parsed.success) {
    return { error: parsed.error.issues[0].message };
  }

  // 3) 変更処理（DB更新）
  await db.todo.create({ data: { title: parsed.data.title, userId: user.id } });

  // 4) 再検証：一覧画面のキャッシュを破棄して最新化
  revalidatePath("/todos");
  redirect("/todos"); // 必要なら遷移
}
```

```tsx
// app/todos/TodoForm.tsx
"use client";
import { useActionState } from "react"; // 旧 useFormState（React 19 で改称）
import { useFormStatus } from "react-dom";
import { createTodo } from "./actions";

function SubmitButton() {
  const { pending } = useFormStatus(); // 送信中かどうか（<form> の子で使う）
  return <button disabled={pending}>{pending ? "送信中…" : "追加"}</button>;
}

export function TodoForm() {
  const [state, formAction] = useActionState(createTodo, null);
  return (
    <form action={formAction}>
      <input name="title" />
      {state?.error && <p role="alert">{state.error}</p>}
      <SubmitButton />
    </form>
  );
}
```

## 実務での使い方・定番パターン
- **`action={serverAction}`**：`<form>` に直接渡すのが基本形。`FormData` が第1引数（`useActionState` 併用時は第2引数）で来る。
- **`useActionState(action, initial)`**：直前の戻り値（バリデーションエラー等）を `state` として受け取り、画面に表示する。サーバから `{ error }` や `{ fieldErrors }` を返す設計が定番。
- **`useFormStatus()`**：送信中フラグ。`<form>` の**子コンポーネント**でしか効かないので、ボタンを別コンポーネントに切り出す。
- **再検証の使い分け**：パス単位なら `revalidatePath("/todos")`、タグ単位なら `fetch(..., { next: { tags: ["todos"] } })` と組で `revalidateTag("todos")`。
- **クライアントから直接呼ぶ**：`onClick={() => deleteTodo(id)}` のようにフォーム以外からも呼べる（`.bind` で引数を固定するのも定番）。
- ボタン単位で `formAction={anotherAction}` を分けると、1つのフォームに複数アクションを持たせられる。

## ハマりどころ / アンチパターン
- **`"use server"` の付け忘れ**：付いていない関数は通常のサーバ関数で、`action` に渡しても動かない／クライアントへバンドルされてしまう。ファイル先頭か関数先頭に必ず書く。
- **入力検証を省く**：`formData` はユーザが自由に改ざんできる。zod 等で**必ず検証**してから DB に渡す。型注釈は実行時チェックにならない点に注意。
- **認可の抜け**：Server Action は実質「公開エンドポイント」。誰でも呼べる前提で、関数の冒頭で**ログイン・権限チェック**を入れる。これを忘れると IDOR（他人のデータ操作）になる。
- **再検証忘れ**：DB は更新したのに `revalidatePath/Tag` を呼ばず、画面が古いまま。「更新したのに反映されない」の典型原因。
- **戻り値にシリアライズ不能な値**：Server Action の戻り値はクライアントへ渡るので、関数やクラスインスタンスは返さない（プレーンなオブジェクトに整形する）。
- **巨大ファイルへの集約**：アクションは機能ごとに `actions.ts` を分け、太らせない。複雑な手続きはサービス層へ。

## 関連
[data_fetching.md](./data_fetching.md) / [route_handlers.md](./route_handlers.md)
