---
title: "React Server Componentとサーバー関数をざっくり理解する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [React, Next.js]
published: true
---

## はじめに
Next.jsに入門しようと思ったら、React Server Component（RSC）やサーバー関数（Server Functions）の話が出てきました。[Reactの公式ドキュメント](https://ja.react.dev/reference/rsc/server-components)を読んだので、その内容をまとめます。

## React Server Component とは

サーバーコンポーネントは「クライアント（ブラウザ）に送信される前に、サーバー側で事前にレンダリングされるコンポーネント」です。従来のReactでは、すべてのコンポーネントがブラウザで動作していましたが「ブラウザ側で動かす必要がないコンポーネントは、サーバー側で動かしてしまおう」という狙いなのかなと思います。

## 従来の課題とサーバーコンポーネントの解決策

**課題**: Markdownを表示するだけなのに、ブラウザが75KBのライブラリをダウンロードし、さらにAPIリクエストも発生する。

```javascript
// 従来: ブラウザでfetch → ライブラリでパース → 表示
useEffect(() => {
  fetch(`/api/content/${page}`).then(...)
}, [page]);
```
**サーバーコンポーネントの解決策**: サーバー側でMarkdownをHTMLに変換し、ブラウザにはHTMLだけを送信する。

```javascript
// Server Component: サーバーでパース → HTMLをブラウザに送信
async function Page({page}) {
  const content = await file.readFile(`${page}.md`);
  return <div>{sanitizeHtml(marked(content))}</div>;
}
```
結果として、パースするためのライブラリをブラウザに送信する必要がなくなり、APIリクエストも発生しないため、パフォーマンスが向上します。

## サーバーコンポーネントで使えない機能
サーバーコンポーネントはサーバー側で実行されるため、`useState`などのインタラクティブな機能は使用できません。また、ブラウザのAPI（`window`や`document`など）も利用できません。
インタラクティブな機能が必要な場合は、クライアントコンポーネントとして別途実装しサーバーコンポーネントから呼び出す必要があります。

```javascript
// Server Component（親）
async function Notes() {
  const notes = await db.notes.getAll();
  return (
    <Expandable>
      {notes.map(note => <p key={note.id}>{note.content}</p>)}
    </Expandable>
  );
}
```

```javascript
// Client Component（子）- これだけブラウザで動く
"use client"

function Expandable({children}) {
  const [expanded, setExpanded] = useState(false);
  // ボタンクリックで開閉
}
```

## 簡単な例
以下は、Next.jsのApp Routerでサーバーコンポーネントを使ってデータを取得する例です。

```tsx
// app/users/page.tsx
async function getUsers() {
  const res = await fetch('https://jsonplaceholder.typicode.com/users');
  return res.json();
}

export default async function UsersPage() {
  const users = await getUsers();

  return (
    <main>
      <h1>ユーザー一覧</h1>
      <ul>
        {users.map((user: { id: number; name: string }) => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </main>
  );
}
```

この例では、`UsersPage`がサーバーコンポーネントとして動作します。`async`関数として定義し、コンポーネント内で直接`await`を使ってデータを取得できます。従来のように`useEffect`や`useState`を使う必要がなく、シンプルに記述できます。

データ取得はサーバー側で行われるため、APIキーなどの機密情報をクライアントに露出させずに済むというメリットもあります。

## サーバー関数（Server Functions）とは

サーバー関数は「クライアント（ブラウザ）から、サーバー上の関数を直接呼び出せる仕組み」です。

**従来の方法**: APIエンドポイントを定義して、fetchで呼び出す
```javascript
// クライアント側
await fetch('/api/users', {
  method: 'POST',
  body: JSON.stringify({ name })
});
```

**サーバー関数**: 関数を直接呼び出すだけ
```javascript
// クライアント側
await updateName(name);  // これだけでサーバーで実行される
```

APIルートを自分で定義しなくても、関数呼び出しの形でサーバー処理を実行できます。

## サーバー関数の定義方法

ファイルの先頭に`"use server"`を書くと、そのファイル内のエクスポートされた関数はすべてサーバー関数になります。

```javascript
// actions.js
"use server";

export async function updateName(name) {
  if (!name) {
    return { error: 'Name is required' };
  }
  await db.users.updateName(name);
}
```

クライアントコンポーネントからは通常の関数と同じようにインポートして使えます。

```javascript
// Client Component
"use client";
import { updateName } from './actions';
```

## サーバー関数の例

以下は、Next.jsのApp Routerでサーバー関数を使う簡単な例です。フォームから名前を送信し、サーバー側で挨拶メッセージを生成して返します。

```tsx
// app/server-function/actions.ts
"use server";

export type GreetState = {
    message: string | null;
    error: string | null;
};

export async function greet(prevState: GreetState, formData: FormData): Promise<GreetState> {
    const name = formData.get('name') as string;

    if (!name) {
        return { message: null, error: '名前を入力してください' };
    }

    return { message: `こんにちは、${name}さん！`, error: null };
}
```

```tsx
// app/server-function/page.tsx
"use client";

import { greet } from './actions';
import { useActionState } from 'react';

export default function GreetPage() {
  const [state, submitAction, isPending] = useActionState(greet, { message: null, error: null });

  return (
    <div>
      <form action={submitAction}>
        <input type="text" name="name" placeholder="名前を入力" disabled={isPending} />
        <button type="submit" disabled={isPending}>
          {isPending ? '送信中...' : '送信'}
        </button>
      </form>
      {state?.error && <p style={{ color: 'red' }}>{state.error}</p>}
      {state?.message && <p>{state.message}</p>}
    </div>
  );
}
```

この例では、フォーム送信時に`greet`がサーバー側で実行され、結果がクライアントに返されます。外部APIやデータベースを使わずに、サーバー関数の基本的な動作を確認できます。

## サーバー関数の裏側

先ほどの例で、分かりやすくするために`greet`関数に遅延を入れて、ブラウザの開発者ツールでネットワークタブを確認してみます。

すると、確かにAPIリクエストが発生していることが分かります。
![ネットワークタブでAPIリクエストを確認](/images/server-component1a61b2766b2572/img.png)

実際にcurlでリクエストを送ってみると、以下のようなレスポンスが返ってきます。

```bash
$ curl -X POST "http://localhost:3000/server-function" \
  -H "Next-Action: 606ff4047cb4a710a8ff4213835980c0dd41a6e981" \
  -H "Content-Type: multipart/form-data" \
  -F "name=aaaa"
:N1766222359175.8567
0:{"a":"$@1","f":"","b":"development"}
1:D{"time":0.59825000166893}
1:E{"digest":"2755580262","name":"Error","message":"Connection closed.","stack":[],"env":"Server","owner":null}
```

サーバー関数は、書き心地としては関数の定義と呼び出しだけで完結しますが、裏側ではHTTPリクエストが発生しています。つまり、APIエンドポイントが自動的に公開されるということです。セキュリティの観点から、サーバー関数内でも認証・認可のチェックは必要になります。
