# zenn-contents

[Zenn](https://zenn.dev/azukiazusa) へ投稿される記事を管理するためのレポジトリです。

小さなtypoなどの修正も歓迎しています！

## Installation

```
npm install
```

## Usage

### 記事を作成する

新しい記事を作成するために、下記コマンドを実行します。

```
npm run new:article
```

### プレビューを表示

プレビューをする場合、ローカルサーバーを利用します。
下記コマンド実行後、 http://localhost:3333 へアクセスすることでプレビューが見られます。

```
npm run preview
```

### Lint

[textint](https://github.com/textlint/textlint) によるLintを実行します。

```
npm run textlint
```
