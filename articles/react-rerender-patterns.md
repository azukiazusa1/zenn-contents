---
title: "React の再レンダリングを防ぐ3つのパターン"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react]
published: true
---

React のパフォーマンスについて語るとき、コンポーネントの再レンダリングは外せない話題となるでしょう。React は以下の条件のときに再レンダリングが発生します。

- コンポーネントの state が更新された
- 親のコンポーネントが再レンダリングされた

例えば典型的なカウンターアプリのように、ボタンをクリックしたとき `count` state を更新する場合には必要な再レンダリングといえます。状態が更新されても再レンダリングされなければ、表示されるカウント数は一向に「0」のままですから。

```tsx
import { useState } from "react";

const Counter = () => {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
};
```

パフォーマンス上で問題になるのは不必要な再レンダリングが実行されている場合です。ユーザーが入力フィールドになにか入力をおこなっているとき、入力フィールドのみが再レンダリングされていればユーザーは変更に気がつくことができます。しかし、ユーザーが入力するたびにページ全体が再レンダリングされているような場合には、不要な再レンダリングされていると考えられるでしょう。

通常 React は非常に高速に動作するので、不要な再レンダリングが発生すること自体は大きな問題となりません。しかしながら、再レンダリングが頻繁に発生したり、非常に遅いコンポーネントが再レンダリングされたりすると、入力フィールドへの入力などのインタラクションにおいて遅延を感じるようになります。

## 非常に遅いコンポーネント

実際に非常に遅いコンポーネントが不必要に再レンダリングされてしまう例を試してみましょう。`<SuperSlowComponent>` は `while` ループで約 200ms の同期的な遅延を仕込んでいます。実際に下記のサンドボックスで入力フォームに入力すると遅延があることを確認できます。

```jsx:App.js
import { useState } from "react";

function SuperSlowComponent() {
  const now = performance.now();
  while (performance.now() - now < 200) {}
  return <div>Super slow component</div>;
}

export default function App() {
  const [name, setName] = useState("");
  return (
    <div className="App">
      <label htmlFor="name">Name</label>
      <input
        id="name"
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <SuperSlowComponent />
    </div>
  );
}
```

@[codesandbox](https://codesandbox.io/embed/serene-dewdney-g3rnqg?fontsize=14&hidenavigation=1&theme=dark)
   
   
この問題は入力フォームに入力して `setName` が呼ばれ state が更新されるたびに `<SuperSlowComponent>` が再レンダリングされるため発生しています。まさに不要な再レンダリングによって引き起こされた問題と言えます。この問題を解決する方法をいくつか見ていきます。

## state リフトダウンパターン

はじめに state のリフトダウンパターンと呼ばれる解決策です。これは 1 つのコンポーネントの中で特定の部分のみが state に依存している場合に適しています。state に依存している部分を state と一生に別のコンポーネントに切り出すことで、その他の部分は state が更新されても再レンダリングされないようになります。状態は子コンポーネントが持つようになるためです。

![スクリーンショット 2022-10-02 14.02.54](//images.ctfassets.net/in6v9lxmm5c8/3qxokJ1y1ymckHjHNxsGnh/6a5c05ffe93f3bb44fb5c8905cf88d8e/____________________________2022-10-02_14.02.54.png)

![スクリーンショット 2022-10-02 14.04.46](//images.ctfassets.net/in6v9lxmm5c8/6fE0xfUxxhQoli2IDGSdxY/b5958e77e13f76c4863b32d76b05986b/____________________________2022-10-02_14.04.46.png)

ここでは `useState` と `<input />` の部分を `<Form />` コンポーネントとして切り出すことでその部分のみ state が更新されたとき再レンダリングされるようにできます。実際に以下のデモで試してみてください。`<SuperSlowComponent />` が再レンダリングされないため入力時の遅延を感じなくなるはずです。

```jsx:App.js
import { useState } from "react";

function SuperSlowComponent() {
  const now = performance.now();
  while (performance.now() - now < 200) {}
  return <div>Super slow component</div>;
}

function Form() {
  const [name, setName] = useState("");

  return (
    <>
      <label htmlFor="name">Name</label>
      <input
        id="name"
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
    </>
  );
}

export default function App() {
  return (
    <div className="App">
      <Form />
      <SuperSlowComponent />
    </div>
  );
}
```

@[codesandbox](https://codesandbox.io/embed/modest-gianmarco-ll3b9g?fontsize=14&hidenavigation=1&theme=dark)

## リフトアップパターン

入力フォームに入力した state は他のコンポーネントと共有して使いたいことでしょう。上記の解決策は有効ですが、リフトダウンパターンは state を共有できないのでこのような場合には適していません。

リフトダウンパターンとは反対に state に依存する部分をリフトアップ（＝親コンポーネント）とするパターンでも不要な再レンダリングを防ぐことができます。多少強引な例ですが、入力フォームに「hide」と入力されている場合のみ `<SuperSlowComponent />` を表示しない例を考えてます。単純に考えると公式のドキュメントでも推奨されているとおり、[state のリフトアップ](https://ja.reactjs.org/docs/lifting-state-up.html) を行い state を共有することになるでしょう。

```jsx:App.js
import { useState } from "react";

function SuperSlowComponent() {
  const now = performance.now();
  while (performance.now() - now < 200) {}
  return <div>Super slow component</div>;
}

function Form({ name, setName }) {
  return (
    <>
      <label htmlFor="name">Name</label>
      <input
        id="name"
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
    </>
  );
}

export default function App() {
  const [name, setName] = useState("");
  return (
    <div className="app">
      <Form name={name} setName={setName} />
      {name !== "hide" && <SuperSlowComponent />}
    </div>
  );
}
```

@[codesandbox](https://codesandbox.io/embed/agitated-poincare-qve67s?fontsize=14&hidenavigation=1&theme=dark)
   
しかしこの方法ですとまたもやユーザーが入力フォールドへ入力するたびに、非常に遅いコンポーネントが再レンダリングされてしまいます。いったん `useState` は `<App />` で呼び出さす `<Form />` コンポーネントで管理するように戻しておきましょう。

リフトアップパターンでは、状態に依存している部分をより小さなコンポーネントに切り出しカプセル化する点はリフトダウンパターンと同様です。リフトダウンパターンとの違いは、非常に遅いコンポーネントを `children` として切り出したコンポーネントに渡すところです。`children` は単なる Prop ですので、状態の変化を受けないため、再レンダリングされません。

```jsx:App.js
import { useState } from "react";

function SuperSlowComponent() {
  const now = performance.now();
  while (performance.now() - now < 200) {}
  return <div>Super slow component</div>;
}

function Form({ children }) {
  const [name, setName] = useState("");

  return (
    <>
      <label htmlFor="name">Name</label>
      <input
        id="name"
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      {name !== "hide" && children}
    </>
  );
}

export default function App() {
  return (
    <Form>
      <SuperSlowComponent />
    </Form>
  );
}
```

@[codesandbox](https://codesandbox.io/embed/loving-kalam-p55qq8?fontsize=14&hidenavigation=1&theme=dark)

リフトアップパターンの亜種として、 `children` としてコンポーネントを渡す代わりにコンポーネントを Props として渡すパターンもあります。

```jsx:App.js
import { useState } from "react";

function SuperSlowComponent() {
  const now = performance.now();
  while (performance.now() - now < 200) {}
  return <div>Super slow component</div>;
}

function Form({ left, right }) {
  const [name, setName] = useState("");
  return (
    <>
      {left}
      <label htmlFor="name">Name</label>
      <input
        id="name"
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      {right}
    </>
  );
}

export default function App() {
  return (
    <div className="app">
      <Form right={<SuperSlowComponent />} left={<h1>Hello</h1>} />
    </div>
  );
}
```

@[codesandbox](https://codesandbox.io/embed/romantic-moon-2s40lz?fontsize=14&hidenavigation=1&theme=dark)

## メモ化パターン

最後にコンポーネントを `React.memo` でラップしてメモ化するパターンです。パフォーマンス最適化の文脈でメモ化という用語を聞いたことがある人も多いでしょう。

`React.memo` は親コンポーネントの再レンダリング時に渡された Props と前回の Props の値を比較し、同じ値であればコンポーネントの再レンダリングを行わずにメモ化したコンポーネントを再利用します。下記のデモでは `name` state が変更さ `<App />` コンポーネントが再レンダリングされても `<SuperSlowComponent />` コンポーネントに渡した Props が変化しないため `<SuperSlowComponent />` の再レンダリングが実行されません。

```jsx:App.js
import { memo, useState } from "react";

function SuperSlowComponent() {
  const now = performance.now();
  while (performance.now() - now < 200) {}
  return (
    <>
      <div>Super slow component</div>
    </>
  );
}

const MemoSupserSlowComponent = memo(SuperSlowComponent);

export default function App() {
  const [name, setName] = useState("");

  return (
    <div className="app">
      <label htmlFor="name">Name</label>
      <input
        id="name"
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <MemoSupserSlowComponent />
    </div>
  );
}
```

@[codesandbox](https://codesandbox.io/embed/quirky-dew-egm39t?fontsize=14&hidenavigation=1&theme=dark)

## 参考

- [React re-renders guide: everything, all at once](https://www.developerway.com/posts/react-re-renders-guide)
- [Before You memo()](https://overreacted.io/before-you-memo/)
