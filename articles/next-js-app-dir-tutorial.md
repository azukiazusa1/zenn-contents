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

記事の詳細ページを作成します。ここでは記事の本文を取得して表示するとともに、記事に対するコメントを表示する機能を実装します。特定の記事は `api/articles/{slug}` から、記事に対するコメントは `api/articles/{slug}/comments` から取得します。

それぞれのキャッシュ戦略も考えてみましょう。記事の本文は、更新頻度が高くないと想定できるので、一定期間キャッシュしておくことができます。キャッシュの生存期間は `fetch` のオプションに `next.revalidate` を指定することで設定できます。これは従来の ISR に匹敵する機能と言えるでしょう。

`revalidate` は `fetch` のオプションとして渡すだけでなく、ルート単位の設定もできます。ルート単位で設定するには `page` または `layout` ファイルで以下のように記述します。

```tsx
export const revalidate = 60
```

一方で、コメントは投稿した後に即座に反映されないと不自然ですので、キャッシュを利用しないほうが良いでしょう。

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

記事を取得する際に API が 404 を返した場合は `notFoutd` 関数を呼び出しています。この関数が呼ばれると同じディレクトリにある `not-found.tsx` が表示されます。

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

そこでコメント一覧の取得にストリーミングを使ってみましょう。ストリーミングを使うと、コメントの一覧の取得が完了するまで待つことなく、記事の本文を表示できます。

### コメントをストリーミングで取得する

ストリーミングはページの HTML を小さなチャンク部分解してクライアントに漸進的に送信します。これにより、すべてのデータの取得が完了するまで待つことなく、ページの一部分から表示を開始できます。

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

それでは動作を確認してみましょう。1000 ミリ秒経過した後に記事の本文が表示され、その間コメント一覧には `Loading comments` と表示されています。その後さらに 2000 ミリ秒経過した後にコメントの一覧が表示されます。

![コメント一覧をストリーミングで取得している](https://storage.googleapis.com/zenn-user-upload/b091e1508dbb-20230114.gif)

### `<head>` タグ

記事の詳細ページでは SEO のためにも `<head>` タグに `<title>` タグを設定しておきたいものです。ルーツごとに `<head>` タグを設定するには、`head.tsx` という特殊なファイルを配置します。

`Head` コンポーネント内では以下のタグを使用できます。

- `<title>`
- `<meta>`
- `<link>`
- `<script>`

ヘッダコンポーネントはサーバーコンポーネントと同様に `async/await` を使って動的に値を取得して設定できます。

```tsx:app/pages/head.tsx
import { Article } from "../../types";

const getArticle = async (slug: string) => {
  const res = await fetch(`http://localhost:3000/api/articles/${slug}`, {
    next: { revalidate: 60 },
  });

  if (!res.ok) {
    throw new Error("Failed to fetch article");
  }

  const data = await res.json();
  return data as Article;
};

export default async function Head({ params }: { params: { slug: string } }) {
  const article = await getArticle(params.slug);
  return (
    <>
      <title>{article.title}</title>
      <meta name="description" content={article.content} />
    </>
  );
}
```

Head コンポーネントを使用する際に、上位のディレクトリの `head.tsx` ファイルの内容を自動的に引き継がないことに注意してください。

例えば `app/head.tsx` では以下のように記述されています。

```tsx:app/head.tsx
export default function Head() {
  return (
    <>
      <title>Create Next App</title>
      <meta content="width=device-width, initial-scale=1" name="viewport" />
      <meta name="description" content="Generated by create next app" />
      <link rel="icon" href="/favicon.ico" />
    </>
  )
}
```

ファビコンや viewport の設定は全てのページで共通なので、下位のディレクトリで設定しなくても引き継いでほしいのですが、一番近い箇所にある `head.tsx` ファイルですべて上書きされるようになっています。

そのため記事詳細ページでは `<title>` と `<meta name="description">` のみが存在することになってしまいます。

このような場合には共通のコンポーネント作成する方法が紹介されています。まずは `app/DefaultTags` ファイルを作成します。ここには全ページで共通の `<head>` を設定します。

```tsx:app/DefaultTags.tsx
export default function DefaultTags() {
  return (
    <>
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <link href="/favicon.ico" rel="shortcut icon" />
    </>
  );
}
```

そして、各ページの `head.tsx` では `DefaultTags` をインポートして `<head>` に追加します。

```tsx:app/pages/head.tsx
+ import DefaultTags from "../../DefaultTags";

  export default async function Head({ params }: { params: { slug: string } }) {
    const article = await getArticle(params.slug);
    return (
      <>
        <title>{article.title}</title>
        <meta name="description" content={article.content} />
+       <DefaultTags />
      </>
    );
  }
```

### スタイリング

## 記事の作成

## まとめ



## 参考

- [Getting Started | Next.js](https://beta.nextjs.org/docs/getting-started)
- [React Server Componentsの仕組み：詳細ガイド | POSTD](https://postd.cc/how-react-server-components-work/)