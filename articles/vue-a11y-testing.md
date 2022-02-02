---
title: "Vue.js でウェブアクセシビリティを自動でテストする仕組み"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['vue', 'storybook', 'jest', 'アクセシビリティ']
published: true
---

近年の Web アクセシビリティはますます重要な要素となっています。フロントエンドの開発を行う中でアクセシビリティに興味を持った人も多いことでしょう。

アクセシビリティを確認するためには実際のデバイスで手動で確認することが最も正確なのですが、それを行うには専門的な知識の多くの時間が必要です。

そこでアクセシビリティに関して自動テストと手動テストを組み合わせて実践するのが効果的です。Deque 社の調査によると、自動テストによってアクセシビリティの問題の 57% がカバーされていることが発見されています。

https://www.deque.com/blog/automated-testing-study-identifies-57-percent-of-digital-accessibility-issues/

この記事では主に自動でアクセシビリティをテストする方法について羅列します。

## Eslint

Eslint のプラグインである [eslint-plugin-vue-a11y](https://github.com/maranran/eslint-plugin-vue-a11y) は `.vue` ファイルにアクセシビリティのルールを追加します。

Eslint がまだインストールされていないならまずは Eslint 本体をインストールします。

```sh
npm i eslint --save-dev
```

その後プラグインをインストールします。

```sh
npm install eslint-plugin-vue-a11y --save-dev
```

インストールが完了したら Eslint の設定ファイル（`eslintrc.cjs`）にプラグインを追加します。

```js
module.exports = {
 extends: [
    "plugin:vue-a11y/recommended",
  ]
}
```

プラグインを追加した後、以下のようにアクセシブルではないコードに対してエラーが検出されます。

![スクリーンショット 2022-01-29 22.14.33](//images.ctfassets.net/in6v9lxmm5c8/4FsW7g3h4ZSx322HiMq2YN/8edf27fcf2baf8624e3ebace619874a1/____________________________2022-01-29_22.14.33.png)

Eslint による検査はコードを書いている最中にすぐにフィードバックを受けることができるというメリットはありますが、適用されるルールは多くはありません。

## Vue Axe

[Vue Axe](https://axe.vue-a11y.com/) は React-axe に影響を受けたライブラリであり Vue.js アプリケーションの開発中にアクセシビリティをチェックすることができます。Vue Axe は[Deque 社](https://www.deque.com/) によって開発された [axe-core](https://github.com/dequelabs/axe-core) を使用しています。

Vue.js 3 を使用している場合には [vue-axe-next](https://github.com/vue-a11y/vue-axe-next) を使用する必要があります。

Vue.js 3 & Vite を使用したアプリケーションに Vue Axe を導入してみましょう。まずはパッケージをインストールします。

```sh
npm install -D axe-core vue-axe@next
```

Vue Axe を開発中のみ有効化するために `main.ts` を編集します。

```ts:src/main.ts
import { createApp, h, Fragment, App as IApp } from "vue";
import App from "./App.vue";

const setup = async () => {
  let app: IApp<Element>;
  if (import.meta.env.DEV) {
    const VueAxe = await import("vue-axe");
    app = createApp({
      render: () => h(Fragment, [h(App), h(VueAxe.VueAxePopup)]),
    });
    app.use(VueAxe.default);
  } else {
    app = createApp(App);
  }

  return app;
};

setup().then((app) => app.mount("#app"));
```

TypeScript を使用しているのなら型定義ファイルを追加しておく必要があるでしょう。

```ts:vue-axe.d.ts
declare module "vue-axe";
```

さらに `vite.config.js` ファイルの `optimizeDeps` に `axe-core` を追加します。

```js:vite.config.js
import { defineConfig } from "vite";

export default defineConfig({
  // ..
  optimizeDeps: {
    include: ["axe-core"],
  },
});

```

これで準備は整いました。いつもの通りに開発モードでアプリケーションを起動してみます。

```sh
npm run dev
```

画面の右下にアクセシビリティのボタンが追加されていることがわかります。表示を見るに、2件のアクセシビリティのエラーが検出されているようです。

![スクリーンショット 2022-01-29 22.56.35](//images.ctfassets.net/in6v9lxmm5c8/2lkmL11PNtrtUFMTaVy6eY/613d95264642fb1b163bca02814513f0/____________________________2022-01-29_22.56.35.png)

ボタンをクリックすると、エラー内容が表示されます。

![スクリーンショット 2022-01-29 23.00.20](//images.ctfassets.net/in6v9lxmm5c8/7xFJNaOHxpebZ287OV0dGH/63634329bffc6381390a83cf40e14514/____________________________2022-01-29_23.00.20.png)

さらに、この状態で試しに `App.vue` の `<img>` タグの `alt` 属性を取り除いてみましょう。

```diff:src/App.vue
 <template>
   <header>
     <img
-      alt="vue logo"
       class="logo"
       src="@/assets/logo.svg"
       width="125"
       height="125"
     />

     <div class="wrapper">
       <h1>Hello</h1>
     </div>
   </header>
 </template>
 ```

 HMR が走ると再度アクセシビリティの検査が実行されます。その後新たに「Images must have alternate text.」というエラーが追加されたことが確認できました。

 ![スクリーンショット 2022-01-29 23.05.59](//images.ctfassets.net/in6v9lxmm5c8/3VB8QcKHS7C3SpsFAUEGqJ/e3199d3582a84950631a20e9e61109ca/____________________________2022-01-29_23.05.59.png)

さらにエラー情報をクリックして詳細を表示させると該当のソースコードが表示されます。「Highlight」をクリックすれば画面上のどこに問題があるのかハイライトで表示してくれます。

![スクリーンショット 2022-01-29 23.07.06](//images.ctfassets.net/in6v9lxmm5c8/vRoFxxMOC7UEbccavE0R5/1e1f019de3d285c0107915a6a4a23abb/____________________________2022-01-29_23.07.06.png)

「Learm more」のリンクをクリックすれば問題の詳細を確認できます。

https://dequeuniversity.com/rules/axe/4.3/image-alt?application=vue-axe

## Storybook

Storybook のアドオンである [storybook-addon-a11y](https://storybook.js.org/addons/@storybook/addon-a11y) を使用すれば Storybook 上でアクセシビリティに問題がないか確認することができます。

まずはアドオンをインストールします。

```sh
npm i --save--dev @storybook/addon-a11y
```

`.storybook/main.js` にインストールしたアドオンを追加します。

```js:.storybook/main.js
module.exports = {
  addons: ['@storybook/addon-a11y'],
};
```

アクセシビリティ上問題があるボタンを試しに作成します。簡単に Props から配色を変更できるようにしています。

```vue:components/BadButton.vue
<script lang="ts" setup>
interface Props {
  color: "primary" | "danger";
}

defineProps<Props>();
</script>

<template>
  <button class="btn" :class="color"><slot /></button>
</template>

<style scoped>
.btn {
  display: inline-block;
  padding: 0.5rem 1rem;
  border-radius: 0.25rem;
  font-size: 1rem;
  font-weight: bold;
  text-align: center;
  cursor: pointer;
  border: 1px solid transparent;
}

.primary {
  background-color: blue;
  color: white;
}

.danger {
  background-color: coral;
  color: red;
}
</style>
```

作成したボタンに対応するストーリーも作成しましょう。Props の color で `praimary` を選択した場合と `danger` を選択した場合のストーリーを作成します。

```ts:components/BadButton.stories.ts
import type { Story, Meta } from "@storybook/vue3";
import BadButton from "./BadButton.vue";

export default {
  title: "BadButton",
  component: BadButton,
  argTypes: {},
} as Meta;

const Template: Story = (args) => ({
  components: { BadButton },
  setup() {
    return { args };
  },
  template: `<BadButton v-bind="args">Button</BadButton>`,
});

export const Primary = Template.bind({});
Primary.args = {
  color: "primary",
};

export const Danger = Template.bind({});
Danger.args = {
  color: "danger",
};
```

Storybook を開いてみましょう・画面下部の Accessibility のタブに「Elements must have sufficient color contrast」という警告が表示されていることがわかります。テキスト要素は、背景に対して十分な色のコントラストを持っている必要があるというルールです。

![スクリーンショット 2022-01-30 10.37.20](//images.ctfassets.net/in6v9lxmm5c8/46OEtkMNo1Thxt4CvHIjet/168baa62633cccc0359f47c27d4f86eb/____________________________2022-01-30_10.37.20.png)

https://dequeuniversity.com/rules/axe/4.3/color-contrast?application=axeAPI

更に画面上部のアクセシビリティのアイコンをクリックすると色覚特性がある場合の見え方をシミュレーションすることができます。Blurred Virsion を選択するとぼやけた状態をシミュレーションできます。

![スクリーンショット 2022-01-30 10.43.06](//images.ctfassets.net/in6v9lxmm5c8/4KERdSVWy1USsftitWPFqo/9c9a6ac798d1270c8c7898479c625dc3/____________________________2022-01-30_10.43.06.png)

## Jest

最後に Jest を使用してアクセシビリティのテストを自動化できるようにしましょう。[jest-axe](https://github.com/nickcolley/jest-axe) は Jest でアクセシビリティをテストするカスタムマッチャーを追加します。

まずはツールをインストールしましょう。

```sh
npm install --save-dev jest-axe
```

TypeScript なら型定義もインストールします。

```sh
npm install --save-dev @types/jest-axe
```

今回もアクセシビリティ上問題があるコンポーネントを作成します。

```vue:components/BadInput.vue
<script lang="ts">
import { defineComponent } from "vue";
export default defineComponent({
  inheritAttrs: false,
});
</script>

<script setup lang="ts">
interface Props {
  label: string;
}

defineProps<Props>();
</script>

<template>
  {{ label }}
  <input v-bind="$attrs" />
</template>
```

作成したコンポーネントに対するテストを作成します。

```ts:components/BadInput.spec.ts
import { axe, toHaveNoViolations } from "jest-axe";
import { render } from "@testing-library/vue";
import BadInput from "./BadInput.vue";

expect.extend(toHaveNoViolations);

describe("components/BadInput", () => {
  it("should have no accessibility violations", async () => {
    const { container } = render(BadInput, {
      props: {
        label: "name",
      },
    });
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

普段のテストと同じように Testing-library を利用してコンポーネントをレンダリングします。レンダリング結果を `axe` 関数の引数として渡す必要があります。`toHaveNoViolations` を呼び出すことによってコンポーネントのアクセシビリティをテストすることができます。

テストを実行してみましょう。

```sh
$ npm run test

> vue-a11y-testing@0.0.0 test
> jest

 FAIL  src/components/BadInput.spec.ts
  components/BadInput
    ✕ should have no accessibility violations (117 ms)

  ● components/BadInput › should have no accessibility violations

    expect(received).toHaveNoViolations(expected)

    Expected the HTML found at $('input') to have no violations:

    <input>

    Received:

    "Form elements must have labels (label)"

    Fix any of the following:
      Form element does not have an implicit (wrapped) <label>
      Form element does not have an explicit <label>
      aria-label attribute does not exist or is empty
      aria-labelledby attribute does not exist, references elements that do not exist or references elements that are empty
      Element has no title attribute
      Element has no placeholder attribute
      Element's default semantics were not overridden with role="none" or role="presentation"

    You can find more information on this issue here: 
    https://dequeuniversity.com/rules/axe/4.2/label?application=axeAPI

      13 |     });
      14 |     const results = await axe(container);
    > 15 |     expect(results).toHaveNoViolations();
         |                     ^
      16 |   });
      17 | });
      18 |

      at Object.<anonymous> (src/components/BadInput.spec.ts:15:21)

Test Suites: 1 failed, 1 total
Tests:       1 failed, 1 total
Snapshots:   0 total
Time:        3.069 s, estimated 7 s
Ran all test suites.
```

`<input>` 要素に `<label>` が存在しないという旨の警告が表示されテストに失敗することが確認できました。

## 終わりに

以上、アクセシビリティを自動でテストする4つの方法について紹介しました。サンプルコードは以下のレポジトリを参照してください。

https://github.com/azukiazusa1/vue-a11y-testing

最後に自動テストのみではアプリケーションがアクセシブルであることを保証できないという点に注意してください。あくまで自動テストの役割は一般的な問題を解決するだけです。

真のアクセシビリティを獲得するためには実際のユーザーのデバイスで手動でテストする、ユーザー調査に障がい者を含めるといったことを必ず実施する必要があります。