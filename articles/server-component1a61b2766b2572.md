---
title: "React Server Component 入門"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [React, Next.js]
published: false
---

## はじめに
Next.jsに入門しようと思ったら、React Server Component (RSC) の話が出てきたので、[Reactの公式ドキュメント](https://ja.react.dev/reference/rsc/server-components)を読んだのでその内容をまとめます。

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
**Server Componentの解決作**: サーバー側でMarkdownをHTMLに変換し、ブラウザにはHTMLだけを送信する。

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
  return <Expandable><p note={note} /></Expandable>;
}

// Client Component（子）- これだけブラウザで動く
"use client"
function Expandable({children}) {
  const [expanded, setExpanded] = useState(false);
  // ボタンクリックで開閉
}
```

## 簡単な例
以下は、Next.jsのApp Routerでサーバーコンポーネントを使う簡単な例です。

```tsx
// app/server-action/page.tsx
import { greet } from './actions'

export default function Page() {
    return (
        <main>
            <h1>Server Action デモ</h1>
            <form action={greet}>
                <input type="text" name="name" placeholder="名前を入力" />
                <button type="submit">送信</button>
            </form>
        </main>
    )
}
```

```tsx
// app/server-action/actions.ts
'use server'

export async function greet(formData: FormData) {
    const name = formData.get('name')
    console.log(`[Server] name = ${name}`)
}
```

この例では、`page.tsx`がサーバーコンポーネントとして動作し、フォームの送信時に`actions.ts`の`greet`関数がサーバー側で実行されます。  
console.logの出力はブラウザのコンソールではなく、`npm run dev`のターミナルに表示されることからサーバー側で動作していることが確認できます。


