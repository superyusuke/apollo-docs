---
title: "6. クエリを用いてデータをフェッチする"
description: useQuery フックを用いてデータをフェッチする方法を学ぶ
---

 Time to accomplish: _15 Minutes_

Apollo Client ではシンプルな仕組みで graph API からデータをフェッチできます。というのも、Apollo Client は賢い仕組みででデータをキャッシュするだけでなく、ローディング状態やエラー状態も追跡するためです。前のセクションでは、 view への組み込みはせずに、 Apollo Client を用いてサンプルクエリをフェッチする方法を学びました。このセクションでは、より複雑なクエリをフェッチしたり、ページネーションのような機能を実装するために、 `@apollo/react-hooks` が提供する `useQuery` フックを使う方法を学びます。

> Apollo Client simplifies fetching data from a graph API because it intelligently caches your data, as well as tracks loading and error state. In the previous section, we learned how to fetch a sample query with Apollo Client without using a view integration. In this section, we'll learn how to use the `useQuery` hook from `@apollo/react-hooks` to fetch more complex queries and execute features like pagination.

## useQuery フック

`useQuery` フックは Apollo アプリケーションの中でも最も重要な構成要素の一つです。 `useQuery` フックは、 GraphQL クエリをフェッチし、フェッチしたデータをもとに UI を描画できるような結果を返す React フックです。

> The `useQuery` hook is one of the most important building blocks of an Apollo app. It's a React Hook that fetches a GraphQL query and exposes the result so you can render your UI based on the data it returns.

`useQuery` フックは、クエリからデータをフェッチし、 UI にロードします。 `useQuery` フックは、 result オブジェクトを介して、 `error` `loading` `data` といった値を返します。それらは、コンポーネントにデータを渡したり、コンポーネントを描画するのに役立ちます。  
例を見てみましょう。

> The `useQuery` hook leverages React's [Hooks API](https://reactjs.org/docs/hooks-intro.html) to fetch and load data from queries into our UI. It exposes `error`, `loading` and `data` properties through a result object, that help us populate and render our component. Let's see an example:

## リストをフェッチする

`useQuery` を用いてコンポーネントを作成するには、 `@apollo/react-hooks` から `useQuery` をインポートします。そして、 `gql` でラップされたクエリを第一引数に渡します。そして、 result オブジェクトに含まれる `loading` `data` `error` プロパティをコンポーネントに繋ぎこむことで、アプリケーションの UI を描画します。

> To create a component with `useQuery`, import `useQuery` from `@apollo/react-hooks`, pass your query wrapped with `gql` in as the first parameter, then wire your component up to use the `loading`, `data`, and `error` properties on the result object to render UI in your app.

まずはじめに、 launches の一覧をフェッチする GraphQL クエリを作成します。また、次のステップで必要になるいくつかのコンポーネントを先にインポートしておきます。 `src/pages/launches.js` に移動し、以下のコードをファイルにコピーしましょう。

> First, we're going to build a GraphQL query that fetches a list of launches. We're also going to import some components that we will need in the next step. Navigate to `src/pages/launches.js` to get started and copy the code below into the file.

_src/pages/launches.js_

```js
import React, { Fragment } from 'react';
import { useQuery } from '@apollo/react-hooks';
import gql from 'graphql-tag';

import { LaunchTile, Header, Button, Loading } from '../components';

const GET_LAUNCHES = gql`
  query launchList($after: String) {
    launches(after: $after) {
      cursor
      hasMore
      launches {
        id
        isBooked
        rocket {
          id
          name
        }
        mission {
          name
          missionPatch
        }
      }
    }
  }
`;
```

それでは、われわれのスキーマから、 `launches` クエリをコールし、 launches の一覧をフェッチするクエリを定義します。 `launches` クエリは、 launches の一覧 (`launches`) に加え、ページングされたリストのカーソル (`cursor`) 、 リストが更に launches を持つかどうか (`hasMore`) を含む object type を返します。クエリを AST でパースするには、クエリを `gql` 関数でラップする必要があります。

> Here, we're defining a query to fetch a list of launches by calling the `launches` query from our schema. The `launches` query returns an object type with a list of launches, in addition to the `cursor` of the paginated list and whether or not the list `hasMore` launches. We need to wrap the query with the `gql` function in order to parse it into an AST.

それでは、 Apollo の `useQuery` コンポーネントにクエリを渡し、リストを描画してみましょう。

> Now, let's pass that query to Apollo's `useQuery` component to render the list:

_src/pages/launches.js_

```jsx
export default function Launches() {
  const { data, loading, error } = useQuery(GET_LAUNCHES);
  if (loading) return <Loading />;
  if (error) return <p>ERROR</p>;

  return (
    <Fragment>
      <Header />
      {data.launches &&
        data.launches.launches &&
        data.launches.launches.map(launch => (
          <LaunchTile
            key={launch.id}
            launch={launch}
          />
        ))}
    </Fragment>
  );
}
```

リストを出力するためには、先ほどのステップで作成した、 `GET_LAUNCHES` クエリを `useQuery` フックに渡す必要があります。そして、`loading` `error` `data` のそれぞれの状態に応じて、 ローディングインジケータ、エラーメッセージ、もしくは launches の一覧を描画します。

> To render the list, we pass the `GET_LAUNCHES` query from the previous step into our `useQuery` hook. Then, depending on the state of `loading`, `error`, and `data`, we either render a loading indicator, an error message, or a list of launches.

まだこれで終わりではありません！このクエリは、一覧から 最初の20件の launches をフェッチしただけです。 launches のすべての一覧をフェッチするには、画面上にさらに結果を表示するための `Load More` ボタンを表示する、ページネーション機能が必要になります。ではその方法を学びましょう！

> We're not done yet! Right now, this query is only fetching the first 20 launches from the list. To fetch the full list of launches, we need to build a pagination feature that displays a `Load More` button for loading more items on the screen. Let's learn how!

### ページングされた一覧を実装する

Apollo Client には、われわれがページネーションのロジックを自前で書くよりももっと簡単な方法で、アプリケーションにページネーションを追加するための、すでに用意されたヘルパーがあります。

> Apollo Client has built-in helpers to make adding pagination to our app much easier than it would be if we were writing the logic ourselves.

Apollo でページングされたリストを実装するには、まず `useQuery` の result オブジェクトを destructure して得られる、 `fetchMore` 関数が必要になります。

> To build a paginated list with Apollo, we first need to destructure the `fetchMore` function from the `useQuery` result object:

_src/pages/launches.js_

```jsx
export default function Launches() {
  const { data, loading, error, fetchMore } = useQuery(GET_LAUNCHES); // highlight-line
  return (
    // 上と同じ
  );
}
```

`fetchMore` を得られたので、クリックされた際に更に結果をフェッチする Load More ボタンに `fetchMore` を繋ぎこみましょう。そのためには、 `fetchMore` に、 `updateQuery` 関数を指定する必要があります。 それはApollo のキャッシュにわれわれがフェッチしている新しい結果を用いてクエリを更新するよう伝えます。

> Now that we have `fetchMore`, let's connect it to a Load More button to fetch more items when it's clicked. To do this, we will need to specify an `updateQuery` function on the return object from `fetchMore` that tells the Apollo cache how to update our query with the new items we're fetching.

以下のコードをコピーし、先ほどのステップで追加したレンダープロップ関数の中の `</Fragment>` 閉じタグの上に追加します。

> Copy the code below and add it above the closing `</Fragment>` tag in the render prop function we added in the previous step.

_src/pages/launches.js_

```jsx
{data.launches &&
  data.launches.hasMore && (
    <Button
      onClick={() =>
        fetchMore({ // highlight-line
          variables: {
            after: data.launches.cursor,
          },
          updateQuery: (prev, { fetchMoreResult, ...rest }) => { // highlight-line
            if (!fetchMoreResult) return prev;
            return {
              ...fetchMoreResult,
              launches: {
                ...fetchMoreResult.launches,
                launches: [
                  ...prev.launches.launches,
                  ...fetchMoreResult.launches.launches,
                ],
              },
            };
          },
        })
      }
    >
      Load More
    </Button>
  )
}
```

まず、クエリの中でそれ以上 launches の結果があるかどうかチェックします。もしあるなら、 Apollo の `fetchMore` 関数を実行するクリックハンドラを持つボタンを描画します。 `fetchMore` 関数は launches の一覧をフェッチするクエリに渡すための、新しい変数を受け取ります。その変数は、われわれの例では cursor として使われています。

> First, we check to see if we have more launches available in our query. If we do, we render a button with a click handler that calls the `fetchMore` function from Apollo. The `fetchMore` function receives new variables for the list of launches query, which is represented by our cursor.

また、 Apollo にキャッシュの中にある launches の一覧を更新するよう伝える `updateQuery` 関数を定義します。そのためには、前のクエリの結果を利用して、 `fetchMore` から得られる新しいクエリの結果と組み合わせます。

> We also define the `updateQuery` function to tell Apollo how to update the list of launches in the cache. To do this, we take the previous query result and combine it with the new query result from `fetchMore`.

次のステップでは、一覧の中のアイテムがクリックされた際に、単一の launch の詳細を表示するページを作成する方法を学びます。

> In the next step, we'll learn how to wire up the launch detail page to display a single launch when an item in the list is clicked.

## 単一の launch をフェッチする

詳細ページを実装するために、 `src/pages/launch.js` に移動しましょう。まず、いくつかのコンポーネントをインポートし、 launch の詳細を取得する GraphQL クエリを定義します。

> Let's navigate to `src/pages/launch.js` to build out our detail page. First, we should import some components and define our GraphQL query to get the launch details.

_src/pages/launch.js_

```jsx
import React, { Fragment } from 'react';
import { useQuery } from '@apollo/react-hooks';
import gql from 'graphql-tag';

import { Loading, Header, LaunchDetail } from '../components';
import { ActionButton } from '../containers';

export const GET_LAUNCH_DETAILS = gql`
  query LaunchDetails($launchId: ID!) {
    launch(id: $launchId) {
      id
      site
      isBooked
      rocket {
        id
        name
        type
      }
      mission {
        name
        missionPatch
      }
    }
  }
`;
```

クエリを記述したので、 `useQuery` を用いてクエリを実行するためのコンポーネントを描画しましょう。今回は、 `launchId` を変数としてクエリに渡す必要があります。それは、 `useQuery` の `variable` オプションに追加することで可能です。 `launchId` は router から prop として渡ってきます。

> Now that we have a query, let's render a component with `useQuery` to execute it. This time, we'll also need to pass in the `launchId` as a variable to the query, which we'll do by adding a `variables` option to `useQuery`. The `launchId` comes through as a prop from the router.

_src/pages/launch.js_

```jsx
export default function Launch({ launchId }) {
  const { data, loading, error } = useQuery(
    GET_LAUNCH_DETAILS,
    { variables: { launchId } }
  );
  if (loading) return <Loading />;
  if (error) return <p>ERROR: {error.message}</p>;

  return (
    <Fragment>
      <Header image={data.launch.mission.missionPatch}>
        {data.launch.mission.name}
      </Header>
      <LaunchDetail {...data.launch} />
      <ActionButton {...data.launch} />
    </Fragment>
  );
}
```

先ほどと同じように、 `loading` 状態、 `error` 状態、データの取得が完了した状態を描画するために、クエリの状態を使用します、

> Just like before, we use the status of the query to render either a `loading` or `error` state, or data when the query completes.

### コードの共通化のために fragments を使用する

お気づきかもしれませんが、 launches の一覧をフェッチするクエリと、単一の launch をフェッチするクエリには、同じフィールドがたくさんあります。同じフィールドを含む2つの GraphQL オペレーションがある場合には、同じフィールドを共通化するための、 **fragment** を使用できます。

> You may have noticed that the queries for fetching a list of launches and fetching a launch detail share a lot of the same fields. When we have two GraphQL operations that contain the same fields, we can use a **fragment** to share fields between the two.

fragment を実装する方法を学ぶために、 `src/pages/launches.js` に移動し、以下のコードをコピーしてファイルに記述してください。

> To learn how to build a fragment, navigate to `src/pages/launches.js` and copy the code below into the file:

_`src/pages/launches.js`_

```js
export const LAUNCH_TILE_DATA = gql`
  fragment LaunchTile on Launch {
    id
    isBooked
    rocket {
      id
      name
    }
    mission {
      name
      missionPatch
    }
  }
`;
```

GraphQL フラグメントを定義し、 `LaunchTile` と命名します。そして、 スキーマ (`Launch`) の上に定義します。フラグメントにつける名前は何でも構いませんが、フラグメントの type は、スキーマに含まれる type と対応していなければいけません。

> We define a GraphQL fragment by giving it a name (`LaunchTile`) and defining it on a type on our schema (`Launch`). The name we give our fragment can be anything, but the type must correspond to a type in our schema.

クエリの中でフラグメントを使用するために、 GraphQL ドキュメントの中にフラグメントをインポートし、 スプレッド演算子を用いて、クエリの中でフィールドを展開します。

> To use our fragment in our query, we import it into the GraphQL document and use the spread operator to spread the fields into our query:

_`src/pages/launches.js`_

```js{6,10}
const GET_LAUNCHES = gql`
  query launchList($after: String) {
    launches(after: $after) {
      cursor
      hasMore
      launches {
        ...LaunchTile
      }
    }
  }
  ${LAUNCH_TILE_DATA}
`;
```

launch の詳細クエリにも、フラグメントを適用しましょう。フラグメントを使う前に、必ず `launches` ページからフラグメントをインポートするようにしてください。

> Let's use our fragment in our launch detail query too. Be sure to import the fragment from the `launches` page before you use it:

_src/pages/launch.js_

```js{1,10,13}
import { LAUNCH_TILE_DATA } from './launches';

export const GET_LAUNCH_DETAILS = gql`
  query LaunchDetails($launchId: ID!) {
    launch(id: $launchId) {
      site
      rocket {
        type
      }
      ...LaunchTile
    }
  }
  ${LAUNCH_TILE_DATA}
`;
```

素晴らしい。これでフラグメントを用いてクエリをリファクタリングすることができました。フラグメントは、 GraphQL クエリやミューテーションを実装するなら多用することになる、便利なツールです。

> Great, now we've successfully refactored our queries to use fragments. Fragments are a helpful tool that you'll use a lot as you're building GraphQL queries and mutations.

### フェッチポリシーをカスタマイズする

時々、常に更新する必要があるデータがある場合には、Apollo Client にキャッシュを完全にバイパスするように伝えたほうが便利でしょう。 `useQuery` フックの `fetchPolicy` をカスタマイズすることで、これを実現できます。

> Sometimes, it's useful to tell Apollo Client to bypass the cache altogether if you have some data that constantly needs to be refreshed. We can do this by customizing the `useQuery` hook's `fetchPolicy`.

First, let's navigate to `src/pages/profile.js` and write our query:

_src/pages/profile.js_

```js
import React, { Fragment } from 'react';
import { useQuery } from '@apollo/react-hooks';
import gql from 'graphql-tag';

import { Loading, Header, LaunchTile } from '../components';
import { LAUNCH_TILE_DATA } from './launches';

const GET_MY_TRIPS = gql`
  query GetMyTrips {
    me {
      id
      email
      trips {
        ...LaunchTile
      }
    }
  }
  ${LAUNCH_TILE_DATA}
`;
```

続いて、ログイン済みユーザーの trip の一覧をフェッチする `useQuery` を用いたコンポーネントを描画しましょう。デフォルトでは、 Apollo Client のフェッチポリシーは `cache-first` です。それは、ネットワークリクエストを送信する前に、キャッシュを見て結果があるかどうかを確認するということを意味しています。われわれは、この一覧が常に graph API からの最新のデータを反映していることを期待しているので、このクエリの `fetchPolicy` を `network-only` にセットしておきます。

> Next, let's render a component with `useQuery` to fetch a logged in user's list of trips. By default, Apollo Client's fetch policy is `cache-first`, which means it checks the cache to see if the result is there before making a network request. Since we want this list to always reflect the newest data from our graph API, we set the `fetchPolicy` for this query to `network-only`:

_src/pages/profile.js_

```jsx
export default function Profile() {
  const { data, loading, error } = useQuery(
    GET_MY_TRIPS,
    { fetchPolicy: "network-only" } // highlight-line
  );

  if (loading) return <Loading />;
  if (error) return <p>ERROR: {error.message}</p>;

  return (
    <Fragment>
      <Header>My Trips</Header>
      {data.me && data.me.trips.length ? (
        data.me.trips.map(launch => (
          <LaunchTile key={launch.id} launch={launch} />
        ))
      ) : (
        <p>You haven't booked any trips</p>
      )}
    </Fragment>
  );
}
```

このクエリを描画すると、 null が返ってくることに気づくでしょう。まずログイン機能を実装する必要があるためです。次のセクションでログインに挑戦します。

> If you try to render this query, you'll notice that it returns null. This is because we need to implement our login feature first. We're going to tackle login in the next section.

これで ページングされた一覧をフェッチするコンポーネント、フラグメントの共通化、フェッチポリシーのカスタマイズを実装すべく、 `useQuery` を最大限活用する方法について学びました。 次のセクションに進み、ミューテーションを用いてデータを更新する方法を学びましょう。

> Now that we've learned how to leverage `useQuery` to build components that can fetch a paginated list, share fragments, and customize the fetch policy, it's time to progress to the next section so we can learn how to update data with mutations!
