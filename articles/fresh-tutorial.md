---
title: "Deno の Web フレームワーク Fresh チュートリアル"
emoji: "🍋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fresh", "deno", "twind", "denodeploy", "supabase"]
published: true
---

:::message
Fresh は現在（2022/06/19）プロダクション環境で使用することは推奨されません
:::

Fresh は Deno 製の Web フレームワークです。事前のビルドを必要せず、エッジでレンダリングを提供するという特徴があります。また [Islands Architecture](https://jasonformat.com/islands-architecture/) を採用しており、デフォルトではクライアントに JavaScript が配信されることがありません。

https://fresh.deno.dev/

この記事では Fresh を使用して記事投稿サービスのチュートリアルを紹介します。

完成したサイトは以下のようになります。

https://azukiazusa1-fresh-blog.deno.dev/

ソースコードは以下のレポジトリから確認できます。

https://github.com/azukiazusa1/fresh-blog

## インストール

Fresh を始めるには Deno の v1.22.3 バージョン以降が必要です。Deno をまだインストールしたことがないのならば、[installation](https://deno.land/#installation) を参考に Deno をインストールしましょう。

```sh
# Shell (Mac, Linux)
$ curl -fsSL https://deno.land/install.sh | sh 
# PowerShell (Windows):
$ iwr https://deno.land/install.ps1 -useb | iex
```

すでに Deno をインストール済みなら下記コマンドで Deno を最新バージョンに更新します。

```sh
$ deno upgrade
```

バージョンを確認しましょう。

```sh
$ deno --version
deno 1.23.0 (release, x86_64-apple-darwin)
v8 10.4.132.5
typescript 4.7.2
```

Deno のバージョンが v1.22.3 以降であることを確認したら、下記コマンドで Fresh のプロジェクトを作成します。

```sh
$ deno run -A --no-check https://raw.githubusercontent.com/lucacasonato/fresh/main/init.ts my-app
```

プロジェクトの作成が完了したら、次のコマンドで開発サーバーを立ち上げてみましょう。

```sh
$ cd my-app
$ deno task start
```

http://localhost:8080 にアクセスしてみましょう。以下の画面が表示されているはずです！

![スクリーンショット 2022-06-18 9.06.20](//images.ctfassets.net/in6v9lxmm5c8/69ncN1fcCE7VF3pZi2q6ya/b121377ffd204fba9197ba434450b0b1/____________________________2022-06-18_9.06.20.png)

## ディレクトリ構成

コードを書き始める前にまずは生成されたディレクトリ構成を確認してみましょう。

### `dev.ts`

開発時のエントリーポイントです。`deno task start` コマンドでは `dev.ts` ファイルを実行します。

### `main.ts`

本番時のエントリーポイントです。このファイルは Deno Deploy でリンクします。

### `fresh.gen.ts`

ルーティングと islands の情報を含むマニフェストファイルです。このファイルは `routes/` ディレクトリと `islands/` ディレクトリをもとに開発中は自動で生成されます。

### `import_map.json`

[Import maps](https://deno.land/manual/linking_to_external_code/import_maps) は Deno がサポートしている機能です。この機能は依存関係の管理を容易に行うために使用されます。

Deno は Node.js における npm のようなパッケージマネージャーを持っていないので、以下のように URL 指定でインポートして実行時にパッケージをインストールします。

```ts
import chunk from "https://deno.land/x/lodash@4.17.15-es/chunk.js";

console.log(chunk(["a", "b", "c", "d"], 2));
```

この方法は書き捨てのスクリプトならば問題ないのですが、大きなプロジェクトになった際に問題が発生します。例えば `lodash` が複数のファイルにおいて import されている場合、バージョンアップのたびにすべてのファイルを修正する必要があります。この問題を解決するために慣例的に `deps.ts` という名前のファイルを作成して依存関係を 1 つにまとめるという方法が使われていました。

https://deno.land/manual/examples/manage_dependencies

Fresh では前述のとおり Import maps を使用して依存関係を管理します。Import maps は JSON 形式で記述しキー名が import に使用する名前、値がその import したときに解決されるパスを指定しています。

```json
{
  "imports": {
    "$fresh/": "https://raw.githubusercontent.com/lucacasonato/fresh/main/",
    "lodash/": "https://deno.land/x/lodash@4.17.15-es/"
  }
}
```

lodash を使用するファイルでは `from "lodash/..."` の形式で import できます。

```ts
import chunk from "lodash/chunk.js";

console.log(chunk(["a", "b", "c", "d"], 2));
```

Import maps を使用する際には Deno の実行時に `--import-map=<FILE>` オプションを指定します。

```sh
$ deno run test.ts --import-map=import_map.json
```

### `deno.json`

[deno.json](https://deno.land/manual/getting_started/configuration_file) は Deno の設定ファイルです。TypeScript コンパイラ、フォーマッタ、リンタをカスタマイズするために使われます。`deno.json` ファイルは必須の機能ではなく、あくまでオプションの立ち位置です。

自動生成された `deno.json` ファイルでは以下の 2 つの設定が記載されています。

- `tasks`：npm scripts と同じように、プロジェクトで使用するコマンドをまとめることができます。ここの記述したコマンドは `deno task` コマンドで使用できます。
- `importMap`：Import map の配置場所をコマンドオプションで指定する代わりに記述します。

```json
{
  "tasks": {
    "start": "deno run -A --watch=static/,routes/,islands/ --no-check dev.ts"
  },
  "importMap": "./import_map.json"
}
```

### `routes/`

Next.js のようなファイルベースのルーティングを提供します。例えば `routes/articles/create.tsx` に配置したファイルは `/articles/create` のパスに対応しています。

また`routes/` ディレクトリ中のコードはクライアントに配信されることはありません。クライアント上で動作させたいコードは `islands/` ディレクトリに配置します。

### `islands/`

`islands` はもう少し先で説明する Islands Architecture に由来するディレクトリです。
このフォルダの中のコードは、クライアントとサーバーの両方から実行できます。

### `statis/`

静的なファイルを配置するディレクトリです。このディレクトリに配置した静的ファイルは自動では配信されます。また `routes/` ディレクトリも高い優先順位で提供されます。

## トップページの作成

それでは早速記事投稿サイトの作成に取り掛かりましょう。`routes/index.tsx` ファイルを修正します。

```tsx:routes/index.tsx
/** @jsx h */
import { h, PageProps } from "$fresh/runtime.ts"; // ①
import { Head } from "$fresh/src/runtime/head.ts";
import { Handlers } from "$fresh/server.ts";

interface Article {
  id: string;
  title: string;
  created_at: string;
}

export const handler: Handlers<Article[]> = { // ②
  async GET(_, ctx) {
    const articles: Article[] = [
      {
        id: "1",
        title: "Article 1",
        created_at: "2022-06-17T00:00:00.000Z",
      },
      {
        id: "2",
        title: "Article 2",
        created_at: "2022-06-10T00:00:00.000Z",
      },
    ];
    return ctx.render(articles);
  },
};

export default function Home({ data }: PageProps<Article[]>) { // ③
  return (
    <div>
      <Head>
        <title>Fresh Blog</title>
      </Head>
      <div>
        <h1>Fresh Blog</h1>
        <section>
          <h2>Posts</h2>
          <ul>
            {data.map((article) => (
              <li key={article.id}>
                <a href={`articles/${article.id}`}>
                  <h3>{article.title}</h3>
                  <time dateTime={article.created_at}>{article.created_at}</time>
                </a>
              </li>
            ))}
          </ul>
        </section>
      </div>
    </div>
  );
}

```

①：最初の 1 行では `/** @jsx h */` というコメントと `$fresh/runtime.ts` から `h` 関数を import しています。これらは JSX をレンダリングするためのボイラープレートです。React における `import React from 'react'` と似たようなものだと考えてよいでしょう。

②：ルートモジュールではカスタムハンドラを作成できます。カスタムハンドラは `handler` という名前で名前付きエクスポートを行う必要があります。

カスタムハンドラは第 1 引数に `Request` オブジェクトを受け取り `Response` オブジェクトを返す関数です。この例では `GET` ハンドラを定義し `ctx.render` 関数に `articles` 配列を渡しています（今のところはベタ書きです）。`ctx.render` 関数に渡したデータはページコンポーネントの `props.data` からアクセスできます。

③：JSX コンポーネントを作成します。ルートモジュールではコンポーネントを default export することで HTML のレンダリングを行います。なお Fresh では React ではなく [Preact](https://preactjs.com/) を使用しています。

コンポーネント内部ではカスタムハンドラから受け取った `articles` 配列をリストレンダリングしています。また `$fresh/src/runtime/head.ts` から import した `<Head>` タグを使用することでページの `head` に要素を追加できます。

ここまでの内容を確認してみましょう。 http://localhost:8000/ にアクセスします。

![スクリーンショット 2022-06-18 11.40.47](//images.ctfassets.net/in6v9lxmm5c8/ku0YZhVTl9Lh2DXc5Hd2h/f446906f2cea95e33df644a55e65ec57/____________________________2022-06-18_11.40.47.png)

想定通り記事の一覧が表示されていますね！

### スタイリング

ただこのままの表示ですとちょっと味気ないですので、スタイルを追加しましょう。`import_map.json` に以下を追加します。

```json
{
  "imports": {
    "@twind": "./util/twind.ts",

    "$fresh/": "https://raw.githubusercontent.com/lucacasonato/fresh/main/",
    "twind/": "https://esm.sh/twind@0.16.16/",
    "dayjs/": "https://esm.sh/dayjs@1.11.3/"
  }
}

```

[twind](https://github.com/tw-in-js/twind) は Tailwind ライクな CSS フレームワークです。Tailwind CSS と異なりビルドステップが不要で Deno でも使用できます。

[dayjs](https://deno.land/x/dayjs@v1.11.3) は npm でもおなじみの日付操作ライブラリです。

まずは `util/twind.ts` ファイルを作成します。このファイルでは twind の設定を行うとともに、必要な関数を再 export します。

```ts:utis/twind.ts
import { IS_BROWSER } from "$fresh/runtime.ts";
import { apply, Configuration, setup, tw } from "twind";
export { css } from "twind/css";

export { apply, setup, tw };
export const config: Configuration = {};

if (IS_BROWSER) setup(config);
```

続いて `routes/_render.ts` ファイルを作成します。

```ts:routes/_render.ts
import { config, setup } from "@twind";
import { RenderContext, RenderFn } from "$fresh/server.ts";
import { virtualSheet } from "twind/sheets";

const sheet = virtualSheet();
sheet.reset();
setup({ sheet, ...config });

export function render(ctx: RenderContext, render: RenderFn) {
  const snapshot = ctx.state.get("twindSnapshot") as unknown[] | null;
  sheet.reset(snapshot || undefined);
  render();
  ctx.styles.splice(0, ctx.styles.length, ...(sheet).target);
  const newSnapshot = sheet.reset();
  ctx.state.set("twindSnapshot", newSnapshot);
}
```

これで twind の準備は整いました。`routes/index.tsx` を修正して確認してみましょう。`@twind` から `tw` 関数を import して使用します。

```tsx diff:routes/index.tsx
  /** @jsx h */
  import { h, PageProps } from "$fresh/runtime.ts";
  import { Head } from "$fresh/src/runtime/head.ts";
  import { Handlers } from "$fresh/server.ts";
+ import { tw } from "@twind";

  // 省略...
  
  export default function Home({ data }: PageProps<Article[]>) {
    return (
      <div>
        <Head>
          <title>Fresh Blog</title>
        </Head>
        <div>
-       <h1>Fresh Blog</h1>
+       <h1 class={tw("text-red-500")}>Fresh Blog</h1>
          <section>
            <h2>Posts</h2>
            <ul>
              {data.map((article) => (
                <li key={article.id}>
                  <a href={`articles/${article.id}`}>
                    <h3>{article.title}</h3>
                    <time dateTime={article.created_at}>{article.created_at}</time>
                  </a>
                </li>
              ))}
            </ul>
          </section>
        </div>
      </div>
    );
  }
```

指定したスタイル（`tw("text-red-500")`）が適用されていることがわかります！

![スクリーンショット 2022-06-18 13.48.32](//images.ctfassets.net/in6v9lxmm5c8/13dmzcIMStEm0muU7vEbwC/33bdf282990dfdded1d758b882a15bdc/____________________________2022-06-18_13.48.32.png)

Twind が使えるようになったので本格的にスタイリングを始めましょう。ここでは Twind の解説は省きますが、最終的に次のようなコードになります。

```tsx:routes/index.tsx
/** @jsx h */
import { h, PageProps } from "$fresh/runtime.ts";
import { Head } from "$fresh/src/runtime/head.ts";
import { Handlers } from "$fresh/server.ts";
import { tw } from "@twind";

// dayjs で相対日付を表示するために import
import dayjs from "https://esm.sh/dayjs@1.11.3";
import relativeTime from "dayjs/plugin/relativeTime";
import "dayjs/locale/ja";
dayjs.extend(relativeTime);
dayjs.locale("ja");

interface Article {
  id: string;
  title: string;
  created_at: string;
}

export const handler: Handlers<Article[]> = {
  async GET(_, ctx) {
    const articles: Article[] = [
      {
        id: "1",
        title: "Article 1",
        created_at: "2022-06-17T00:00:00.000Z",
      },
      {
        id: "2",
        title: "Article 2",
        created_at: "2022-06-10T00:00:00.000Z",
      },
    ];
    return ctx.render(articles);
  },
};

export default function Home({ data }: PageProps<Article[]>) {
  return (
    <div class={tw("h-screen bg-gray-200")}>
      <Head>
        <title>Fresh Blog</title>
      </Head>
      <div
        class={tw(
          "max-w-screen-sm mx-auto px-4 sm:px-6 md:px-8 pt-12 pb-20 flex flex-col"
        )}
      >
        <h1 class={tw("font-extrabold text-5xl text-gray-800")}>Fresh Blog</h1>
        <section class={tw("mt-8")}>
          <h2 class={tw("text-4xl font-bold text-gray-800 py-4")}>Posts</h2>
          <ul>
            {data.map((article) => (
              <li
                class={tw("bg-white p-6 rounded-lg shadow-lg mb-4")}
                key={article.id}
              >
                <a href={`articles/${article.id}`}>
                  <h3
                    class={tw(
                      "text-2xl font-bold mb-2 text-gray-800 hover:text-gray-600 hover:text-underline"
                    )}
                  >
                    {article.title}
                  </h3>
                </a>
                <time
                  class={tw("text-gray-500 text-sm")}
                  dateTime={article.created_at}
                >
                  {dayjs(article.created_at).fromNow()}
                </time>
              </li>
            ))}
          </ul>
        </section>
      </div>
    </div>
  );
}
```

![スクリーンショット 2022-06-18 14.55.15](//images.ctfassets.net/in6v9lxmm5c8/2HeYhs5c9RK7Ylbn1kPDHP/6028da8f4334b8858ae74ede9b98c514/____________________________2022-06-18_14.55.15.png)

## データベースを使用する

続いてベタ書きしていた記事一覧を Postgress データベースから取得するように修正しましょう。ここでは無料枠で 500MB までの Postgres サーバーをすぐに使うことができる Supabase を使うこととします。

https://supabase.com/

### Supbase でプロジェクトを作成

まずは Start your projest をクリックします。

![スクリーンショット 2022-06-18 14.59.31](//images.ctfassets.net/in6v9lxmm5c8/29aDFycTYF680APcUtaGD9/cc87dac9c255b1a0ad87c1b1604ba7c6/____________________________2022-06-18_14.59.31.png)

GitHub で認可を求められるので許可する必要があります。

![スクリーンショット 2022-06-18 15.00.59](//images.ctfassets.net/in6v9lxmm5c8/jDkIwTT6N2sAh383vJBpi/38012e66fcbdf78677e19c906d0516cb/____________________________2022-06-18_15.00.59.png)

New Project から新しいプロジェクトを作成しましょう。

![スクリーンショット 2022-06-18 15.03.40](//images.ctfassets.net/in6v9lxmm5c8/6a6CmHRzhILlZgYI3Vff1e/a60692e9ed95aa966785bbf74ef6f4a1/____________________________2022-06-18_15.03.40.png)

プロジェクト名とデータベースのパスワードを入力し、リージョンを選択します。

![スクリーンショット 2022-06-18 15.05.53](//images.ctfassets.net/in6v9lxmm5c8/59jOorF3SpGQ6vwcxoV8Tj/16d4acb1670891ccbbaba179891401d8/____________________________2022-06-18_15.05.53.png)

プロジェクトの作成の完了後、サイドバーの Setting → Database → Connection info からホスト名をコピーして保存しておきます。

![スクリーンショット 2022-06-18 17.03.45](//images.ctfassets.net/in6v9lxmm5c8/7aCeJGLGFmGet9PLEV24ll/97ad9c7ea3edd7e78c2072b9ccfd9020/____________________________2022-06-18_17.03.45.png)

サイドバーの Table Edier メニューを選択し、Create Table からテーブルを作成します。

![スクリーンショット 2022-06-18 15.12.18](//images.ctfassets.net/in6v9lxmm5c8/5d98LI3aWqVcwbZnVewCb2/00f320263412851d93f26323a1b254f6/____________________________2022-06-18_15.12.18.png)

次のようにテーブルを作成しました。

![スクリーンショット 2022-06-18 15.16.50](//images.ctfassets.net/in6v9lxmm5c8/vBhvFLEb79CzzrhmiuCLG/427ae8525daca3d93bc7be0b3ab7433d/____________________________2022-06-18_15.16.50.png)


|  Name   | Type     | Default Value     | Primary     |
| ---------- | ---------- | ---------- | ---------- |
| id       | uuid       | uuid_generate_v4       | ◯       |
| created_at       | timestamptz       | now()       |        |
| title       | text       |        |        |
| content       | text       |        |        |

いくつかテストデータを挿入しておきましょう。Insert row からデータを作成します。

![スクリーンショット 2022-06-18 15.26.33](//images.ctfassets.net/in6v9lxmm5c8/6T2GmNAZQenNDe8orpWDIT/09fa4aeaa0ad6dc425661bc09e0d0331/____________________________2022-06-18_15.26.33.png)

![スクリーンショット 2022-06-18 15.26.14](//images.ctfassets.net/in6v9lxmm5c8/2b1W7zeN2DOg5WaJR3I5s4/aeccee412a08214eac8ee07ced7dfea3/____________________________2022-06-18_15.26.14.png)

### Deno から Postgress に接続する

それでは、作成したデータベースに対して接続します。ここでは [deno-postgres](https://deno.land/x/postgres@v0.16.1) ライブラリを使用します。また環境変数を使用するために [dotenv](https://deno.land/x/dotenv@v3.2.0) ライブラリも追加します。

`import_map.json` に記述しましょう。

```json:import_map.json
{
  "imports": {
    "@db": "./util/db.ts",

    "postgress": "https://deno.land/x/postgres@v0.16.1/mod.ts",    
    "dotenv/": "https://deno.land/x/dotenv@v3.2.0/"
  }
}
```

`.env` ファイルを作成してさきほどの手順で保存しておいたパスワードとホスト名を記述します。

```txt:.env
DB_USER=postgres
POSTGRES_DB=postgres
DB_PORT=${ポート}
DB_PASSWORD=${パスワード}
DB_HOST=${ホスト名}
```

`util/db.ts` ファイルを作成して Postgres へのコネクションを作成します。

```ts:util/db,ts
import { Client } from "postgress";
import "dotenv/load.ts";

/**
 * articls テーブルの型
 */
export interface Article {
  id: string;
  title: string;
  content: string;
  created_at: string;
}

// DB クライアントを作成する
const client = new Client({
  user: Deno.env.get("DB_USER"),
  database: Deno.env.get("POSTGRES_DB"),
  hostname: Deno.env.get('DB_HOST'),
  password: Deno.env.get('DB_PASSWORD'),
  port: Deno.env.get('DB_PORT'),
});

// データベースに接続
await client.connect();

/**
 * すべての記事を取得する
 */
export const findAllArticles = async () => {
  try {
    const result = await client.queryObject<Article>(
      "SELECT * FROM articles ORDER BY created_at DESC"
    );
    return result.rows;
  } catch (e) {
    console.error(e);
    return [];
  }
}

/**
 * ID を指定して記事を取得する
 */
export const findArticleById = async (id: string) => {
  try {
    const result = await client.queryObject<Article>(
      "SELECT * FROM articles WHERE id = $1",
      [id]
    );
    if (result.rowCount === 0) {
      return null
    }
    return result.rows[0];
  } catch (e) {
    console.error(e);
    return null;
  }
}

/**
 * 記事を新規作成する
 */
export const createArticle = async (article: Pick<Article, 'title' | 'content'>) => {
  try {
    const result = await client.queryObject<Article>(
      "INSERT INTO articles (title, content) VALUES ($1, $2) RETURNING *",
      [article.title, article.content]
    );
    return result.rows[0];
  } catch (e) {
    console.error(e);
    return null;
  }
}
```

`dotenv/load.ts` をインポートすることで自動的に `.env` ファイルを読み込みます。Deno では `Deno.env` から環境変数を読み込みます。

`deno-postgres` から `Client` クラスをインポートし、Postgres への接続情報をコンストラクタに渡します。そして作成した `client` を利用し `await client.connect()` でデータベースに接続します。

このモジュールでは以下の関数を export します。

- `findAllArticles`：すべての記事を取得する
- `findArticleById`：ID を指定して記事を取得する
- `createArticle`：記事を新規作成する

データベースへの問い合わせは `connect.queryObject` 関数で行います。型引数で `Article` を指定することでクエリ結果に `Article` 型が付与されます。

`routes/index.tsx` ファイルに戻り、カスタムハンドラをデータベースから記事を取得するように修正しましょう。

```tsx:routes/index.tsx
import { Article, findAllArticles } from "@db";

export const handler: Handlers<Article[]> = {
  async GET(_, ctx) {
    const articles = await findAllArticles();
    return ctx.render(articles);
  },
};
```

ここまでうまくいけば、データベースに保存してある記事が表示されるはずです！

![スクリーンショット 2022-06-18 18.00.55](//images.ctfassets.net/in6v9lxmm5c8/7MK1YjAnBBv2HneRpuUpCW/98858fd95263e3eed56c7cb8e0ea68a5/____________________________2022-06-18_18.00.55.png)

## 記事詳細ページ

続いて、記事の詳細を取得するページを作成しましょう。Fresh は Next.js のようなファイルベースのルーティングを採用しています。

`routes/articles/[id].tsx` という名前のファイルを作成すれば動的なルーティングを作成できます。早速試してみましょう。

```tsx:routes/articles/[id].tsx
/** @jsx h */
import { h, PageProps } from "$fresh/runtime.ts";

export default function ArticlePage(props: PageProps) {
  const { id } = props.params;
  return (
    <div>
      <h1>Article: {id}</h1>
    </div>
  );
}
```

動的に設定したパスパラメータは `props.params` から取得できます。

トップページからいずれかの記事を選択してクリックすると記事詳細ページへ遷移するはずです。

![スクリーンショット 2022-06-18 18.10.21](//images.ctfassets.net/in6v9lxmm5c8/2p6Ffvzt3CEkbLBynfFvkJ/62309cc6daa060b622d6941919ae933d/____________________________2022-06-18_18.10.21.png)

確かに、記事詳細ページが作成されていますね。ここで `fresh.get.ts` ファイルを確認してみると自動でルーティング情報が追加されていることが確認できます。

データベースから記事を取得するように修正しましょう。カスタムハンドラ内でパスパラメータを取得し、`findArticleById` 関数を呼び出します。記事を取得できない場合には `Not Found` を表示します。

```tsx:routes/articles/[id].tsx
/** @jsx h */
import { h, PageProps } from "$fresh/runtime.ts";
import { Head } from "$fresh/src/runtime/head.ts";
import { Handlers } from "$fresh/server.ts";
import { tw } from "@twind";
import { Article, findArticleById } from "@db";
import dayjs from "dayjs";

export const handler: Handlers<Article | null> = {
  async GET(_, ctx) {
    // パスパラメータを取得
    const { id } = ctx.params;
    // パスパラメータの ID を引数に記事を取得
    const article = await findArticleById(id);

    // 記事が取得できなかった場合は null を渡す
    if (!article) {
      return ctx.render(null);
    }

    // 記事が取得できた場合は取得した記事を渡す
    return ctx.render(article);
  },
};

export default function ArticlePage({ data }: PageProps<Article | null>) {
  // Props.data に null が渡された時には `Not Found` を表示する
  if (!data) {
    return <div>Not Found</div>;
  }

  return (
    <div class={tw("min-h-screen bg-gray-200")}>
      <Head>
        <title>{data.title}</title>
      </Head>
      <div
        class={tw(
          "max-w-screen-sm mx-auto px-4 sm:px-6 md:px-8 pt-12 pb-20 flex flex-col"
        )}
      >
        <article class={tw("rounded-xl border p-5 shadow-md bg-white")}>
          <header>
            <h1 class={tw("font-extrabold text-5xl text-gray-800")}>
              {data.title}
            </h1>
            <time class={tw("text-gray-500 text-sm")} dateTime={data.createdAt}>
              {dayjs(data.created_at).format("YYYY-MM-DD HH:mm:ss")}
            </time>
          </header>
          <section class={tw("mt-6")}>
            <p>{data.content}</p>
          </section>
        </article>
      </div>
    </div>
  );
}
```

記事を取得して表示できました！

![スクリーンショット 2022-06-18 19.16.17](//images.ctfassets.net/in6v9lxmm5c8/7xT0hV0weSRgNFcc3gn6TY/0a6f688df8ed5837f633505a47d7a23f/____________________________2022-06-18_19.16.17.png)

記事の内容はマークダウンで記述されているので HTML に変換して表示できるようにしましょう。`import_map.json` に [marked](https://github.com/markedjs/marked) と [sanitize-html](https://github.com/apostrophecms/sanitize-html) を追加します。

```json:import_map.json
{
  "imports": {
    "marked": "https://esm.sh/marked@4.0.17",
    "sanitize-html": "https://esm.sh/sanitize-html@2.7.0",
  }
}
```

マークダウン用のスタイルシートを `static/article.css` に記述します。

```css:static/article.css
#contents {
  font-size: 16px;
  color: #1f2937;
}

#contents h1 {
  font-size: 3.0rem;
  font-weight: 700;
  border-bottom: 1px solid  #ddd;
  padding-bottom: 0.1rem;
  margin-bottom: 1.1rem;

}

#contents h2 {
  font-size: 2.25em;
  font-weight: 700;
  border-bottom: 1px solid #ddd;
  padding-bottom: 0.1rem;
  margin-bottom: 1.1rem;
}

#contents h3 {
  font-size: 1.875rem;
  font-weight: 700;
  padding-bottom: 0.1rem;
  margin-bottom: 1.1rem;
}

#contents h4 {
  font-size: 1.5rem;
  padding-bottom: 0.1rem;
  margin-bottom: 1.1rem;
}

#contents h5 {
  font-size: 1.25rem;
}

#contents h6 {
  font-size: 1.125rem;
}

#contents ul {
  list-style-type: disc;
}

#contents ol {
  list-style-type: decimal;
}

#contents ul,
#contents ol {
  padding-left: 1.5em;
  margin: 1.5em 0;
  line-height: 1.9;
}

#contents blockquote {
  border-left: 5px solid #ddd;
  font-weight: 100;
  padding: 1rem;
  padding-right: 0;
  margin: 1.5rem 0;
}

#contents p {
  margin: 0 0 1.125rem 0;
  line-height: 1.9;
}

#contents img {
  margin: 1.5rem auto;
}

#contents a {
  color: #0f83fd;
}

#contents a:hover {
  text-decoration: underline;
}

#contents table {
  margin: 1.2rem auto;
  width: auto;
  border-collapse: collapse;
  font-size: .95em;
  line-height: 1.5;
  word-break: normal;
  display: block;
  overflow: auto;
  -webkit-overflow-scrolling: touch;
}

#contents th {
  font-weight: 700;
  color: black;
  padding: .5rem;
  border: 1px solid #cfdce6;
  background: #edf2f7;
}

#contents td {
  padding: .5rem;
  border: 1px solid #cfdce6;
}

#contents code:not([class*="language-"]) {
  background-color: #eee;
  color: #333;
  padding: 0.1em 0.4em;
}
```

マークダウンを変換して表示するように `routes/articles/[id].tsx` を修正しましょう。カスタムハンドラ内で `article` を取得した後、`marked` でマークダウンをパースし、`DOMPurify.sanitize` でサニタイズ処理を行います。

```tsx:routes/articles/[id].tsx
import { marked } from "marked";
import sanitize from "sanitize-html";

interface Data {
  /** DB から取得した記事 */
  article: Article;
  /** パース済みのコンテンツ */
  parsedContent: string;
}

export const handler: Handlers<Data | null> = {
  async GET(_, ctx) {
    const { id } = ctx.params;
    const article = await findArticleById(id);

    if (!article) {
      return ctx.render(null);
    }

    // マークダウンをパースする
    const parsed = parked(article.content);
    // HTML をサニタイズする
    const parsedContent = sanitize(parsed);;

    return ctx.render({
      article,
      parsedContent,
    });
  },
};
```

コンポーネント内では `<Head>` タグ内でさきほど作成した `article.css` ファイルを読み込みます。パース済みの内容は `dangerouslySetInnerHTML` で表示します。

```tsx:routes/articles/[id].tsx
export default function ArticlePage({ data }: PageProps<Data | null>) {
  if (!data) {
    return <div>Not Found</div>;
  }

  const { article, parsedContent } = data;

  return (
    <div class={tw("min-h-screen bg-gray-200")}>
      <Head>
        <title>{article.title}</title>
        <link rel="stylesheet" href="/article.css" />
      </Head>
      <div
        class={tw(
          "max-w-screen-sm mx-auto px-4 sm:px-6 md:px-8 pt-12 pb-20 flex flex-col"
        )}
      >
        <article class={tw("rounded-xl border p-5 shadow-md bg-white")}>
          <header>
            <h1 class={tw("font-extrabold text-5xl text-gray-800")}>
              {article.title}
            </h1>
            <time
              class={tw("text-gray-500 text-sm")}
              dateTime={article.created_at}
            >
              {dayjs(article.created_at).format("YYYY-MM-DD HH:mm:ss")}
            </time>
          </header>
          <section class={tw("mt-6")}>
            <div
              id="contents"
              dangerouslySetInnerHTML={{ __html: parsedContent }}
            />
          </section>
        </article>
      </div>
    </div>
  );
}
```

これにより、記事の内容が HTML に変換して表示されるようになりました。

![スクリーンショット 2022-06-18 20.26.34](//images.ctfassets.net/in6v9lxmm5c8/6Qgx6EqngSjUuNpxQ7KdTe/95b92af86405eb2ec745543704478b17/____________________________2022-06-18_20.26.34.png)

## 記事作成ページ

最後に、記事を新しく作成できるように実装しましょう。`routes/articles/create.tsx` ファイルを作成します。

```tsx:routes/articles/create.tsx
/** @jsx h */
import { h } from "$fresh/runtime.ts";

export default function CreateArticlePage() {
  return (
    <h1>Create Post</h1>
  );
}
```

トップページに記事作成ページへのリンクを作成しましょう。`routes/index.tsx` ファイルを修正します。

```tsx:routes/index.tsx
<section class={tw("mt-8")}>
    <div class={tw("flex justify-between items-center")}>
      <h2 class={tw("text-4xl font-bold text-gray-800 py-4")}>Posts</h2>
      <a
        href="/articles/create"
        class={tw(
          "bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md"
        )}
      >
        Create Post
      </a>
    </div>
```
![スクリーンショット 2022-06-18 20.50.01](//images.ctfassets.net/in6v9lxmm5c8/2dOmHOuDI4TjGLHwqFYlIZ/6a0cd533540b29e89ae8b906fc885f99/____________________________2022-06-18_20.50.01.png)

続いて `ruotes/articles/create.tsx` でコンテンツを入力できるようにフォームを追加しましょう。Fresh は デフォルトでクライアントの JavaScript は一切使用せずネイティブの `<form>` 要素を利用します。つまり入力中の状態を保持したり、フォームのサブミット後に `e.preventDefault()` を呼び出す必要はありません。

```tsx:routes/articles/create.tsx
/** @jsx h */
import { h } from "$fresh/runtime.ts";
import { Head } from "$fresh/src/runtime/head.ts";
import { tw } from "@twind";

export default function CreateArticlePage() {
  return (
    <div class={tw("min-h-screen bg-gray-200")}>
      <Head>
        <title>Create Post</title>
      </Head>
      <div
        class={tw(
          "max-w-screen-sm mx-auto px-4 sm:px-6 md:px-8 pt-12 pb-20 flex flex-col"
        )}
      >
        <h1 class={tw("font-extrabold text-5xl text-gray-800")}>Create Post</h1>

        <form
          class={tw("rounded-xl border p-5 shadow-md bg-white mt-8")}
          method="POST"
        >
          <div class={tw("flex flex-col gap-y-2")}>
            <div>
              <label class={tw("text-gray-500 text-sm")} htmlFor="title">
                Title
              </label>
              <input
                id="title"
                class={tw("w-full p-2 border border-gray-300 rounded-md")}
                type="text"
                name="title"
              />
            </div>
            <div>
              <label class={tw("text-gray-500 text-sm")} htmlFor="content">
                Content
              </label>
              <textarea
                id="content"
                rows={10}
                class={tw("w-full p-2 border border-gray-300 rounded-md")}
                name="content"
              />
            </div>
          </div>
          <div class={tw("flex justify-end mt-4")}>
            <button
              class={tw(
                "bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md"
              )}
              type="submit"
            >
              Create
            </button>
          </div>
        </form>
      </div>
    </div>
  );
}
```

見た目は何も変哲のないフォームです。

![スクリーンショット 2022-06-18 21.16.37](//images.ctfassets.net/in6v9lxmm5c8/4FPZ4bjo0ZGojHjQ9XjuXh/e49532020617065130d637b271708513/____________________________2022-06-18_21.16.37.png)

まだサブミットした後の処理を実装していないので Create ボタンをクリックしても何も起こりません。カスタムハンドラを定義しましょう。

```tsx:routes/articles/create.tsx
import { createArticle } from "@db";
import { Handlers } from "$fresh/server.ts";

interface Data {
  /** バリデーションエラー情報 */
  error: {
    title: string;
    content: string;
  };
  /** 前回のタイトルの入力値 */
  title?: string;
  /** 前回のコンテンツの入力値 */
  content?: string;
}

export const handler: Handlers<Data> = {
  async POST(req, ctx) {
    // フォームデータの入力値を取得
    const formData = await req.formData();
    const title = formData.get("title")?.toString();
    const content = formData.get("content")?.toString();

    // タイトルまたはコンテンツどちらも未入力の場合はバリデーションエラー
    if (!title || !content) {
      return ctx.render({
        error: {
          title: title ? "" : "Title is required",
          content: content ? "" : "Content is required",
        },
        title,
        content,
      });
    }

    const article = {
      title,
      content,
    };

    // データベースに保存
    await createArticle(article);

    // トップページにリダイレクト
    return new Response("", {
      status: 303,
      headers: {
        Location: "/",
      },
    });
  },
};
```

カスタムハンドラに `POST` メソッドを定義します。これにより POST リクエストが送信されたときにハンドラが呼び出されます。

ハンドラ内ではまず `req.formData()` 関数を呼び出してフォームの入力値を取り出します。`title` または `content` のどちらかが未入力であった場合はバリデーションエラーとして処理を中断します。ここで `ctx.render` 関数を呼び出す際にどの項目にエラーがあったのかの情報と、前回の入力値を引数で渡すようにしておきます。

`title` と `content` どちらも入力されている場合には `createArticle` 関数を呼び出してデータベースに保存します。その後 `Response` を生成して返却しトップページへリダイレクトさせます。カスタムハンドラは `ctx.render()` だけでなく `Response` 型であれば返り値にできます。

バリデーションエラー発生時に値を返すことにしたので、コンポーネント側も値を受け取るように修正しましょう。

```tsx:routes/articles/create.tsx
export default function CreateArticlePage({
  data,
}: PageProps<Data | undefined>) {
  return (
    // 省略...
          <div class={tw("flex flex-col gap-y-2")}>
            <div>
              <label class={tw("text-gray-500 text-sm")} htmlFor="title">
                Title
              </label>
              <input
                id="title"
                class={tw("w-full p-2 border border-gray-300 rounded-md")}
                type="text"
                name="title"
                value={data?.title} // 前回の入力値を初期値に渡す
              />
              // タイトルの入力にバリデーションエラーがあった場合表示する
              {data?.error?.title && (
                <p class={tw("text-red-500 text-sm")}>{data.error.title}</p>
              )}
            </div>
            <div>
              <label class={tw("text-gray-500 text-sm")} htmlFor="content">
                Content
              </label>
              <textarea
                id="content"
                rows={10}
                class={tw("w-full p-2 border border-gray-300 rounded-md")}
                name="content"
                value={data?.content} // 前回の入力値を初期値に渡す
              />
               // コンテンツの入力にバリデーションエラーがあった場合表示する
              {data?.error?.content && (
                <p class={tw("text-red-500 text-sm")}>{data.error.content}</p>
              )}
            </div>
          </div>
  );
}
```

`data.title` の値をタイトルの input の初期値として `value` に渡します。また`data.error.title` の値が存在する場合にはエラーメッセージを表示します。コンテンツ入力欄も同様に修正します。

それでは実際に試してみましょう。タイトルとコンテンツどちらも未入力の場合、フォームをサブミットした後エラーメッセージが表示されます。

![スクリーンショット 2022-06-19 9.32.05](//images.ctfassets.net/in6v9lxmm5c8/6JFVNOMUF9a6jqm3cAEs0L/6a48bf491a0d10cce753ce0d7f2b8d4d/____________________________2022-06-19_9.32.05.png)

フォームに正しく入力できている場合、サブミット後トップページへリダイレクトします。トップページの先頭に新しく記事が追加されていることが確認できるでしょう。

![スクリーンショット 2022-06-19 9.36.48](//images.ctfassets.net/in6v9lxmm5c8/2KlvnSFlEpgdwoFRHIyOzp/4c0eb4267b5212bb6a738baa09a7b7a5/____________________________2022-06-19_9.36.48.png)

### プレビューを表示する
ここまではクライアントの JavaScript を一切使用せずに実装してきました（試しにブラウザの設定から JavaScript を無効にして動かしてみてください！）Fresh は **Zero runtime overhead** を謳っているとおり、デフォルトではクライアントに JavaScript が送信されません。

このことはパフォーマンス上メリットが得られますが、とはいえインタラクティブな操作を提供することの制限にもなりかねません。例えば、記事の内容を入力している最終にプレビューが表示できればより便利なアプリケーションとなるでしょうが、この機能を実装するためにはクライアントに JavaScript を動作させる必要があります。

現在の多くの Web フレームワークでは、クライアントに JavaScript を提供しないか、ページ全体のレンダラーを提供するかを選択できます。

しかし、この選択はページの一部分でのみインタラクティブ性を持たせたい場合を考えると、あまり柔軟ではありません。Fresh では静的なページの中の一部分で JavaScript を与える [Islands Architecture](https://jasonformat.com/islands-architecture/) を採用しています。大半の静的なコンテンツはサーバーでレンダリングし、動的な部分のみをプレースホルダーに挿入するというシンプルな考え方です。

![dave-hoefler-NYVc84Gh78I-unsplash](//images.ctfassets.net/in6v9lxmm5c8/63gGethlH2ACROO0DQzpX5/1d249eb9f1954b0d04aefe1aa9615cb6/dave-hoefler-NYVc84Gh78I-unsplash.jpg)

その他 Islands Architecture を採用している Web フレームワークに [Astro](https://astro.build/) が存在します。

Fresh では Islands Architecture を実現するために `islands/` ディレクトリを使用しています。このフォルダ内のモジュールは、それぞれ 1 つの Islands コンポーネントをカプセル化します。またこのモジュールのファイル名はパスカルケースとする必要があります。

それでは実際に Islands コンポーネントを作成しましょう。`islands/ContentForm.tsx` ファイルを作成します。

```tsx:islands/ContentForm.tsx
/** @jsx h */
import { h, useState } from "$fresh/runtime.ts";
import { tw } from "@twind";
import { marked } from "marked";
import DOMPurify from "dompurify";

interface Props {
  initialValue?: string;
}

export default function ContentForm({ initialValue = "" }: Props) {
  // コンテンツの入力値を保持する
  const [value, setValue] = useState(initialValue);
  // プレビュー表示するかどうかの状態
  const [preview, setPreview] = useState(false);

  /**
   * マークダウンをパースする関数
   */
  const parse = (content: string) => {
    const parsed = marked(content);
    const purified = DOMPurify.sanitize(parsed);
    return purified;
  };

  const handleChange = (e: Event) => {
    const target = e.target as HTMLTextAreaElement;
    setValue(target.value);
  };
  return (
    <div>
      <div class={tw("flex justify-between")}>
        <label class={tw("text-gray-500 text-sm")} htmlFor="content">
          Content
        </label>
        <label class={tw("text-gray-500 text-sm")}>
          Preview
          <input
            type="checkbox"
            id="preview"
            class={tw("ml-2")}
            checked={preview}
            onChange={() => setPreview((prev) => !prev)}
          />
        </label>
      </div>
      // preveiw が true の場合パース済のコンテンツを表示
      // それ以外の場合にはコンテンツの入力フォームを表示
      {preview ? (
        <div
          id="contents"
          dangerouslySetInnerHTML={{
            __html: parse(value),
          }}
        />
      ) : (
        <textarea
          id="content"
          rows={10}
          class={tw("w-full p-2 border border-gray-300 rounded-md")}
          name="content"
          value={value}
          onChange={handleChange}
        />
      )}
    </div>
  );
}
```

大まかな部分はもともとのコンテンツ入力フォームと同じです。

はじめに Props として値の初期値を受け取ります。大きな違いはフォームの入力値を `useState` で保持するように変更したところです。状態を保持するように変更したことで、入力値が変更されるたびにプレビューの表示を更新させることができます。

`useState` で `preview` という状態も保持しており、この値はチェックボックスにより操作されます。`preview` が `true` の場合には入力値のプレビューを表示し、`false` の場合には入力フォームを表示するようにしました。

続いて `routes/articles/create.tsx` において `ContentForm` コンポーネントを利用するように修正します。またプレビューを表示させるので `article.css` を読み込む必要があります。

```diff tsx:routes/articles/create.tsx
  export default function CreateArticlePage({
    data,
  }: PageProps<Data | undefined>) {
    return (
      <div class={tw("min-h-screen bg-gray-200")}>
        <Head>
          <title>Create Post</title>
+        <link rel="stylesheet" href="/article.css" />
        </Head>
        <div
          class={tw(
            "max-w-screen-sm mx-auto px-4 sm:px-6 md:px-8 pt-12 pb-20 flex flex-col"
          )}
        >
          <h1 class={tw("font-extrabold text-5xl text-gray-800")}>Create Post</h1>
  
          <form
            class={tw("rounded-xl border p-5 shadow-md bg-white mt-8")}
            method="POST"
          >
            <div class={tw("flex flex-col gap-y-2")}>
              <div>
                <label class={tw("text-gray-500 text-sm")} htmlFor="title">
                  Title
                </label>
                <input
                  id="title"
                  class={tw("w-full p-2 border border-gray-300 rounded-md")}
                  type="text"
                  name="title"
                  value={data?.title}
                />
                {data?.error?.title && (
                  <p class={tw("text-red-500 text-sm")}>{data.error.title}</p>
                )}
              </div>
              <div>
-              <textarea
-                id="content"
-                rows={10}
-                class={tw("w-full p-2 border border-gray-300 rounded-md")}
-                name="content"
-              />
+              <ContentForm initialValue={data?.content} />
                {data?.error?.content && (
                  <p class={tw("text-red-500 text-sm")}>{data.error.content}</p>
                )}
              </div>
            </div>
            <div class={tw("flex justify-end mt-4")}>
              <button
                class={tw(
                  "bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md"
                )}
                type="submit"
              >
                Create
              </button>
            </div>
          </form>
        </div>
      </div>
    );
  }
```

最後に `import_map.json` に [dompurify](https://github.com/cure53/DOMPurify)  を追加します。（`sanitize-html`` をクライアントで使用できなかったので追加しています）

```json:import_map.json
{
  "imports": {
    "dompurify": "https://esm.sh/dompurify@2.3.8"
  }
}
```

それでは試してみましょう、フォームの入力に応じてプレビューが変化することがわかります！

![preview](//images.ctfassets.net/in6v9lxmm5c8/1CWLrJzM3JJ5E9wCNorOW2/8a936b6b8d6664eeb8b01c29d80749a4/preview.gif)

## デプロイ

最後に作成したアプリケーションをデプロイしましょう！Fresh のデプロイ先として Deno Deploy があげられます。Deno Deploy は Deno の開発元である Deno land 社により提供されているホスティングサービスです。グローバルに分散したエッジランタイムで、開発者が迅速かつ容易にインターネットにウェブアプリケーションをデプロイできます。

https://deno.com/deploy

Deno Deploy にアクセスして Sing up をクリックします。ここでは GitHub の認可が求められます。

![スクリーンショット 2022-06-19 13.45.46](//images.ctfassets.net/in6v9lxmm5c8/2cfjbI9fZnZIY4IaglkwKb/7f8dafd4b7853f1df8cb9b9ff9e65713/____________________________2022-06-19_13.45.46.png)

New Project から新しいプロジェクトを作成します。

![スクリーンショット 2022-06-19 13.48.25](//images.ctfassets.net/in6v9lxmm5c8/3CJrReFtkbXPKGzoHmg0FV/bdc1507e1af59baa82eaa3b8b341a4c3/____________________________2022-06-19_13.48.25.png)

デプロイの方法として、GitHub レポジトリと連携するか、オンラインエディタで編集したファイルを利用するか選択できます。ここでは GitHub レポジトリと連携する方法を選択します。Env Variables も忘れずに追加しておきましょう。

![スクリーンショット 2022-06-19 13.54.27](//images.ctfassets.net/in6v9lxmm5c8/4tck71hmEHm7cMYlFvbDd9/951cfb8bfd6ee9a96d365295c73caf21/____________________________2022-06-19_13.54.27.png)

エントリーファイルには `main.ts` を選択します。

![スクリーンショット 2022-06-19 14.09.21](//images.ctfassets.net/in6v9lxmm5c8/FtZIuKC1eCagMfXRy80eQ/4ce910198b5abf7181ee5baf680c87bc/____________________________2022-06-19_14.09.21.png)

Link をクリックして少し待つとデプロイが完了します！

## おわりに

Deno の Web フレームワークである Fresh を使ったチュートリアルを紹介しました。

[deno.land](https://deno.land/) は Fresh で実装されているようなので、deno.land のソースコードを参考に実装すると良さそうです。

https://github.com/denoland/dotland

以下の記事も大変参考になりました。

https://eh-career.com/engineerhub/entry/2022/06/17/093000

https://zenn.dev/uki00a/articles/frontend-development-in-deno-2022-spring