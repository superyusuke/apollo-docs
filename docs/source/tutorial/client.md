---
title: "5. API をクライアントに接続する"
description: graph を Apollo Client に繋ぎこむ
---

> 翻訳担当：keifff  
> github: https://github.com/keiffff  
> twitter: https://twitter.com/kei_ffff

Time to accomplish: _10 Minutes_

このチュートリアルの後半部分では、Apollo Client を用いて、 graph API を フロントエンドに接続することにのみフォーカスします。 **Apollo Client** はいかなるクライアントライブラリでも使用可能な、データ管理に関する完全なソリューションになります。 Apollo Client は view 層については何も知りません。すなわち、React, Vue, Angular はもちろん、vanilla JS とも統合可能です。賢いキャッシュの仕組みにより、 Apollo Client はあなたのアプリケーションにおけるすべてのローカル、リモートデータに対する、信頼できる唯一の情報源を提供します。

> The next half of this tutorial exclusively focuses on connecting a graph API to a frontend with Apollo Client. **Apollo Client** is a complete data management solution for any client. It's view-layer agnostic, which means it can integrate with React, Vue, Angular, or even vanilla JS. Thanks to its intelligent cache, Apollo Client offers a single source of truth for all of the local and remote data in your application.

Apollo Client はいかなる view 層でも動作しますが、React と共に使用されるのが最も一般的です。この章では、先ほどチュートリアルの前半部分で作成した graph API を React アプリケーションに接続する方法を学ぶでしょう。たとえあなたが Vue や Angular のほうがしっくりくると思っていたとしても、概念は同じなので、例の多くを参考にすることが可能です。このチュートリアルの中ではさらに、認証やページネーションといった、基本的な機能を実装する方法はもちろん、ワークフローを最適化する tips についても学ぶでしょう。

> While Apollo Client works with any view layer, it's most commonly used with React. In this section, you'll learn how to connect the graph API you just built in the previous half of this tutorial to a React app. Even if you're more comfortable with Vue or Angular, you should still be able to follow many of the examples since the concepts are the same. Along the way, you'll also learn how to build essential features like authentication and pagination, as well as tips for optimizing your workflow.

## 開発環境のセットアップ

このチュートリアルの後半部分では、プロジェクトの `client/` フォルダの中で作業します。すでにサーバーの部分からプロジェクトを作成しているはずです、もしそうでない場合、[このチュートリアル](https://github.com/apollographql/fullstack-tutorial/)をクローンするようにしてください。プロジェクトのルートから以下のコマンドを実行します。

> For this half of the tutorial, we will be working in the `client/` folder of the project. You should have the project already from the server portioned, but if you don't, make sure to clone [the tutorial](https://github.com/apollographql/fullstack-tutorial/). From the root of the project, run:

```bash
cd start/client && npm install
```

依存関係がインストールされました。以下がフロントエンドを実装する際に使うパッケージです。

> Now, our dependencies are installed. Here are the packages we will be using to build out our frontend:

- `apollo-client`: 賢いキャッシュの仕組みをもつ、データ管理に関する完全なソリューションです。このチュートリアルでは、 ローカルでの状態管理が可能であることや、キャッシュ管理の仕組みを構築可能であることから、 Apollo Client 3.0 プレビュー版を使用します。
- `react-apollo`: React向けに `Query` や `Mutation` といったコンポーネントをエクスポートする、view 層との統合を実現するためのものです。
- `graphql-tag`: ASTでパースするためのクエリ文字列をラップする、 `gql` というタグ関数です。

> - `apollo-client`: A complete data management solution with an intelligent cache. In this tutorial, we will be using the Apollo Client 3.0 preview since it includes local state management capabilities and sets your cache up for you.
> - `react-apollo`: The view layer integration for React that exports components such as `Query` and `Mutation`
> - `graphql-tag`: The tag function `gql` that we use to wrap our query strings in order to parse them into an AST

### VSCode で Apollo の設定をする

Apollo VSCode は、このチュートリアルを完了するためには必須ではありませんが、設定することで、操作時の補完機能や、定義部分へのコードジャンプ等のたくさんの便利な機能が使用可能になります。

> While Apollo VSCode is not required to successfully complete the tutorial, setting it up unlocks a lot of helpful features such as autocomplete for operations, jump to fragment definitions, and more.

まず、 `client/` にある `.env.example` をコピーし、ファイル名を `.env` にします。 第4章ですでに作成してある Graph Manager API キー をファイルに追加します。

> First, make a copy of the `.env.example` file located in `client/` and call it `.env`. Add your Graph Manager API key that you already created in step #4 to the file:

```
ENGINE_API_KEY=service:<your-service-name>:<hash-from-apollo-engine>
```

基本的に以下のような記入になるはずです。

> The entry should basically look something like this:

```
ENGINE_API_KEY=service:my-service-439:E4VSTiXeFWaSSBgFWXOiSA
```

キーは `ENGINE_API_KEY` という環境変数に保存されました。 Apollo VSCode はレジストリからあなたのスキーマを取得する際にその API キーを使用します。

> Our key is now stored under the environment variable `ENGINE_API_KEY`. Apollo VSCode uses this API key to pull down your schema from the registry.

続いて、 `apollo.config.js` という Apollo の設定ファイルを作成します。この設定ファイルは、 Apollo VSCode の拡張機能と CLI の両方を設定するファイルです。ファイルに以下のスニペットをペーストしてください。

> Next, create an Apollo config file called `apollo.config.js`. This config file is how you configure both the Apollo VSCode extension and CLI. Paste the snippet below into the file:

```js
module.exports = {
  client: {
    name: 'Space Explorer [web]',
    service: 'space-explorer',
  },
};
```

素晴らしい、これで設定はすべて完了です！それでは最初のクライアントを構築するところから始めましょう。

> Great, we're all set up! Let's dive into building our first client.

## Apollo Client を作成する

必要なパッケージはインストールしたので、 `ApolloClient` のインスタンスを作成しましょう。

> Now that we have installed the necessary packages, let's create an `ApolloClient` instance.

`src/index.js` に移動し、クライアントを作成します。 `uri` という引数に渡す値は、第4章でデプロイしたサービスの graph エンドポイントです。

> Navigate to `src/index.js` so we can create our client. The `uri` that we pass in is the graph endpoint from the service you deployed in step 4.

サーバー部分のチュートリアルを完了していなければ、以下のコードから `uri` の値を使用できます。そうでなければ、自分でデプロイしたものの URL を使用してください。それは以下のコードとは異なるものであるかもしれません。 `src/index.js` に移動し、以下のコードをコピーしてください。

> If you didn't complete the server portion, you can use the `uri` from the code below. Otherwise, use your own deployment's URL, which may be different than the one below. Navigate to `src/index.js` and copy the code below:

_src/index.js_

```js
import { ApolloClient } from 'apollo-client';
import { InMemoryCache } from 'apollo-cache-inmemory';
import { HttpLink } from 'apollo-link-http';

const cache = new InMemoryCache();
const link = new HttpLink({
  uri: 'http://localhost:4000/'
});

const client = new ApolloClient({
  cache,
  link
});
```

たった数行のコードですが、クライアントは、データをフェッチする準備ができました！次のセクションでクエリを作成してみましょう。

> In just a few lines of code, our client is ready to fetch data! Let's try making a query in the next section.

## 最初のクエリを作成する

Apollo を React と統合する方法をお見せする前に、 vanila JavaScript でクエリを送信してみましょう。

> Before we show you how to use the React integration for Apollo, let's send a query with vanilla JavaScript.

`client.query()` が呼ばれると、 qraph API のクエリを使用できます。 `src/index.js` のインポート部分に、以下のコードを追加してください。

> With a `client.query()` call, we can query our graph's API. Add the following line of code to your imports in `src/index.js`.

_src/index.js_

```js
import gql from "graphql-tag"; // highlight-line
```

そして、 `index.js` のいちばん下に、以下のコードを追加してください。

> And add this code to the bottom of `index.js`:

_src/index.js_

```js
// ...上はクライアントオブジェクトをインスタンス化しているコードです。
client
  .query({
    query: gql`
      query GetLaunch {
        launch(id: 56) {
          id
          mission {
            name
          }
        }
      }
    `
  })
  .then(result => console.log(result));
```

コンソールを開いて、 `npm start` を実行してください。あなたのクライアントアプリケーションをコンパイルするでしょう。それが終了すると、自動的にブラウザが `http://localhost:3000/` を開きます。インデックスページが開いたら、[デベロッパーツール コンソール](https://developers.google.com/web/tools/chrome-devtools/console/)を開きます。すると、クエリの結果を含む、 `data` という値を持つオブジェクトが見えるはずです。他にも、 `loading` や `networkStatus` といった値が見えるでしょう。それは、 Apollo Client が、あなたのクエリのローディング状態を追跡しているからなのです。

> Open up your console and run `npm start`. This will compile your client app. Once it is finished, your browser should open to `http://localhost:3000/` automatically. When the index page opens, open up your [Developer Tools console](https://developers.google.com/web/tools/chrome-devtools/console/) and you should see an object with a `data` property containing the result of our query. You'll also see some other properties, like `loading` and `networkStatus`. This is because Apollo Client tracks the loading state of your query for you.

Apollo Client は、いかなる JavaScript フロントエンドからも graph データをフェッチできるようにデザインされています。フレームワークは必要ありません。しかし、　異なるフレームワークに対して、クエリをUIに結びつけるのを簡単にするための view 層との統合があります。

> Apollo Client is designed to fetch graph data from any JavaScript frontend. No frameworks needed. However, there are view layer integrations for different frameworks that makes it easier to bind queries to the UI.

次に進み、 先ほど作成した、 `client.query()` の呼び出しと、 `gql` のインポート宣言を削除してください。それでは、クライアントをReactに接続しましょう。

> Go ahead and delete the `client.query()` call you just made and the `gql` import statement. Now, we'll connect our client to React.

## クライアントを React に接続する

Apollo の hooks を用いて、 Apollo Client を React アプリケーション に接続することは、 GraphQL の操作を UI に結びつけることを簡単にします。

> Connecting Apollo Client to our React app with Apollo's hooks allows us to easily bind GraphQL operations to our UI.

Apollo Client を React に接続するために、 `@apollo/react-hooks` パッケージからエクスポートされた、 `ApolloProvider` コンポーネントでアプリケーションをラップして、 `client` というpropにクライアントを渡します。 `ApolloProvider` コンポーネントは、 React の context provider に似ています。それは React アプリケーションをラップし、コンテキストにクライアントを渡します。そうすることで、コンポーネントツリーの中のどの場所でも、アクセスが可能になります。

> To connect Apollo Client to React, we will wrap our app in the `ApolloProvider` component exported from the `@apollo/react-hooks` package and pass our client to the `client` prop. The `ApolloProvider` component is similar to React’s context provider. It wraps your React app and places the client on the context, which allows you to access it from anywhere in your component tree.

`src/index.js` を開き、以下のコードを追加してください。

> Open `src/index.js` and add the following lines of code:

_src/index.js_

```jsx
import { ApolloProvider } from '@apollo/react-hooks';
import React from 'react';
import ReactDOM from 'react-dom';
import Pages from './pages';

// 前に行った変数宣言

ReactDOM.render(
  <ApolloProvider client={client}>
    <Pages />
  </ApolloProvider>, document.getElementById('root')
);
```

次のセクションで、 `useQuery` フックを用いて最初のコンポーネントを実装するための準備が整いました。

> Now, we're ready to start building our first component with the `useQuery` hook in the next section.
