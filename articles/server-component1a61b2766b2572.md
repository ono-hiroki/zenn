---
title: "React Server Componentをざっくり理解する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [React, Next.js]
published: false
---

## はじめに
Next.jsに入門しようと思ったら、React Server Component (RSC) の話が出てきました。[Reactの公式ドキュメント](https://ja.react.dev/reference/rsc/server-components)を読んだので、その内容をまとめます。

## React Server Component とは

Server Componentsは「クライアント（ブラウザ）に送信される前に、サーバー側で事前にレンダリングされるコンポーネント」です。 従来のReactでは、すべてのコンポーネントがブラウザで動作していましたが「ブラウザ側で動かす必要がないコンポーネントは、サーバー側で動かしてしまおう」という狙いなのかなと思います。

## 従来の課題とServer Componentの解決策

**課題**: Markdownを表示するだけなのに、ブラウザが75KBのライブラリをダウンロードし、さらにAPIリクエストも発生する。

```javascript
// 従来: ブラウザでfetch → ライブラリでパース → 表示
useEffect(() => {
  fetch(`/api/content/${page}`).then(...)
}, [page]);
```
**Server Componentの解決策**: サーバー側でMarkdownをHTMLに変換し、ブラウザにはHTMLだけを送信する。

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
