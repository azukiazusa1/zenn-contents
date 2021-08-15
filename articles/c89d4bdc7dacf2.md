---
title: "JSDocで型を定義してTypeScriptの恩恵を受ける"
emoji: "👯‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['javascript', 'typescript', 'jsdoc']
published: false
---

普段からTypeScriptで開発されている方ならば、TypeScriptの恩恵を十分に堪能されていることと思います。

特に私はVS Codeのインテリセンスには頼りっぱなしでこれがなければまともにコーディングができない体になってしまいました。

しかし時には、TypeScriptという概念が存在しない退屈な世界で開発を行わなけらばならない状況はあるでしょう。私はもはやキーボードでタイピングすることすらままなりません。

そんなJavaScriptしか利用できない状況で代替手段となるのが、JSDocです。JSDocのアノテーションによって型を付与することによって最低限の支援を受けることができます。

# @Type

`@Type`タグを使用すると、TypeScriptで型をつけるときと同じように型を宣言することができます。

```js
/** @type {number}  年齢*/
let age;
```

マウスオーバーをするとJSDocに記載した情報を表示してくれますし、もちろんインテリセンスもしっかりと効いてくれます。


![](https://storage.googleapis.com/zenn-user-upload/af2b40c24e7bd9b045bc315f.png)

![](https://storage.googleapis.com/zenn-user-upload/6e0530c7b6d86ec10d9b430b.png)

## 基本的な型

TypeScriptに用意されている基本的な型はほとんどサポートされています。

```js
/** @type {string} */
let string

/** @type {number} */
let number

/** @type {boolean} */
let boolean

/** @type {string[]} */
let array

/** @type {[number, number]} */
let tuple

/** @type {null} */
let __null__

/** @type {undefined} */
let __undefined__

/** @type {any} */
let any

/** @type {(num1: number, num2: number) => number)} */
const fn = (num1, num2) => {
  return num1 + num2
}

/** @type {{ name: string, age: number}} */
let obj
```

## Union型

Union型も定義することができます。その場合、型ガードによって絞り込みをすることもできます。

```js

/** @type {string|number} */
let strOrNum

if (typeof strOrNum === 'string') {
  strOrNum // string
} else if (typeof strOrNum === 'number') {
  strOrNum // number
} 
```

```js
/** @type {'red'|'green'} */
let color
```

## Enum

@enumでEnum(列挙型)を表現することができますが、TypeScriptの提供するEnumとは異なります。
まず、JSDocのEnumはとてもシンプルです。

```js
/** @enum {number} */
const color = {
  red: 0,
  green: 1,
  blue: 2
}

color.blue
```

![](https://storage.googleapis.com/zenn-user-upload/62c2855fb4e39f534aaa3b3a.png)
![](https://storage.googleapis.com/zenn-user-upload/6a34967a81ac5559e029ba66.png)

また、任意の型を持つこともできます。

```js
/** @enum {(n1: number, n2: number) => number)} */
const MathFuncs = {
  add: (n1, n2) => n1 + n2,
  sub: (n1, n2) => n1 - n2,
};
 
MathFuncs.add
```

![](https://storage.googleapis.com/zenn-user-upload/4cf3ee4532fd1af0025d9695.png)

## インデックスシグネチャ

インデックスシグネチャ(`{ [x: string]: number }`)ど同等の型も定義できます。

```js
/**
 *
 * @type {Object.<string, number>}
 */
var stringToNumber;

stringToNumber['a'] = 'h'
```

## this

`@this`によって、コンテキスト型`this`の型を推測できない場合にも明示的に型を指定することができます。

```js
const button = document.getElementById('button')

button.addEventListener('click', onClick)

/**
 * @this {HTMLElement}
 */
function onClick() {
  this.innerHTML = 'clicked!'
}
```

![](https://storage.googleapis.com/zenn-user-upload/fa6270ecd0dc130fdb77609c.png)

## キャスト

丸括弧で囲まれた式の前に@typeタグを追加することで、型を他の型にキャストすることができます。

```js
/**
 * @type {number | string}
 */
 let numberOrString = Math.random() < 0.5 ? "hello" : 100;
 let typeAssertedNumber = /** @type {number} */ (numberOrString);
```

## インポート型

他ファイルの型宣言から、インポートして使うことができます。これはTypeScript固有のもので、JSDocの標準ではありません。

```ts
// User.ts
export interface User {
  name: string
  age: number
}
```

```js
// index.js
/** @type { import("./User").User;} */
let user
```

もちろんライブラリの型定義をインポートすることもできます。

```js
/** @type {import('axios').AxiosInstance} */
let instance
```

![](https://storage.googleapis.com/zenn-user-upload/c69d165330863b52f8b6da3b.png)


# @paramと@returns

`@type`と同じ構文で、関数の引数(`@param`)と返り値(`@returns`)に型をつけることができます。

```js
/**
 * 
 * @param {number} a パラメータ1
 * @param {number} b パラメータ2
 * @param {number} [c] 任意のパラメータ
 * @param {number} [d=1] デフォルト値をもつパラメータ 
 * @returns {number} 返り値
 */
const calc = (a, b, c, d = 1) => {
  return a + b + c + d
}
```

![スクリーンショット 2021-08-11 8.17.05.png](/open.file/download?fileNo=3255)


# 型を定義する

## @typedef

Object型は複雑になりやすく、再利用するときに同じような記述をするのは冗長です。

TypeScriptで`interface`や`type`を定義するのと同じように、`@typedef`タグで型を定義することができます。

```js
/**
 * @typedef User
 * @property {string} name ユーザー名
 * @property {number} age ユーザーの年齢
 * 
 */

/**
 * @type {User}
 */
let user1

/**
 * @type {User}
 */
let user2
```

## @param

`@param`を使うと、一度限りの方を指定することができます。プロパティはネストした表現をします。

```js
/**
 * 
 * @param {Object} user 
 * @param {string} user.name 
 * @param {number} user.age 
 */
const fn = ({ name, age }) => {
  console.log({ name, age })
}
```

## @callback

`@callback`は、オブジェクトではなく関数の型を定義します。

```js
/**
 * @callback Calc
 * @param {number} num1
 * @param {number} num2
 * @returns {number}
 */

/** @type {Calc} */
const add = (num1, num2) => {
  return num1 + num2
}

/** @type {Calc} */
const sub = (num1, num2) => {
  return num1 - num2
} 
```

# ジェネリクス

`@template`を使うと、ジェネリクスを表現することができます。

```js
/**
 * @template T
 * @param {T} x
 * @return {T}
 */
const test = (x) => {
  return x;
}
 
const a = test("string");
const b = test(123);
const c = test({});
```

しかしながら、ジェネリクスを使えるのは関数のみで、クラスや型宣言にはサポートされていません。

# @ts-check

JSDocによって型を手に入れ快適な生活を送ることができましたが、一つだけ重大な問題点が存在します。

JSDocによって得られたものは所詮型ヒントによる推測にすぎず、実際はただのJavaScriptファイルであることを忘れてはいけません。

以下のように、JSDocで宣言した型以外を代入してもなにも警告をしてくれません。

```js
/** @type {number} */
let num;

num = 1
num = '10'
num = [1, 2, 3]

num.toFixed() // TypeError: num.toFixed is not a function
```

JavaScriptファイルにおいて警告を有効にしたい場合には、最初の行に`@ts-check`を使用します。

```js
// @ts-check
/** @type {number} */
let num;

num = 1
num = '10' // Type 'string' is not assignable to type 'number'.
num = [1, 2, 3] // type 'number[]' is not assignable to type 'number'
```

もし多くのファイルに対して`@ts-check`を適用させたい場合には、代わりに`tsconfig.json`を配置するようにしましょう。

## クラスのアクセス修飾子

JSDocによって、`public`・`private`・`protected`・`readonly`のようなアクセス修飾子を付与することができます。

```js
// @ts-check
class User {
  /**
   * 
   * @param {string} name 
   * @param {number} age 
   */
  constructor(name, age) {
    /** @readonly */
    this.id = 1
  
    /** @public */
    this.name = name
  
    /** @private */
    this.age = age
  }
}

const user = new User('user1', 24)

user.name // ok
user.age // Property 'age' is private and only accessible within class 'User'.
user.id = 2 // Cannot assign to 'id' because it is a read-only property.
```

# 参考

- [TypeScriptを活用したJSプロジェクト](https://www.typescriptlang.org/ja/docs/handbook/intro-to-js-ts.html)
- [JSDocリファレンス](https://www.typescriptlang.org/ja/docs/handbook/jsdoc-supported-types.html)