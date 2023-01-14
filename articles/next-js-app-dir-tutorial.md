---
title: "Next.js 13 app directory で記事投稿サイトを作ってみよう"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs]
published: true
---

:::message alert
Next.js 13 の `app` directory は 2023 年 1 月 13 日現在ベータ版の機能です。本記事の内容は変更される可能性があります。
:::

以下のコマンドでパッケージをインストールして、開発環境を起動してみましょう。

```bash
npm run install
npm run dev
```

http://localhost:3000/ にアクセスすると、以下のような画面が表示されます。

![Next.js で開発環境を起動したデフォルトで表示されるブラウザの画面](https://storage.googleapis.com/zenn-user-upload/74542e682840-20230113.png)

まずは `app` ディレクトリに初期状態で存在しているファイルを見ていきましょう。

- `page.tsx`：`pages` ディレクトリにあるページの共通のレイアウトを定義するファイル
- `layout.tsx`：`page.tsx` で定義したレイアウトを使用して、ページのレイアウトを定義するファイル
- `header.tsx`：`layout.tsx` で定義したレイアウトを使用して、ページのヘッダーを定義するファイル

`page.tsx` ファイルを編集して表示が切り替えることを確認してみましょう。

```tsx:app/page.tsx
export default function Home() {
  return (
    <div>
      <h1>新着記事</h1>
      <ul>
        <li>記事1</li>
        <li>記事2</li>
        <li>記事3</li>
      </ul>
    </div>
  )
}
```

ファイルを編集した後に http://localhost:3000/ にアクセスすると、以下のように表示が切り替わっていることが確認できます。

![app/page.tsx で変更した内容がブラウザに反映されている](https://storage.googleapis.com/zenn-user-upload/630028a2dc78-20230113.png)

続いて `/articles/{slug}` というパスにアクセスしたときに、記事の詳細を表示するページを作ってみましょう。`app` ディレクトリの構造と URL のパスがマッピングされているので、`app/articles/[slug]` というディレクトリを作成します。

```bash
mkdir app/articles/[slug]
```

作成した URL パスの UI を担当するファイルは `page.tsx` という名前で作成します。

```bash
touch app/articles/[slug]/page.tsx
```

`app/articles/[slug]/page.tsx` ファイルを以下のように編集しましょう。動的なパスの値（`slug`）を取得するために、引数から `params` を受け取っています。

:::message
将来的に `params` の型は [TypeScript plugin](https://beta.nextjs.org/docs/configuring/typescript#using-the-typescript-plugin) により自動で生成されるようになる予定です。
:::

```tsx:app/articles/[slug]/page.tsx
export default function Article({ params }: { params: { slug: string } }) {
  return (
    <div>
      <h1>記事の詳細</h1>
      <p>記事のスラッグ: {params.slug}</p>
    </div>
  );
}

```

http://localhost:3000/articles/next-js-app-dir-tutorial にアクセスすると、上記で編集したファイルの内容が表示されていることが確認できます。

![app/articles/[slug]/page.tsx で変更した内容がブラウザに反映されている](https://storage.googleapis.com/zenn-user-upload/cfc8be5edf6e-20230114.png)

Layout も編集してみましょう。`app/layout.tsx` ファイルは Root Layout と呼ばれています。Root Layout はすべてのページに対して適用されるレイアウトです。Next.js は `<html>` や `<body>` タグを自動的に生成しないので、`app/layout.tsx` ファイルで必ず定義する必要があります。

```tsx:app/layout.tsx
import Link from "next/link";

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <head />
      <body>
        <header>
          <h1>
            <Link href="/">ブログ</Link>
          </h1>
          <Link href="/articles/new">記事を書く</Link>
        </header>
        {children}
        <footer>
          <small>© 2023 azukiazusa</small>
        </footer>
      </body>
    </html>
  );
}
```

http://localhost:3000/ と http://localhost:3000/articles/next-js-app-dir-tutorial にアクセスして、どちらのページも同じレイアウトが適用されていることを確認してみましょう。

![/ のパスにレイアウトが提供されている](https://storage.googleapis.com/zenn-user-upload/64b67e0d1411-20230114.png)
![articles/next-js-app-dir-tutorial のパスにレイアウトが提供されている](https://storage.googleapis.com/zenn-user-upload/8c775202e4c0-20230114.png)

## Chakra UI を導入する

ここまで作成したアプリケーションは少し味気ないので、スタイリングのために [Chakra UI](https://chakra-ui.com/) を導入してみましょう。Chakra UI は React で使える UI ライブラリです。コンポーネントが多く提供されているうえ、カスタマイズ性に優れているのが特徴です。

まずは Chakra UI をインストールします。

```bash
npm i @chakra-ui/react @emotion/react@^11 @emotion/styled@^11 framer-motion@^6
```

### Provider を設定する

Chakra UI を使うためには `ChakraProvider` をアプリケーションのルートに設定する必要があります。`app` ディレクトリにおいては `app/layout.tsx` がルート要素になります。ここに `ChakraProvider` を設定してみましょう。

```diff tsx:app/layout.tsx
  import Link from "next/link";
+ import { ChakraProvider } from "@chakra-ui/react";

  export default function RootLayout({
    children,
  }: {
    children: React.ReactNode;
  }) {
    return (
      <html lang="ja">
        <head />
        <body>
+         <ChakraProvider>
            <header>
              {/* ... */}
+         </ChakraProvider>
        </body>
      </html>
    );
  }
```

しかし、このままではコンパイルエラーが発生します。

![Failed to compileと表示され、エラーメッセージが表示されている](https://storage.googleapis.com/zenn-user-upload/a100325c9817-20230114.png)

### Server Component と Client Component を使い分ける

これは `app` ディレクトリ内のコンポーネントがデフォルトで React Server Component として扱われることが原因です。[Server Component](https://reactjs.org/blog/2020/12/21/data-fetching-with-react-server-components.html) はコンポーネントをサーバーサイドでのみレンダリングする仕組みです。

Server Component はクライアントに JavaScript を送信しない、データベースや GraphQL エンドポイントへのアクセスをより近い場所で行うことができる、などのメリットがあります。そのため最初のページの読み込みが早くなり、クライアント側の JavaScript バンドルサイズも小さくなります。

しかし、Server Component にはいくつかの制限があります。
- 状態を持たないので `useState` のようなフックや `Context` は使えない
- `useEffect` のようなフックは使えない
- ブラウザのみ利用可能な API は使えない
- `onClick` や `onChange` のようなイベントハンドラーは使えない

`useState` を使用したりイベントハンドラを利用するコンポーネントは Client Component として扱う必要があります。`app` ディレクトリ内のコンポーネントを Client Component として扱うにはファイルの先頭で `"use client"` を宣言します。

このように Server Component と Client Compoennt は互いに長所と短所を補い合っているため、適切に使い分ける必要があります。

`<ChakraProvider>` は内部で `useState` を使用しているので Server Component として扱うことができません。そのためコンパイルエラーが発生したのです。

この問題を解決するために、`<ChakraProvider>` を Server Component として扱うのではなく、Client Component として扱うように設定する必要があります。

直接 `<ChakraProvider>` を import するのではなく、`"use client"` を宣言したファイルラップして使うことで Client Compoent として扱うことができます。`app/Provider.tsx` を作成します。

```tsx:app/Provider.tsx
"use client";

import { ChakraProvider } from "@chakra-ui/react";

export default function Provider({ children }: { children: React.ReactNode }) {
  return <ChakraProvider>{children}</ChakraProvider>;
}
```

そして `app/layout.tsx` で `<ChakraProvider>` を `<Provider>` に置き換えます。

```diff tsx:app/layout.tsx
  import Link from "next/link";
- import { ChakraProvider } from "@chakra-ui/react";
+ import Provider from "./Provider";

  export default function RootLayout({
    children,
  }: {
    children: React.ReactNode;
  }) {
    return (
      <html lang="ja">
        <head />
        <body>
-         <ChakraProvider>
+         <Provider>
            <header>
              {/* ... */}
-         </ChakraProvider>
+         </Provider>
        </body>
      </html>
    );
  }
```

これでコンパイルエラーが解消され、Next.js アプリケーションが正常に起動するようになりました。

### ヘッダーコンポーネントの作成

それでは Chakra UI を利用してヘッダーを作りましょう。`app` ディレクトリでは従来の `page` ディレクトリと異なり、`page.tsx` のような特殊なファイル名を使わない限り自由にファイルを配置できますので、`app` ディレクトリ配下に `Header.tsx` を作成します。

Chakra UI のコンポーネントはすべて Client でのみ動作するため、`"use client"` を宣言する必要があります。

```tsx:app/Header.tsx
"use client";
import { Box, Flex, Heading, Button } from "@chakra-ui/react";
import NextLink from "next/link";

export default function Header() {
  return (
    <Box as="header">
      <Flex
        bg="white"
        color="gray.600"
        minH={"60px"}
        py={{ base: 2 }}
        px={{ base: 4 }}
        borderBottom={1}
        borderStyle="solid"
        borderColor="gray.200"
        align="center"
      >
        <Flex flex={1} justify="space-between" maxW="5xl" mx="auto">
          <Heading as="h1" size="lg">
            <NextLink href="/">Blog App</NextLink>
          </Heading>
          <Button
            as={NextLink}
            fontSize="sm"
            fontWeight={600}
            color="white"
            bg="orange.400"
            href="/articles/new"
            _hover={{
              bg: "orange.300",
            }}
          >
            記事を書く
          </Button>
        </Flex>
      </Flex>
    </Box>
  );
}
```

同様に、`app/Main.tsx` と `app/Footer.tsx` も作成します。

```tsx:app/Main.tsx
"use client";
import { Container } from "@chakra-ui/react";

export default function Main({ children }: { children: React.ReactNode }) {
  return (
    <Container
      as="main"
      maxW="container.lg"
      mt="4"
      minH="calc(100vh - 115px - 1rem)"
    >
      {children}
    </Container>
  );
}
```

```tsx:app/Footer.tsx
"use client";

import { Container, Box, Text } from "@chakra-ui/react";

export default function Footer() {
  return (
    <Box bg="gray.50" color="gray.700">
      <Container maxW="5xl" py={4}>
        <Text as="small">© 2023 azukiazusa</Text>
      </Container>
    </Box>
  );
}
```

そして `app/layout.tsx` で `<Header>` と `<Main>`、`<Footer>` を配置します。

```diff tsx:app/layout.tsx
  import Link from "next/link";
  import Provider from "./Provider";
+ import Header from "./Header";
+ import Main from "./Main";
+ import Footer from "./Footer";

  export default function RootLayout({
    children,
  }: {
    children: React.ReactNode;
  }) {
    return (
      <html lang="ja">
        <head />
        <body>
          <Provider>  
+           <Header />
+           <Main>{children}</Main>
+           <Footer />
          </Provider>
        </body>
      </html>
    );
  }
```

これで基本的なレイアウトが完成しました。期待どおりに表示されていることを確認してみましょう。

![Chakra UI を適用した後のレイアウト](https://storage.googleapis.com/zenn-user-upload/ef82b2d3c41e-20230114.png)


## 記事の一覧を表示する




## 参考

- [React Server Componentsの仕組み：詳細ガイド | POSTD](https://postd.cc/how-react-server-components-work/)