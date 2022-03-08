---
title: "Vitest テストコードを実装ファイルと同一のファイルに記述する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vitest", "javascript"]
published: true
---

例えば Rust などの言語はテストコードを実装ファイルと同一のファイルに書くことができます。

```rust

#![allow(unused)]
fn main() {
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

https://doc.rust-jp.rs/book-ja/ch11-03-test-organization.html#%E9%9D%9E%E5%85%AC%E9%96%8B%E9%96%A2%E6%95%B0%E3%82%92%E3%83%86%E3%82%B9%E3%83%88%E3%81%99%E3%82%8B

実装と同一のファイルにテストコードを記述するメリットとして以下のような点があります。

- private にしたい目的で export したくない関数をテストできる
- 実装とテストの距離が近いのでテストが書きやすい（私はテストコードを書くときだけいつもエディタの画面を分割して表示してます）
- さっとプロトタイプのコードを書くたいときに素早く書ける

JavaScript には Rust のように言語仕様として同一ファイルにテストを書く機能が備わってはいないのですが、これを実現するための工夫が書かれた記事が見受けられます。

https://zenn.dev/mizchi/articles/inline-testing

https://zenn.dev/ptpadan/articles/jest-same-test-file

そしてどうやら [Vitest](https://zenn.dev/ptpadan/articles/jest-same-test-file#discuss) と呼ばれるテストツールは同一ファイルにテストコードを書く方法を提供しているようですので、これを試してみます。

## Vitest の環境構築

Vitest を利用するには [Vite](https://vitejs.dev/) の v2.7.10 以降と Node の v14 移行が必要です。適当に Vite のプロジェクトを作成します。

```sh
$ npm create vite -- --template vanilla-ts
```

Vitest をインストールします。v0.6.0 以上が必要です。

```sh
$ npm i vitest
```

諸々の設定を記述します。まずは `vite.config.ts` ファイルを作成します。

```ts:vite.config.ts
// vite.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  define: {
    'import.meta.vitest': false,
  },
  test: {
    includeSource: ['src/**/*.{js,ts}'],
  },
})
```

`define` オプションはプロダクションビルド時にテストコードがバンドルファイルに含まれないようにするための設定です。

`test.includeSource` は `src` 配下のファイルがテスト対象であることを伝えます。

続いて `tsconfig.json` に `import.meta.vitest` の型情報を追加します。

```diff son:tsconfig.json
{
  "compilerOptions": {
+    "types": [
+      "vitest/importMeta"
+    ]
  }
}
```

## テストコードを書く

それでは、テストコードを書いてみましょう。

```ts
const add = (a: number, b: number) => {
  return a + b
}

if (import.meta.vitest) {
  const { describe, it, expect } = import.meta.vitest
  describe('calc', () => {
    it('add', () => {
      expect(add(1, 2)).toBe(3)
    })
  })
}
```

テストコードは `if (import.meta.vitest)` のブロック内に記述します。テストコードがランタイムで取り除かれるようにするためですね。

通常 Vitest を使用するときは `describe`、`expect` などは `import` する必要がありますが、その方法ですとやはりランタイム含まれてしまうため `import.meta.vitest` オブジェクトより取得します。

それではテストを実行してみましょう。

```sh
$ npx vitest

 DEV  v0.6.0 ~/vite-project

 √ src/calc.ts (1)

Test Files  1 passed (1)
     Tests  1 passed (1)
      Time  5.30s (in thread 3ms, 179055.32%)


 PASS  Waiting for file changes...
       press h to show help, press q to quit
```

問題なくテストが実施されていますね。

その他、例えば `lodash.cloneDeep` のようにテスト時のみに使用したいライブラリがあることでしょう。そのような場合にトップレベルで `import` 文を記述するとそのライブラリがバンドルファイルに含まれてしまします。

テスト時のみに使用したいライブラリがある場合には `if (import.meta.vitest)` ブロク内で dynamic import を使います。

```ts
const add = (a: number, b: number) => {
  return a + b
}

const obj = {
  a: 1,
  b: {
    c: 2,
  }
}

if (import.meta.vitest) {
  const { describe, it, expect } = import.meta.vitest
  const { cloneDeep } = await import('lodash-es')

  const copy = cloneDeep(obj)

  describe('calc', () => {
    it('add', () => {
      expect(add(copy.a, copy.b.c)).toBe(3)
    })
  })
}
```

## 参考

https://vitest.dev/guide/features.html#in-source-testing

https://twitter.com/antfu7/status/1500377124267429890
