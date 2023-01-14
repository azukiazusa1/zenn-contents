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

`<ChakraProvider>` と同様に Chakra UI のコンポーネントはすべて Client でのみ動作するため、ラップして `"use client"` を宣言する必要があります。

Chakra UI のコンポーネントを使うたびに `"use client"` を宣言するのは手間ですので、`app/common/components/index.tsx` でまとめて Chatra UI のコンポーネントを export するようにします。

```tsx:app/common/components/index.tsx
"use client";
export * from "@chakra-ui/react";
```

### ヘッダーコンポーネントの作成

それでは Chakra UI を利用してヘッダーを作りましょう。`app` ディレクトリでは従来の `page` ディレクトリと異なり、`page.tsx` のような特殊なファイル名を使わない限り自由にファイルを配置できますので、`app` ディレクトリ配下に `Header.tsx` を作成します。
```tsx:app/Header.tsx
import { Box, Flex, Heading, Button } from "./common/components";
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
import { Container } from "./common/components";

export default function Main({ children }: { children: React.ReactNode }) {
  return (
    <Container
      as="main"
      maxW="container.lg"
      my="4"
      minH="calc(100vh - 115px - 2rem)"
    >
      {children}
    </Container>
  );
}
```

```tsx:app/Footer.tsx
import { Container, Box, Text } from "./common/components";

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

API から記事の一覧を取得してトップページに表示してみましょう。API はあらかじめ `pages/api/` ディレクトリに用意されています。

従来の Next.js で提供されていた `getServerSideProps` や `getStaticPorps` は `app` ディレクトリではサポートされていません。その代わりに API からのデータを取得は、サーバーコンポーネント内で `async/await` を使って行います。

`app` ディレクトリによるデータフェッチングは基本的に Fetch API を使用します。Fetch API は Web API にネイティブで備わっている機能ですが、Next.js で使う際には次のように拡張されています。

- 自動的にリクエストの重複排除する
- デフォルトで動的関数の前に呼ばれるリクエストがキャッシュされる
- 独自のキャッシュ戦略として `revalidate` をサポートする

クライアントコンポーネントでもデータフェッチングを行うことができますが、常にサーバーコンポーネント内で行うことを推奨されています。
- データベースなどバックエンドのリソースに直接アクセスできる
- アクセストークンなどの機密情報をクライアントに露出しない
- データの取得とレンダリングを同一環境下で行うのでクライアントとサーバーの通信と、クライアント上のメインスレッドの作業を削減できる
- 複数のデータフェッチングを 1 つのリクエストで行うことができる
- データソースにより近い場所でデータを取得することで、レイテンシを削減できる

それでは実際に Server Component でデータフェッチングを行ってみましょう。先に型定義を用意しておきます。

```ts:app/types.ts
export type Article = {
  id: number;
  title: string;
  content: string;
  slug: string;
  createdAt: string;
  updatedAt: string;
}

export type Comment = {
  id: number;
  body: string;
  articleId: number;
  createdAt: string;
  updatedAt: string;
}
```

`app/pages.tsx` 内で `api/articles` から記事の一覧を取得します。

```tsx:app/pages.tsx
import type { Article } from "./types";

async function getArticles() {
  const res = await fetch("http://localhost:3000/api/articles");

  // エラーハンドリングを行うことが推奨されている
  if (!res.ok) {
    throw new Error("Failed to fetch articles");
  }

  const data = await res.json();
  return data.articles as Article[];
}

export default async function Home() {
  const articles = await getArticles();

  return (
    <div>
      <h1>新着記事</h1>
      <ul>
        {articles.map((article) => (
          <li key={article.id}>{article.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

上記のようにコンポーネントで `async/await` を使用することで自然にデータを取得して使用できます。

### データのキャッシュ

データフェッチングのキャッシュについても考えてみましょう。デフォルトでは `fetch` を使用すると自動的にデータを取得した後にキャッシュされます。これは `fetch` のオプションはデフォルトで `{ cache: "force-cache" }` が設定されていることを意味します。

ここでは、新着の記事の一覧を取得しているので、データの更新が頻繁に行われる可能性があります。そのため、データのキャシュは行わないように設定します。

`fetch` を実行するたびに新しいデータを取得するようにするには、`cache: "no-source"` を設定します。

```tsx:app/pages.tsx
const res = await fetch("http://localhost:3000/api/articles", {
  cache: "no-store",
});
```

## ローディング UI

新着記事一覧を取得に 1500 ミリ秒かかるようにディレイを設定しているのですが、その間データの取得と関係のないヘッダー部分も含めて何も表示されないのはユーザーフレンドリーではありません。そこで、データの取得中にローディング UI を表示するようにしてみましょう。

Next.js 13 では `app` ディレクトリ内の `loading.tsx` という特殊なファイルがローディング UI を表示する役割を果たします。`loading.tsx` はサーバーでデータを取得している最中（= サーバーコンポーネントの Promise が解決するまで）に表示され、レンダリングが完了すると新しいコンテンツを表示します。この挙動は [Suspense](https://ja.reactjs.org/docs/concurrent-mode-suspense.html) における `fallback` と同じです。

```tsx:app/loading.tsx
import { Box, Spinner } from "./common/components";

export default function Loading() {
  return (
    <Box justifyContent="center" display="flex">
      <Spinner color="orange.400" size="xl" />
    </Box>
  );
}
```

`loading.tsx` により、記事の一覧を取得するまでローディング UI が表示されるようになりました。`loading.tsx` は同じディレクトリ内の `page.tsx` をラップするように配置するので、ヘッダーなどのレイアウトは即座に表示されます。

![ローディング UI が表示されている](https://storage.googleapis.com/zenn-user-upload/73b6008a650f-20230114.gif)

### エラーハンドリング

サーバーコンポーネント内で例外が throw された場合、`error.tsx` の内容が表示されます。`error.tsx` は同じディレクトリ内にある `page.tsx` ファイルを [Error Boundary](https://reactjs.org/docs/error-boundaries.html) でラップします。`error.tsx` は必ず Client Component として扱われます。

Error コンポーネントは以下の Props を受け取ります。
- `error`：例外オブジェクト
- `reset`：例外が発生したコンポーネントを再レンダリングするための関数

```tsx:app/error.tsx
"use client"; // Error components must be Client components

import { useEffect } from "react";
import { Heading, Button } from "./common/components";

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  useEffect(() => {
    console.error(error);
  }, [error]);

  return (
    <div>
      <Heading mb={4}>予期せぬエラーが発生しました。</Heading>
      <Button onClick={() => reset()}>Try again</Button>
    </div>
  );
}
```

`app.page.tsx` で意図的に例外を発生させることで、エラーハンドリングの動作を確認してみましょう。

```diff tsx:app/page.tsx
  async function getArticles() {
    const res = await fetch("http://localhost:3000/api/articles", {
      cache: "no-store",
    });

+   throw new Error("Failed to fetch articles");
```

![エラーが発生した場合の UI](https://storage.googleapis.com/zenn-user-upload/50db0d25ddec-20230114.png)

### ArticleList コンポーネント

最後に記事の表示を担当するコンポーネントを作成して見た目を整えましょう。う。まずは `ArticleCard` コンポーネントを作成します。

```tsx:app/components/ArticleCard.tsx
import {
  Card,
  CardHeader,
  CardBody,
  CardFooter,
  Heading,
  Text,
} from "./common/components";
import NextLink from "next/link";
import { Article } from "./types";

export default function ArticleCard({ article }: { article: Article }) {
  const formattedDate = new Date(article.createdAt).toLocaleDateString(
    "ja-JP",
    {
      year: "numeric",
      month: "long",
      day: "numeric",
    }
  );
  return (
    <Card
      as={"li"}
      _hover={{
        boxShadow: "xl",
      }}
    >
      <NextLink href={`/articles/${article.slug}`}>
        <CardHeader>
          <Heading size="md">{article.title}</Heading>
        </CardHeader>
        <CardBody>
          <Text>{article.content.substring(0, 200)}...</Text>
        </CardBody>
        <CardFooter>
          <Text fontSize="sm" color="gray.600">
            {formattedDate}
          </Text>
        </CardFooter>
      </NextLink>
    </Card>
  );
}
```

記事のカードを一覧で表示するために、`ArticleList` コンポーネントを作成します。

```tsx:app/components/ArticleList.tsx
import { VStack } from "./common/components";
import ArticleCard from "./ArticleCard";
import { Article } from "./types";

export default function ArticleList({ articles }: { articles: Article[] }) {
  return (
    <VStack spacing={4} as="ul">
      {articles.map((article) => (
        <ArticleCard key={article.id} article={article} />
      ))}
    </VStack>
  );
}
```

`ArticleList` コンポーネントを `app/page.tsx` に組み込みます。

```tsx:app/page.tsx
import ArticleList from "./ArticleList";
import { Heading } from "./common/components";

// ...

export default async function Home() {
  const articles = await getArticles();

  return (
    <div>
      <Heading as="h1" mb={4}>
        新着記事
      </Heading>
      <ArticleList articles={articles} />
    </div>
  );
}
```

http://localhost:3000 にアクセスすると、次のように記事の一覧が表示されるはずです。

![ArticleList コンポーネントにより描画された記事一覧ページ](https://storage.googleapis.com/zenn-user-upload/2dc15fda846f-20230114.png)

## 記事の詳細ページ



## 参考

- [React Server Componentsの仕組み：詳細ガイド | POSTD](https://postd.cc/how-react-server-components-work/)