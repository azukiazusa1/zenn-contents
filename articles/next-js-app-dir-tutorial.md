---
title: "Next.js 13 app directory で記事投稿サイトを作ってみよう"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,react,chakraui]
published: true
---

Next.js 13 から新たに App Router と呼ばれる機能が追加されました。これは従来の `pages` ディレクトリとは異なるレイアウトシステムです。App Router には以下のような特徴があります。

- ルーティング：`pages` ディレクトリではページのルーティングはファイル名によって決まっていました。例えば `pages/about.js` というファイルは `/about` というパスに対応します。App Router ではルーティングに対応するファイルは `page.js` という固定の名前になります。`/about` というパスに対応するファイルは `app/about/page.js` という名前になるのです。`page.js` 以外にも共通されたレイアウトを担当する `layout.js`、ローディング UI を表示する `loading.js` などさまざまな特殊なファイルが存在します。
- レンダリング：App Router 内のコンポーネントはデフォルトで Server Component として扱われます。Client Component として扱いたい場合には `"use client"` をファイルの先頭で宣言する必要があります。
- データフェッチング：従来の `getStaticProps` や `getServerSideProps` は App Router では使えません。代わりに Server Component で `async/await` を使用してデータを取得できます。またデータの取得時にキャッシュやリクエストの重複排除を活用するため `fetch` API を利用します。
- キャッシュ：`fetch` API 用いてデータを取得する際にはデフォルトで Next.js による HTTP キャッシュが有効になっています。またクライアントサイドでのキャッシュにより、クライアントでのナビゲーションでは余分なリクエストが発生しません。

App Router を使うことで、直感的なレイアウトシステムだけでなく、パフォーマンスの向上が見込めます。この記事では `app` directory を使って簡単な記事投稿サイトを作り、新しい機能の特徴を体験してみましょう。

完成後のコードは以下のリポジトリにあります。

https://github.com/azukiazusa1/nextjs-app-dir-example/tree/complete

## 開発環境の準備

まずは、Next.js アプリケーションを作成します。[azukiazusa1/nextjs-app-dir-example](https://github.com/azukiazusa1/nextjs-app-dir-example) のレポジトリからクローンしていただくと、バックエンドの API が準備済みの状態になっています。

```bash
git clone https://github.com/azukiazusa1/nextjs-app-dir-example.git
```

自分で作成する場合は、以下のコマンドを実行してください。

```bash
npx create-next-app@latest
```

以下のコマンドでパッケージをインストールして、開発環境を起動してみましょう。

```bash
npm run install
npm run dev
```

http://localhost:3000/ にアクセスすると、以下のような画面が表示されます。

![Next.js で開発環境を起動したデフォルトで表示されるブラウザの画面](https://storage.googleapis.com/zenn-user-upload/74542e682840-20230113.png)

## App Router の概要

まずは App Router に初期状態では以下のファイルが存在します。

- `page.tsx`：ルーティングに対応する UI を定義するファイル。
- `layout.tsx`：アプリケーションのルートレイアウト。すべてのページ共通で使われるナビゲーションヘッダーなどに加えて `<html>` タグや `<body>` タグを設定する。

### `page.tsx` ファイルの編集

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

ファイルを編集した後 http://localhost:3000/ にアクセスすると、以下のように表示が切り替わっていることが確認できます。

![app/page.tsx で変更した内容がブラウザに反映されている](https://storage.googleapis.com/zenn-user-upload/630028a2dc78-20230113.png)

続いて `/articles/{slug}` というパスへアクセスしたときに、記事の詳細を表示するページを作ってみましょう。App Router の構造と URL のパスがマッピングされているので、`app/articles/[slug]` というディレクトリを作成します。

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

### `layout.tsx` ファイルの編集

Layout も編集してみましょう。`app/layout.tsx` ファイルは Root Layout と呼ばれています。Root Layout はすべてのページに対して適用されるレイアウトです。Next.js は `<html>` や `<body>` タグを自動的に生成しないので、`app/layout.tsx` ファイルで必ず定義する必要があります。

```tsx:app/layout.tsx
import Link from "next/link";

export const metadata = {
  title: "Create Next App",
  description: "Generated by create next app",
  themeColor: "#ffffff",
};

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

ここまで作成したアプリケーションは少し味気ないので、スタイリングのために [Chakra UI](https://chakra-ui.com/) を導入します。Chakra UI は React で使える UI ライブラリです。UI コンポーネントが多く提供されているうえ、カスタマイズ性に優れているのが特徴です。

まずは Chakra UI をインストールします。

```bash
npm i @chakra-ui/react @emotion/react@^11 @emotion/styled@^11 framer-motion@^6
```

### Provider を設定する

Chakra UI を使うためには `ChakraProvider` をアプリケーションのルートに設定する必要があります。App Router においては `app/layout.tsx` がルート要素になります。ここに `ChakraProvider` を設定してみましょう。

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

これは App Router 内のコンポーネントがデフォルトで React Server Component として扱われることが原因です。

### Server Component と Client Component を使い分ける

[Server Component](https://reactjs.org/blog/2020/12/21/data-fetching-with-react-server-components.html) はコンポーネントをサーバーサイドでのみレンダリングする仕組みです。Server Component は以下のようなメリットが存在します。

- クライアントに JavaScript を送信しない。
- データベースや GraphQL エンドポイントへのアクセスをより近い場所で行うことができる。
  
そのため最初のページの読み込みが早くなり、クライアント側の JavaScript バンドルサイズも小さくなるといった恩恵を受けることができます。基本的にはデフォルトの Server Component のまま利用するべきです。

しかし、Server Component にはいくつかの制限があります。
- 状態を持たないので `useState` のようなフックや `Context` は使えない
- `useEffect` のようなライフサイクルフックは使えない
- `localStorage` のようなブラウザのみ利用可能な API は使えない
- `onClick` や `onChange` のようなイベントハンドラーは使えない

`useState` を利用して状態管理をする、イベントハンドラを利用してインタラクティブなアクションを行うコンポーネントは Client Component として扱う必要があります。App Router 内のコンポーネントを Client Component として扱うにはファイルの先頭で `"use client"` を宣言します。

このように Server Component と Client Compoennt は互いに長所と短所を補い合っているため、適切に使い分ける必要があります。

`<ChakraProvider>` は内部で `useState` を使用しているので Server Component として扱うことができません。これがコンパイルエラーが発生した原因です。

この問題を解決するために、`<ChakraProvider>` を Server Component として扱うのではなく、Client Component として扱うように設定する必要があります。

`<ChakraProvider>` のようなサードパーティのコンポーネントを Client Component として扱うためには、`"use client"` を宣言したファイルでラップします。`app/Provider.tsx` ファイルを作成しましょう。

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

Chakra UI のコンポーネントはすべて `<ChakraProvider>` に依存しているため、Client Component でのみ動作します。そのため UI コンポーネントを使用する場合にはラップして `"use client"` を宣言する必要があります。

Chakra UI のコンポーネントを使うたびに `"use client"` を宣言するのは手間がかかります。`app/common/components/index.tsx` でまとめて Chatra UI のコンポーネントを export して Client Component として使えるようにします。

```tsx:app/common/components/index.tsx
"use client";
export * from "@chakra-ui/react";
```

これで Chakra UI のコンポーネントを使いたい場合には

```tsx
import { Button } from "./common/components";`
```

のように書くことで `"use client"` を意識する必要がなくなりました。

### ヘッダーコンポーネントの作成

それでは Chakra UI を利用して共通のヘッダーコンポーネントを作りましょう。App Router では従来の `page` ディレクトリと異なり、`page.tsx` のような特殊なファイル名を使わない限り自由にファイルを配置できます。`layout.tsx` ファイルの近くにヘッダコンポーネントを配置したいので、App Router 配下に `Header.tsx` を作成します。
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
    <Box bg="gray.50" color="gray.700" as="footer">
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

Next.js でデータフェッチングを行うにはサーバーサイドで行うことが一般的です。しかし、従来の Next.js で提供されていた `getServerSideProps` や `getStaticPorps` は App Router ではサポートされていません。その代わりに API からのデータを取得は、Server Component 内で `async/await` を使って行います。Server Component では非同期コンポーネントとしても動作します。

App Router によるデータフェッチングは基本的に Fetch API を使用します。Fetch API は Web API にネイティブで備わっている機能ですが、Next.js で使う際には次のように拡張されています。

- 自動的にリクエストの重複排除する
- デフォルトで動的関数（`cookies()`, `headers()`, `useSearchParams()`）の前に呼ばれるリクエストが HTTP キャッシュされる
- 独自のキャッシュ戦略として `revalidate` をサポートする

Client Compoennt でもデータフェッチングを行うことができますが、以下の理由から常に Server Component 内で行うことを推奨されています。
- データベースなどバックエンドのリソースに直接アクセスできる
- アクセストークンなどの機密情報をクライアントに露出しない
- データの取得とレンダリングを同一環境下で行うのでクライアントとサーバーの通信と、クライアント上のメインスレッドの作業を削減できる
- 複数のデータフェッチングを 1 つのリクエストで行うことができる
- データソースにより近い場所でデータを取得することで、レイテンシを削減できる

それでは実際に Server Component でデータフェッチングを行ってみましょう。先に型定義を用意しておきます。`app/types.ts` ファイルを作成します。

```ts:app/types.ts
export type Article = {
  id: number;
  title: string;
  content: string;
  slug: string;
  createdAt: string;
  updatedAt: string;
};

export type Comment = {
  id: number;
  body: string;
  articleId: number;
  createdAt: string;
  updatedAt: string;
  author: Author;
};

export type Author = {
  name: string;
  avatarUrl: string;
};
```

`app/pages.tsx` 内で `http://localhost:3000/api/articles` から記事の一覧を取得します。

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

`getArticles` 内の処理は一般的な `fetch` の使い方と同じです。コンポーネント内で例外を投げた場合には後述する `error.tsx` によりエラー画面が表示されます。

`Home` コンポーネントで `async/await` を使用することにより自然な流れでデータを取得して使用できます。

### データのキャッシュ

データフェッチングのキャッシュについても考えてみましょう。デフォルトでは `fetch` を使用すると自動的にデータを取得した後にキャッシュされます。これは `fetch` のオプションはデフォルトで `{ cache: "force-cache" }` が設定されていることを意味します。`force-cache` オプションは `getStaticProp` と近い働きです。

ここでは新着の記事の一覧を取得しているので、データの更新が頻繁に行われる可能性があります。そのため、データのキャシュは行わないようにして、毎回リクエストを行うように設定します。

`fetch` を実行するたびに新しいデータを取得するようにするには、`cache: "no-store"` を設定します。`no-store` は `getServerSideProp` と近い働きです。

```tsx:app/pages.tsx
const res = await fetch("http://localhost:3000/api/articles", {
  cache: "no-store",
});
```

## ローディング UI

新着記事一覧の取得には 1500 ミリ秒かかるようにディレイを設定しています。その間データの取得と関係のないヘッダー部分も含めて何も表示されないのは、ユーザーフレンドリーではありません。そこで、データの取得中にローディング UI を表示するようにしてみましょう。

Next.js 13 では App Router 内の `loading.tsx` という特殊なファイルがローディング UI を表示する役割を果たします。`loading.tsx` はサーバーでデータを取得している最中（= サーバーコンポーネントの Promise が解決するまで）に表示され、レンダリングが完了すると新しいコンテンツを表示します。この挙動は [Suspense](https://ja.reactjs.org/docs/concurrent-mode-suspense.html) における `fallback` と同じです。

イメージとしては以下のような感じです。

```tsx
<html lang="ja">
  <head />
  <body>
    <Provider>
      <Header />
      <Main>
        <Suspense fallback={<Loading />}>
          {/* children には page.tsx のコンテンツが挿入される */}
          {children}
        </Suspense>
      </Main>
      <Footer />
    </Provider>
  </body>
</html>
```

`app/loading.tsx` を次のように作成します。

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

サーバーコンポーネント内で例外が throw された場合、`error.tsx` の内容が表示されます。`error.tsx` は同じディレクトリ内にある `page.tsx` ファイルを [Error Boundary](https://reactjs.org/docs/error-boundaries.html) でラップします。

`error.tsx` ファイルは次のように動作するイメージです。

```tsx
<html lang="ja">
  <head />
  <body>
    <Provider>
      <Header />
      <Main>
        <ErrorBoundary fallback={<Error />}>
          {/* children には page.tsx のコンテンツが挿入される */}
          {children}
        </Suspense>
      </Main>
      <Footer />
    </Provider>
  </body>
</html>
```

Error コンポーネントは以下の Props を受け取ります。
- `error`：throw された例外オブジェクト
- `reset`：例外が発生したコンポーネントを再レンダリングするための関数
  
また `error.tsx` は必ず Client Component として扱われます。

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

最後に記事の表示を担当するコンポーネントを作成して見た目を整えましょう。まずは `ArticleCard` コンポーネントを作成します。

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
      minW="100%"
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

記事のカードを一覧で表示する `ArticleList` コンポーネントを作成します。

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

記事の詳細ページを作成します。ここでは記事の本文を取得して表示するとともに、記事に対するコメントを表示する機能を実装します。特定の記事は `api/articles/{slug}` から、記事に対するコメントは `api/articles/{slug}/comments` から取得します。

それぞれのキャッシュ戦略も考えてみましょう。記事の本文は、更新頻度が高くないと想定できるので、一定期間キャッシュしておくことができます。キャッシュの生存期間は `fetch` のオプションに `next.revalidate` を指定することで設定できます。これは従来の ISR に近い機能です。

:::message
`revalidate` は `fetch` のオプションとして渡すだけでなく、ルート単位の設定もできます。ルート単位で設定するには `page` または `layout` ファイルで以下のように記述します。

```tsx
export const revalidate = 60
```
:::

一方で、コメントは投稿した後即座に反映されないと不自然ですので、キャッシュを利用しないほうが良いでしょう。記事の一覧取得と同様に、`fetch` のオプションに `cache: "no-store"` を指定します。

```tsx:app/pages/articles/[slug].tsx
import { notFound } from "next/navigation";
import { Article, Comment } from "../../types";

const getArticle = async (slug: string) => {
  const res = await fetch(`http://localhost:3000/api/articles/${slug}`, {
    next: { revalidate: 60 },
  });

  if (res.status === 404) {
    // notFound 関数を呼び出すと not-fount.tsx を表示する
    notFound();
  }

  if (!res.ok) {
    throw new Error("Failed to fetch article");
  }

  const data = await res.json();
  return data as Article;
};

const getComments = async (slug: string) => {
  const res = await fetch(
    `http://localhost:3000/api/articles/${slug}/comments`,
    {
      cache: "no-store",
    }
  );

  if (!res.ok) {
    throw new Error("Failed to fetch comments");
  }

  const data = await res.json();
  return data.comments as Comment[];
};
```

記事を取得する際に API が 404 を返した場合は `next/navigation` の `notFoutd` 関数を呼び出しています。この関数が呼ばれるともっとも近いディレクトリにある `not-found.tsx` が表示されます。

`not-found.tsx` も作成しておきましょう。

```tsx:app/pages/articles/not-found.tsx
import { Heading, Button } from "../../common/components";
import NextLink from "next/link";

export default function NotFound() {
  return (
    <div>
      <Heading mb={4}>お探しの記事が見つかりませんでした。</Heading>
      <Button as={NextLink} href="/">
        トップへ戻る
      </Button>
    </div>
  );
}
```

存在しない記事の URL にアクセスすると、以下のように表示されます。

![存在しない記事の URL にアクセスしたときの表示](https://storage.googleapis.com/zenn-user-upload/f74e5bc1a7c2-20230115.png)

記事の詳細の表示に戻りましょう。コンポーネント内で `getArticle` と `getComments` を呼び出して、取得したデータを表示します。依存関係のない複数の API を呼び出す場合は処理が並列になるように `Promise.all` を使うことが推奨されます。


```tsx:app/pages/articles/[slug].tsx
export default async function ArticleDetail({
  params,
}: {
  params: { slug: string };
}) {
  const articlePromise = getArticle(params.slug);
  const commentsPromise = getComments(params.slug);

  const [article, comments] = await Promise.all([
    articlePromise,
    commentsPromise,
  ]);

  return (
    <div>
      <h1>{article.title}</h1>
      <p>{article.content}</p>
      <h2>Comments</h2>
      <ul>
        {comments.map((comment) => (
          <li key={comment.id}>{comment.body}</li>
        ))}
      </ul>
    </div>
  );
}
```

これで記事詳細ページを訪れると、記事の内容とコメントの一覧が表示されるようになりました。しかし 1 点問題があります。記事の表示に時間がかかりすぎる点です。

記事の取得には 1000 ミリ秒、コメントの一覧には 3000 ミリ秒の遅延を設定しています。記事の本文を閲覧するだけであれば本来は 1000 ミリ秒で表示できるはずです。しかし、`Promise.all` でコメント一覧の取得の完了も同時に待機しているため、記事の内容の表示に 3000 ミリ秒かかってしまっています。

![記事の詳細を表示するまでローディングインジケータが表示されている](https://storage.googleapis.com/zenn-user-upload/4d5e801a532a-20230114.gif)

ユーザーが記事詳細ページを訪れる目的は記事の本文を閲覧することであり、コメントの一覧はあくまで補足情報です。そのため、コメントの一覧の取得の完了まで待つのは好ましくありません。

そこでコメント一覧の取得にストリーミングを使ってみましょう。ストリーミングを使うとコメントの一覧の取得が完了するまで待つことなく、記事の取得が完了したタイミングで記事の本文を表示できます。

### コメントをストリーミングで取得する

ストリーミングはページの HTML を小さなチャンクに分解してクライアントに漸進的に送信します。これにより、すべてのデータの取得が完了するまで待つことなく、ページの一部分から表示を開始できます。

ストリーミングを行う箇所を制御するためには [<Suspense>](https://beta.reactjs.org/reference/react/Suspense) で非同期コンポーネントをラップします。次のようにコメント一覧を取得する箇所を別のコンポーネントに分割し、`<Suspense>` でラップします。

```tsx:app/pages/articles/[slug].tsx
export default async function ArticleDetail({
  params,
}: {
  params: { slug: string };
}) {
  const articlePromise = getArticle(params.slug);
  const commentPromise = getComments(params.slug);

  const article = await articlePromise;

  return (
    <div>
      <h1>{article.title}</h1>
      <p>{article.content}</p>
      <h2>Comments</h2>
      <Suspense fallback={<div>Loading comments...</div>}>
        {/* @ts-expect-error 現状は jsx が Promise を返すと TypeScript が型エラーを報告するが、将来的には解決される */}
        <Comments commentPromise={commentPromise} />
      </Suspense>
    </div>
  );
}

async function Comments({
  commentPromise,
}: {
  commentPromise: Promise<Comment[]>;
}) {
  const comments = await commentPromise;
  return (
    <ul>
      {comments.map((comment) => (
        <li key={comment.id}>{comment.content}</li>
      ))}
    </ul>
  );
}
```

それでは動作を確認してみましょう。1000 ミリ秒ほど経過した後に記事の本文が表示され、その間コメント一覧には `Loading comments` と表示されています。その後さらに 2000 ミリ秒経過した後にコメントの一覧が表示されます。

![コメント一覧をストリーミングで取得している](https://storage.googleapis.com/zenn-user-upload/b091e1508dbb-20230114.gif)

### `<head>` タグ

記事の詳細ページでは SEO のためにも記事のタイトルを `<title>` タグを設定しておきたいものです。ルーティングごとに `<head>` タグを設定するには、`page.tsx` または `layout.tsx` ファイル内で `metadata` オブジェクトまたは `generateMetadata` 関数を名前付きエクスポートします。
設定した `metadata` の内容は ルートレイアウトの `<head />` タグ内に挿入されます。

- `metadata` オブジェクト：`<head>` タグの内容を静的に設定する
- `generateMetadata` 関数：`<head>` タグの内容を動的に設定する

`generateMetadata` 関数は 引数に params を受け取り、`async/await` を使って動的に値を取得して オブジェクトを形式で `<head>` を設定できます。第 2 引数の `parent` では上位のディレクトリに設定されている `metadata` を参照できます。

```tsx:app/pages/articles/[slug]/page.tsx
import type { Metadata, ResolvingMetadata } from 'next';

export async function generateMetadata({
  params,
}: {
  params: { slug: string };
  parent?: ResolvingMetadata;
}): Promise<Metadata> {
  const article = await getArticle(params.slug);
  return {
    title: article?.title,
    description: article?.content,
  };
}
```

`ArticleDetail` コンポーネントと同じリクエストを送信しているので一見非効率なように思えますが、Next.js により `fetch` を利用したリクエストは自動で重複排除されるのでパフォーマンスには影響しません。

metadata はルートディレクトリに近いセグメントから最も近いセグメントにある `page.tsx` ファイルの順番に評価されます。例えば `app/layout.tsx` では以下のように記述されています。

```tsx:app/layout.tsx
export const metadata = {
  title: 'Create Next App',
  description: 'Generated by create next app',
  themeColor: "#ffffff",
}
```

`title` と `desciption` は `app/pages/articles/[slug]/page.tsx` 設定した `metadata` のキーと重複しています。この場合、最もページから近い `page.tsx` で設定された `metadata` が優先されるので、記事の詳細ページでは `title` と `description` が記事のタイトルと本文になります。また、`themeColor` は `app/layout.tsx` のみで設定されているので、その内容がそのまま引き継がれています。

### スタイリング

最後に Chakra UI によるスタイリングを適用します。以下のコンポーネントを作成します。

- `app/articles/[slug]/ArticleContent.tsx`：記事のタイトルと本文を表示するコンポーネント
- `app/articles/[slug]/Comments.tsx`：コメントの一覧を表示するコンポーネント
- `app/articles/[slug]/LoadingComments.tsx`：コメントの読み込み中に表示されるコンポーネント

```tsx:app/articles/[slug]/ArticleContent.tsx
import {
  Card,
  CardHeader,
  CardBody,
  Text,
  Heading,
} from "../../common/components";
import { Article } from "../../types";

export default function ArticleContent({ article }: { article: Article }) {
  return (
    <Card as="article">
      <CardHeader>
        <Heading as="h1">{article.title}</Heading>
      </CardHeader>
      <CardBody>
        <Text as="p" fontSize="md">
          {article.content}
        </Text>
      </CardBody>
    </Card>
  );
}
```

```tsx:app/articles/[slug]/Comments.tsx
import {
  Card,
  CardBody,
  StackDivider,
  VStack,
  Text,
  Box,
  Avatar,
  Flex,
} from "../../common/components";
import { Comment } from "../../types";

export default async function Comments({
  commentPromise,
}: {
  commentPromise: Promise<Comment[]>;
}) {
  const comments = await commentPromise;

  if (comments.length === 0) {
    return (
      <Text as="p" fontSize="md">
        コメントはありません。
      </Text>
    );
  }
  return (
    <VStack
      divider={<StackDivider borderColor="gray.200" />}
      spacing={4}
      as="ul"
      align="stretch"
      px={4}
    >
      {comments.map((comment) => (
        <CommentItem key={comment.id} comment={comment} />
      ))}
    </VStack>
  );
}

function CommentItem({ comment }: { comment: Comment }) {
  return (
    <Flex as="li" listStyleType="none" align="center">
      <Avatar
        size="sm"
        name={comment.author.name}
        src={comment.author.avatarUrl}
        mr={4}
      />
      <Text fontSize="sm">{comment.body}</Text>
    </Flex>
  );
}
```

```tsx:app/articles/[slug]/LoadingComments.tsx
import {
  StackDivider,
  VStack,
  Flex,
  SkeletonCircle,
  Skeleton,
} from "../../common/components";

export default function LoadingComments({}) {
  return (
    <VStack
      divider={<StackDivider borderColor="gray.200" />}
      spacing={4}
      as="ul"
      align="stretch"
      px={4}
    >
      <CommentSkeltonItem />
      <CommentSkeltonItem />
      <CommentSkeltonItem />
    </VStack>
  );
}

function CommentSkeltonItem() {
  return (
    <Flex as="li" listStyleType="none" align="center">
      <SkeletonCircle size="8" mr={4} />
      <Skeleton height="14px" width="60%" />
    </Flex>
  );
}
```

そして、`app/articles/[slug]/index.tsx` でこれらのコンポーネントを使用します。

```tsx:app/articles/[slug]/index.tsx
import ArticleContent from "./ArticleContent";
import Comments from "./Comments";
import { Heading } from "../../common/components";
import LoadingComments from "./LoadingComments";

const getArticle = async (slug: string) => {
  // ...
}

const getComments = async (slug: string) => {
  // ...
}

export default async function ArticleDetail({
  params,
}: {
  params: { slug: string };
}) {
  const articlePromise = getArticle(params.slug);
  const commentPromise = getComments(params.slug);

  const article = await articlePromise;

  return (
    <div>
      <ArticleContent article={article} />
      <Heading as="h2" mt={8} mb={4}>
        Comments
      </Heading>
      <Suspense fallback={<LoadingComments />}>
        {/* @ts-expect-error 現状は jsx が Promise を返すと TypeScript が型エラーを報告するが、将来的には解決される */}
        <Comments commentPromise={commentPromise} />
      </Suspense>
    </div>
  );
}
```

最終的には以下のような表示になります。

![Chakra UI によってスタイリングされた記事詳細](https://storage.googleapis.com/zenn-user-upload/b4f23115c947-20230114.gif)

## 記事の作成

次に記事の作成機能を実装します。`/articles/new` というパスで記事の作成フォームを表示したいので、`app/articles/new/page.tsx` というファイルを作成します。フォームによる状態管理を行うので、Client Component として実装します。

```tsx:app/articles/new/index.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import {
  Heading,
  FormControl,
  FormLabel,
  Input,
  Textarea,
  Button,
} from "../../common/components";

export default function CreateArticle() {
  const router = useRouter();
  const [title, setTitle] = useState("");
  const [content, setContent] = useState("");
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setLoading(true);
    await fetch("/api/articles", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ title, content }),
    });
    setLoading(false);
    router.push("/");
  };

  return (
    <div>
      <Heading mb={4}>Create Article</Heading>

      <form onSubmit={handleSubmit}>
        <FormControl>
          <FormLabel>タイトル</FormLabel>
          <Input value={title} onChange={(e) => setTitle(e.target.value)} />

          <FormLabel>本文</FormLabel>
          <Textarea
            value={content}
            onChange={(e) => setContent(e.target.value)}
          />
          <Button
            type="submit"
            color="white"
            bg="orange.400"
            isLoading={loading || isPending}
            mt={4}
          >
            作成
          </Button>
        </FormControl>
      </form>
    </div>
  );
}
```

記事のタイトルと本文を入力するフォームを表示し、作成ボタンを押すと `fetch` で記事作成 API を呼び出します。API のコールが完了したら `router.push("/")` によりトップページに遷移します。

App Router 内で `useRouter` を使うためには `next/router` ではなく `next/navigation` をインポートする必要があります。

以下のように表示され、記事の作成もできるようになりました。

![Chakra UI によってスタイリングされた記事作成フォーム](https://storage.googleapis.com/zenn-user-upload/def5058c7279-20230115.png)

しかし 1 つ問題があります。記事の作成後、トップページに遷移したときに作成した記事が表示されていません。

![記事を投稿した後、記事の一覧画面に遷移すると投稿した記事が表示されていない](https://storage.googleapis.com/zenn-user-upload/724114ce65bd-20230115.gif)

これは `router.push` による画面遷移が Soft Navigation であるためです。Soft Navigation では遷移先のキャッシュが存在する場合再利用され、サーバーへの新たなリクエストが発生しません。

:::message
ここでのキャシュは Next.js が提供する `fetch` API の [HTTP キャッシュ](https://beta.nextjs.org/docs/data-fetching/fundamentals#caching-data)と異なります。[クライアントに in-memory でキャッシュ](https://beta.nextjs.org/docs/routing/linking-and-navigating#client-side-caching-of-rendered-server-components)されるものです。
:::

トップページのキャッシュが存在するため、古い記事一覧が表示されてしまっています。キャッシュを無効にするためには `router.refresh` を呼び出す必要があります。

:::message
現在 Next.js チームは Next.js におけるデータミューテーションに関する REC を作成しています。将来的には `router.refresh` は不要になるかもしれません。
:::

```diff tsx:app/articles/new/index.tsx
- import { useState } from "react";
+ import { useState, useTransition } from "react";

  // ...

+   const [isPending, startTransition] = useTransition();

    const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
      e.preventDefault();
      setLoading(true);
      await fetch("/api/articles", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ title, content }),
      });
      setLoading(false);
      router.push("/");
+     startTransition(() => {
+       router.refresh();
+     });
    };
```

`router.refresh` を呼び出しキャッシュを無効にしたことにより、トップページに遷移した際には新たにサーバーからデータを取得するようになりました。

![記事を投稿した後、記事の一覧画面に遷移すると投稿した記事が表示されている](https://storage.googleapis.com/zenn-user-upload/6874456aaef7-20230115.gif)

## まとめ

App Router の基本的な動作について確認してきました。Server Component がデフォルトとなっているのが大きな特徴で、よりパフォーマンスの高いアプリケーションを作成できるのは嬉しい変更点だと思えます。

また App Router ではページに関連するファイルを 1 つのディレクトリ内にまとめることができるのもポイントの 1 つです。従来のファイル構成と異なるアプローチも考えられるのではないでしょうか。

## 参考

- [Getting Started | Next.js](https://beta.nextjs.org/docs/getting-started)
- [React Server Componentsの仕組み：詳細ガイド | POSTD](https://postd.cc/how-react-server-components-work/)
