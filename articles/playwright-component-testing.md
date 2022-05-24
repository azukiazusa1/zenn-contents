---
title: "Playwright でコンポーネントテスト"
emoji: "🎭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["playwright"]
published: true
---

[Playwright](https://playwright.dev/) は [Cypress](https://www.cypress.io/) [Puppeteer](https://pptr.dev/) と並ぶ E2E テストのための Node.js フレームワークです。Chromium, Chrome, Edge, Firefox, Webkit (Safari)と多くのブラウザに対応しているという特徴があります。

そんな Playwright ですが [v1.22.0](https://github.com/microsoft/playwright/releases/tag/v1.22.0) から React,Vue.js,Svelte のコンポーネントに対してテストを実行できるようになりました。つまり、もともと備えていた E2E レベルのテストに加えて、結合レベルのテストまでカバーできるようになったということです。

Playwright のコンポーネントテスティングの特徴と使い方を見ていきましょう。

## Playwright でテストを書くメリット

現状フロントエンドにおいて、コンポーネントテスティングを実施する際に使用されるフレームワークとして [Testing Library](https://testing-library.com/) が大きな地位を占めています。Testing Library は実際に対象のコンポーネントをユーザーが操作しているようなテストを書くことができるのが特徴で、例えばフォームの要素を取得するためにラベルを使うなど、アクセシブルな要素の取得方法を推奨しています。

Tesing Library と比較してあげられる Playwright のメリットとして、**実際のブラウザ上でコンポーネントテストを実行できる**という点があります。通常 Testing Library は Node.js 上で実行されており [jsdom](https://github.com/jsdom/jsdom) により DOM を再現しています。

Playwright は実際のブラウザを外から操作するため、`jsdom` では発見できないブラウザ特有の問題を自動テストで発見できるメリットがあります。また `viewport` の指定や `prefers-colors-scheme` のエミュレートも可能なので、例えばモバイルでのみ要素が表示されることをテストすることもできます。

## テストを書いてみる

### Playwright のインストール

Playwright は React,Vue.js,Svelte で作成したプロジェクトに簡単に追加できます。試しに以下のコマンドで作成した React プロジェクトを使用します。

```sh
$ npx create-react-app my-app --template typescript
```

Playwright をインストールするために以下のコマンドを実行します。

```sh
$ npm init playwright@latest -- --ct
```

コマンドを実行すると以下のファイルが生成されます。

```sh
├── playwright
│   ├── index.html
│   └── index.ts
├── playwright-ct.config.ts
```

#### `playwright/index.html`

この `html` ファイルはテスト実行時にコンポーネントを描画するために使用されます。このファイルには必ず `id="root"` 属性を持つ要素を含める必要があります。このファイルはビルド時に使用される `index.html` ファイルと内容を合わせておくと良いでしょう。

#### `playwright/index.ts`

このファイルは `playwright/index.html` から呼び出され、`src/main.ts` ファイルのようにスタイルシートを呼び出したりスクリプトを実行するために使用されます。

#### `playwright-ct.config.ts`

[playwright の設定](https://playwright.dev/docs/test-configuration) を記述します。

### 簡単なテストを書いてみる

それでは playwright のインストールも済みましたので、簡単なコンポーネントのテストを実行してみましょう。次のようなカウンターアプリをテストの対象とします。ボタンを押すと表示されるカウントが 1 つづつ増えていく仕様です。さらに「`You clicked {count} times`」という文字は PC 画面のみに表示しモバイル画面で表示したときには「`{count}`」だけ表示することとします。

```tsx:Counter.tsx
import React, { useState } from "react";
import "./Counter.css";

const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p className="hidden md-block">You clicked {count} times</p>
      <p className="md-hidden">{count}</p>
      <button>Click me</button> 
    </div>
  );
};

export default Counter;
```

:::details Counter.css

```css:Counter.css
.hidden {
  display: none;
}

@media (min-width: 768px) {
  .md-hidden {
    display: none;
  }
  .md-block {
    display: block;
  }
}
```

:::

テストファイルとして `Counter.spec.tsx` ファイルを作成します。

```tsx:Counter.spec.tsx
import { test, expect } from "@playwright/experimental-ct-react"; // ①
import Counter from "./Counter";

test("increments the counter", async ({ mount }) => {
  const component = await mount(<Counter />); // ②

  const button = component.locator('role=button[name="Click me"]'); // ③
  await button.click();

  const paragraph = component.locator("role=paragraph"); // ④
  expect(await paragraph.textContent()).toBe("You clicked 1 times");
});
```

① `@playwright/experimental-ct-react` モジュールから `test` と `expect` をインポートします。`test`、`expect` はそれぞれ一般的なテストライブラリと同様の使い方です。

② `test` メソッドのコールバック関数の引数では現在の `playwright` の設定を受け取ります。この引数には `mount` メソッドが含まれているので、これを使用してテスト対象のコンポーネントをマウントします。`mount` メソッドは [Locator](https://playwright.dev/docs/locators) オブジェクトを返します。

③ `component` の `locator` メソッドでボタン要素を取得します。`locator` メソッドはテキスト・CSS セレクタ・Accessible Name などを用いて、マウントされたコンポーネント内に存在する要素を取得できます。要素の取得後は `click` メソッドでボタンクリックを再現します。

④ ボタンクリック後テキストが変化しているはずですので、`paragraph` ロールを取得しその中のテキストが「`You clicked 1 times`」に変化していることを検査します。

それではテストを実行してみましょう。以下コマンドを実行します。

```sh
$ npm run test-ct
```

デフォルトの設定では `chromium`、`firefox`、`webkit` の 3 つのブラウザでそれぞれテストが実行されます。結果を見てみるとテストが失敗していることがわかります。

```sh
  1) [chromium] › src/Counter.spec.tsx:4:1 › increments the counter ================================

    Error: expect(received).toBe(expected) // Object.is equality

    Expected: "You clicked 1 times"
    Received: "You clicked 0 times"

      10 |   const paragraph = component.locator("role=paragraph");
      11 |
    > 12 |   expect(await paragraph.textContent()).toBe("You clicked 1 times");
         |                                         ^
      13 | });
      14 |
```

テストが 1 つでも失敗すると、レポートがブラウザで表示されます。

![スクリーンショット 2022-05-22 23.01.11](//images.ctfassets.net/in6v9lxmm5c8/4cS8CXGWktxul0R7bY2SSc/c56358603c9475f7ca7262d59aff1520/____________________________2022-05-22_23.01.11.png)

項目をクリックするとテストの詳細が表示されます。

![スクリーンショット 2022-05-22 23.04.34](//images.ctfassets.net/in6v9lxmm5c8/61ikLOeMroVlrHG1IR5N27/bd32dcee9f2e749249efe1820bea7a6e/____________________________2022-05-22_23.04.34.png)

テストの詳細を見るに、ボタンクリック時の処理を実装し忘れているようです。以下のようにコンポーネントを修正しましょう。

```diff tsx:Counter.tsx
- <button>Click me</button>
+ <button onClick={() => setCount((prev) => prev + 1)}>Click me</button>
```

再度テストを実行すると、すべてのテストがパスするはずです。

### Viewport に関するテストを書く

[Playwright](https://playwright.dev/docs/test-configuration) の設定をテストごとに上書きすることで、テストごとにブラウザの状態を変更してテストを実行できます。

設定を上書きするにはテストファイルのルートレベル、または `test.describe` ブロック内で `test.use()` メソッドを呼び出します。

下記例はテスト時の viewport を変更して実行しています。

```tsx:Counter.spec.tsx
test.describe("mobie devices", () => {
  test.use({
    viewport: {
      width: 320,
      height: 480,
    },
  });
  test("increments the counter", async ({ mount }) => {
    const component = await mount(<Counter />);

    const button = component.locator('role=button[name="Click me"]');
    await button.click();

    const paragraph = component.locator("role=paragraph");

    expect(await paragraph.textContent()).toBe("1");
  });
});
```

## 感想

やはり、実際のブラウザでテストを実行できるのは嬉しいですね。まだ Experimental な機能であるので、今後に期待が高まるところです。例であげた `viewport` のテストのように、今まで実行できなかったテストが実行できるのも魅力的です。

そういえば [Cypress もコンポーネントテスティングが可能なのですが](https://docs.cypress.io/guides/component-testing/introduction) Cypress は実際に描画される画面を確認しながらテストを実行できるので、同様の機能が欲しいですね。`headless: false` に設定すると一応描画は表示されるのですが、一瞬で閉じてしまうのでほとんど視認できません。
