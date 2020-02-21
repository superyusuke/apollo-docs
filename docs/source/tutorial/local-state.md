---
title: '8. local state の管理'
description: How to store and query local data in the Apollo cache
---

Time to accomplish: _15 Minutes_

実際のフロントエンドアプリケーションでは、GraphAPI から取得したデータに加えて、ネットワークの状態やフォームの状態といった「ローカルデータ」も表示する必要があります。実は Apollo Client は、ローカルデータを Apollo cache の内部に保持し、それを GraphQL の remote データを取得する際に同時に取得することができるのです。

> In almost every app we build, we display a combination of remote data from our graph API and local data such as network status, form state, and more. What's awesome about Apollo Client is that it allows us to store local data inside the Apollo cache and query it alongside our remote data with GraphQL.

われわれは local state についても Apollo cache で管理すること推奨しており、Redux などの状態管理ライブラリは使わなくてもいいのではないかと思っています。そうすることで Apollo が「a single source of truth」つまりデータ一元的管理場所となることができるのです。

> We recommend managing local state in the Apollo cache instead of bringing in another state management library like Redux so the Apollo cache can be a single source of truth.

ローカルデータを Apollo Client で管理する方法は、ここまでのチュートリアルで remote data を扱ってきた方法とほとんど同じです。ローカルデータを管理する Client のための schema を定義し、Client のための resolver を書きます。そのデータを GraphQL を使って取得する手法も学んでいきます。その際には `@client` directive をつけるだけです。ではいきましょう！

> Managing local data with Apollo Client is very similar to how you've already managed remote data in this tutorial. You'll write a client schema and resolvers for your local data. You'll also learn to query it with GraphQL just by specifying the `@client` directive. Let's dive in!

### Write a local schema

サーバー側のデータ構造を定義するために、まず schema を書いたように、クライアントで持つデータの構造を定義するために local schema をまず定義します。

> Just like how a schema is the first step toward defining our data model on the server, writing a local schema is the first step we take on the client.

`src/resolvers.js` に移動し、以下のコードを複写し、client schema を作成しましょう。

> Navigate to `src/resolvers.js` and copy the following code to create your client schema (as well as blank client resolvers for later):

_src/resolvers.js_

```js
import gql from 'graphql-tag';

export const typeDefs = gql`
  extend type Query {
    isLoggedIn: Boolean!
    cartItems: [ID!]!
  }

  extend type Launch {
    isInCart: Boolean!
  }

  extend type Mutation {
    addOrRemoveFromCart(id: ID!): [Launch]
  }
`;

export const resolvers = {};
```

Client schema を書く際には server schema を **extend** して用います。そしてそれをいつも通り `gql` function でラップします。extend keyword を用いることで Apollo VSCode や Apollo DevTools といった developer tooling 内部の schema も統合することができます。（訳注：どちらも使ったことがないのでピンときていない。もしかしたら訳が間違っているかも。）

> To build a client schema, we **extend** the types of our server schema and wrap it with the `gql` function. Using the extend keyword allows us to combine both schemas inside developer tooling like Apollo VSCode and Apollo DevTools.

またサーバーから取得したデータに対して local フィールドを追加することもできます。その際には server 側の type を extend します。今回は `isInCart` という local field  `Launch` type に追加しました。`Launch` type　は graph API から取得されるデータです。

> We can also add local fields to server data by extending types from our server. Here, we're adding the `isInCart` local field to the `Launch` type we receive back from our graph API.

## Initialize the store

Client 側の schema を定義しましたので、次は store を初期化する手法を学びます。query はコンポーネントがマウントされたその瞬間に実行されますので、Apollo cache にデフォルト値を前もって与えて行く必要があります。そうでないとエラーが起きてしまいます。では `isLoggedIn` と `cartItems` の二つのローカル data に初期値を書き込みましょう。

> Now that we've created our client schema, let's learn how to initialize the store. Since queries execute as soon as the component mounts, it's important for us to warm the Apollo cache with some default state so those queries don't error out. We will need to write initial data to the cache for both `isLoggedIn` and `cartItems`:

`src/index.js` に戻って、`cache.writeData` を実行して cache に値を準備していたことを確認しましょう。また後ほど使う `typeDefs` も `resolvers` import しましょう。

> Jump back to `src/index.js` and notice we had already added a `cache.writeData` call to prepare the cache in the last section. While we're here, make sure to also import the `typeDefs` and `resolvers` that we just created so we can use them later:

_src/index.js_

```js{1,11-12,15-20}
import { resolvers, typeDefs } from './resolvers';

const client = new ApolloClient({
  cache,
  link: new HttpLink({
    uri: 'http://localhost:4000/graphql',
    headers: {
      authorization: localStorage.getItem('token'),
    },
  }),
  typeDefs,
  resolvers,
});

cache.writeData({
  data: {
    isLoggedIn: !!localStorage.getItem('token'),
    cartItems: [],
  },
});
```

これで Apollo cache にデフォルト値を与えることができたので、次はローカルデータを React component から query して取得する方法を学びましょう。

> Now that we've added default state to the Apollo cache, let's learn how to query local data from within our React components.

## Query local data

Apollo cache から　local data を query で取得するには、Graph API からデータを query を使って取得するのとほとんど同じ手法を取ります。違いは `@client` directive を local field にはつけないといけない点です。こうすることで　Apollo Client に対して local cache から値を取得するということを伝えます。

> Querying local data from the Apollo cache is almost the same as querying remote data from a graph API. The only difference is that you add a `@client` directive to a local field to tell Apollo Client to pull it from the cache.

では `isLoggedIn` field の値を取得する query の例をみてみましょう。この値は一つ前の章で扱ったもので、mutation を使って更新できるようにしたものですね。

> Let's look at an example where we query the `isLoggedIn` field we wrote to the cache in the last mutation exercise.

_src/index.js_

```jsx{8-12,15}
import { ApolloProvider, useQuery } from '@apollo/react-hooks';
import gql from 'graphql-tag';

import Pages from './pages';
import Login from './pages/login';
import injectStyles from './styles';

const IS_LOGGED_IN = gql`
  query IsUserLoggedIn {
    isLoggedIn @client
  }
`;

function IsLoggedIn() {
  const { data } = useQuery(IS_LOGGED_IN);
  return data.isLoggedIn ? <Pages /> : <Login />;
}

injectStyles();
ReactDOM.render(
  <ApolloProvider client={client}>
    <IsLoggedIn />
  </ApolloProvider>,
  document.getElementById('root'),
);
```

まず `IsUserLoggedIn` という local query を作りました。この `isLoggedIn` field には `@client` directive を付与しました。そして `useQuery` を用いるコンポーネントを作成し、これに query を渡しました。この query のレスポンスに基づいて（つまりユーザーがログインしているかどうかに基づいて）、ログイン画面をレンダーするかホームページをレンダリングするかが決まります。Local cache の値の読みだしは、同期的に行われますのでローディング状態かどうかを気にする必要はありません。

> First, we create our `IsUserLoggedIn` local query by adding the `@client` directive to the `isLoggedIn` field. Then, we render a component with `useQuery`, pass our local query in, and based on the response render either a login screen or the homepage depending if the user is logged in. Since cache reads are synchronous, we don't have to account for any loading state.

では local state から取得する query を実行するコンポーネントの、別の例をみてみましょう。`src/pages/cart.js` です。これも同様にクエリをまず作成します。

> Let's look at another example of a component that queries local state in `src/pages/cart.js`. Just like before, we create our query:

_src/pages/cart.js_

```js
import React, { Fragment } from 'react';
import { useQuery } from '@apollo/react-hooks';
import gql from 'graphql-tag';

import { Header, Loading } from '../components';
import { CartItem, BookTrips } from '../containers';

export const GET_CART_ITEMS = gql`
  query GetCartItems {
    cartItems @client
  }
`;
```

次に `useQuery` に `GetCartItems` query を渡し実行します。

> Next, we call `useQuery` and bind it to our `GetCartItems` query:

_src/pages/cart.js_

```jsx
export default function Cart() {
  const { data, loading, error } = useQuery(GET_CART_ITEMS);
  if (loading) return <Loading />;
  if (error) return <p>ERROR: {error.message}</p>;
  return (
    <Fragment>
      <Header>My Cart</Header>
      {!data.cartItems || !data.cartItems.length ? (
        <p data-testid="empty-message">No items in your cart</p>
      ) : (
        <Fragment>
          {data.cartItems.map(launchId => (
            <CartItem key={launchId} launchId={launchId} />
          ))}
          <BookTrips cartItems={data.cartItems} />
        </Fragment>
      )}
    </Fragment>
  );
}
```

実は local query と remote query を混ぜて用いることが可能です。Local data を GraphQL を用いて取得する方法は完全に理解できたと思いますので、次は local field を server から取得した値に追加する方法をお教えしましょう。

> It's important to note that you can mix local queries with remote queries in a single GraphQL document. Now that you're a pro at querying local data with GraphQL, let's learn how to add local fields to server data.

### Adding virtual fields to server data

Apollo Client で local data を管理する特徴的な利点の一つは、graph API から取得したデータに対して **virtual fields** を追加できることです。追加した **virtual fields** はクライアント側にのみ存在します。これによってサーバーから取得したデータを、local state 修飾することができます。わたしたちの例では `Launch` type に対して `isInCart` という virtual field を追加します。

> One of the unique advantages of managing your local data with Apollo Client is that you can add **virtual fields** to data you receive back from your graph API. These fields only exist on the client and are useful for decorating server data with local state. In our example, we're going to add an `isInCart` virtual field to our `Launch` type.

Virtual field を追加するためには、まずヴァーチャルフィールドを追加したいサーバー側の type を extend し、フィールドを追加します。

> To add a virtual field, first extend the type of the data you're adding the field to in your client schema. Here, we're extending the `Launch` type:

_src/resolvers.js_

```js
import gql from 'graphql-tag';

export const schema = gql`
  extend type Launch {
    isInCart: Boolean!
  }
`;
```

次に the `Launch` type の client resolver を定義し、virtual field をどのように解決するかを指定します。

> Next, specify a client resolver on the `Launch` type to tell Apollo Client how to resolve your virtual field:

_src/resolvers.js_

```js
export const resolvers = {
  Launch: {
    isInCart: (launch, _, { cache }) => {
      const { cartItems } = cache.readQuery({ query: GET_CART_ITEMS });
      return cartItems.includes(launch.id);
    },
  },
};
```

Client resolver については後ほど詳しく取り上げる予定です。ただ基本的には client 側の resolver の書き方は、すでにとりあげた server 側の resolver の書き方と同じです。

> We're going to learn more about client resolvers in the section below. The important thing to note is that the resolver API on the client is the same as the resolver API on the server.

ではついに launch 詳細ページの中で virtual field からクエリで値を取得することにしましょう。以前の例と同じく、`@client` directive を query の virtual filed に追加するだけです。

> Now, you're ready to query your virtual field on the launch detail page! Similar to the previous examples, just add your virtual field to a query and specify the `@client` directive.

_src/pages/launch.js_

```js{4}
export const GET_LAUNCH_DETAILS = gql`
  query LaunchDetails($launchId: ID!) {
    launch(id: $launchId) {
      isInCart @client
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

## Update local data

今のところ、local data を query で取得することだけにフォーカスしてきましたが、Apollo Client は local data を更新することもできます。更新する手法には  **direct cache writes** と **client resolvers** の二つがあります。Direct writes は boolean や単なる文字列のように比較的簡単なデータを更新する際に用いられます。それにたいして Client resolver はより込み入ったデータを更新する場合に用いられます。例えばリストの更新などです。

> Up until now, we've focused on querying local data from the Apollo cache. Apollo Client also lets you update local data in the cache with either **direct cache writes** or **client resolvers**. Direct writes are typically used to write simple booleans or strings to the cache whereas client resolvers are for more complicated writes such as adding or removing data from a list.

### Direct cache writes

Direct cache writes は、値が boolean や string といった簡単な field を書き換える場合に便利な手法です。実行するためには `client.writeData()` を呼び出し、オブジェクトを渡します。このオブジェクトは、data プロパティを持ち、更新したいデータに対応する情報を持ったものにします。すでにこのやり方については、ログインが `onCompleted` であることを示す値の更新のために `client.writeData` を使って mutation 後に書き換える、という手法を紹介していましたね。今回もそれとかなり似たシンプルなものを作ります。以下のコードをコピーしてログアウトボタンを作成しましょう。

> Direct cache writes are convenient when you want to write a simple field, like a boolean or a string, to the Apollo cache. We perform a direct write by calling `client.writeData()` and passing in an object with a data property that corresponds to the data we want to write to the cache. We've already seen an example of a direct write, when we called `client.writeData` in the `onCompleted` handler for the login `useMutation` based component. Let's look at a similar example, where we copy the code below to create a logout button:

_src/containers/logout-button.js_

```jsx
import React from 'react';
import styled from 'react-emotion';
import { useApolloClient } from '@apollo/react-hooks';

import { menuItemClassName } from '../components/menu-item';
import { ReactComponent as ExitIcon } from '../assets/icons/exit.svg';

export default function LogoutButton() {
  const client = useApolloClient();
  return (
    <StyledButton
      onClick={() => {
        client.writeData({ data: { isLoggedIn: false } }); // highlight-line
        localStorage.clear();
      }}
    >
      <ExitIcon />
      Logout
    </StyledButton>
  );
}

const StyledButton = styled('button')(menuItemClassName, {
  background: 'none',
  border: 'none',
  padding: 0,
});
```

ボタンをクリックすると、`client.writeData` 呼び出されます。それに `isLoggedIn` を false にするためのデータオブジェクトを渡しています。こうして cache が無事、更新されます。

> When we click the button, we perform a direct cache write by calling `client.writeData` and passing in a data object that sets the `isLoggedIn` boolean to false.

もちろん `useMutation` の `update` 関数の中で direct writes を用いることもできます。こうすることで Mutation が起きた後に、データを際フェッチすることなく手動で cache を更新することができます。では `src/containers/book-trips.js` の例を見てみましょう。

> We can also perform direct writes within the `update` function of the `useMutation` hook. The `update` function allows us to manually update the cache after a mutation occurs without refetching data. Let's look at an example in `src/containers/book-trips.js`:

_src/containers/book-trips.js_

```jsx{29-31}
import React from 'react';
import { useMutation } from '@apollo/react-hooks';
import gql from 'graphql-tag';

import Button from '../components/button';
import { GET_LAUNCH } from './cart-item';

const BOOK_TRIPS = gql`
  mutation BookTrips($launchIds: [ID]!) {
    bookTrips(launchIds: $launchIds) {
      success
      message
      launches {
        id
        isBooked
      }
    }
  }
`;

export default function BookTrips({ cartItems }) {
  const [bookTrips, { data, loading, error }] = useMutation(
    BOOK_TRIPS,
    {
      refetchQueries: cartItems.map(launchId => ({
        query: GET_LAUNCH,
        variables: { launchId },
      })),
      update(cache) {
        cache.writeData({ data: { cartItems: [] } });
      }
    }
  )
  return data && data.bookTrips && !data.bookTrips.success
    ? <p data-testid="message">{data.bookTrips.message}</p>
    : (
      <Button onClick={bookTrips} data-testid="book-button">
        Book All
      </Button>
    );
}
```

この例では `cache.writeData` を直接実行して、`BookTrips` の mutation がサーバ０に送られた後に、`cartItems` の状態をリセットしています。この direct write は update 関数の中で実行されています。

> In this example, we're directly calling `cache.writeData` to reset the state of the `cartItems` after the `BookTrips` mutation is sent to the server. This direct write is performed inside of the update function, which is passed our Apollo Client instance.

### Local resolvers

まだ終わりではありません。もしより複雑な local data を更新したい場合にはどうしたらいいのでしょうか。例えば list の item を追加したり削除したりする場合です。こういった場合には、local resolver を用いるのが良いでしょう。Local resolver は remote resolver と同じ構造を持つ関数です。つまり、おなじみの `(parent, args, context, info) => data` という形です。唯一の違いは、context に自動的に Apollo cache が追加されていることです。ですので resolver の中で cache を使って、読み書きが可能です。

> We're not done yet! What if we wanted to perform a more complicated local data update such as adding or removing items from a list? For this situation, we'll use a local resolver. Local resolvers have the same function signature as remote resolvers (`(parent, args, context, info) => data`). The only difference is that the Apollo cache is already added to the context for you. Inside your resolver, you'll use the cache to read and write data.

では local resolver を書いていきましょう。`addOrRemoveFromCart` mutation のための resolver を定義します。

> Let's write the local resolver for the `addOrRemoveFromCart` mutation. You should place this resolver underneath the `Launch` resolver we wrote earlier.

_src/resolvers.js_

```js
export const resolvers = {
  Mutation: {
    addOrRemoveFromCart: (_, { id }, { cache }) => {
      const { cartItems } = cache.readQuery({ query: GET_CART_ITEMS });
      const data = {
        cartItems: cartItems.includes(id)
          ? cartItems.filter(i => i !== id)
          : [...cartItems, id],
      };
      cache.writeQuery({ query: GET_CART_ITEMS, data });
      return data.cartItems;
    },
  },
};
```

Resolver 内で `cache` を context から取り出すことができます。この cache に対して query を発行して cart item を取得します。cart item のデータが取得できたら次に、mutation に渡された id を用いて、cart item の list から item を取り除いたり追加したりします。そうしてできたデータを更新用の list として返します。

> In this resolver, we destructure the Apollo `cache` from the context in order to read the query that fetches cart items. Once we have our cart data, we either remove or add the cart item's `id` passed into the mutation to the list. Finally, we return the updated list from the mutation.

では `addOrRemoveFromCart` mutation をコンポーネント内で呼び出す部分を読んでみましょう。

> Let's see how we call the `addOrRemoveFromCart` mutation in a component:

_src/containers/action-button.js_

```js
import gql from 'graphql-tag';

const TOGGLE_CART = gql`
  mutation addOrRemoveFromCart($launchId: ID!) {
    addOrRemoveFromCart(id: $launchId) @client
  }
`;
```

以前と同じように、mutation に `@client` directive を加え、Apollo に対し、 remote server ではなく cache に対して mutation をすることを伝えます。

> Just like before, the only thing we need to add to our mutation is a `@client` directive to tell Apollo to resolve this mutation from the cache instead of a remote server.

これで local mutation 関連の実装は終了です。では残りの `ActionButton` コンポーネントを実装し、カートを完成させましょう。

> Now that our local mutation is complete, let's build out the rest of the `ActionButton` component so we can finish building the cart:

_src/containers/action-button.js_

```jsx
import React from 'react';
import { useMutation } from '@apollo/react-hooks';
import gql from 'graphql-tag';

import { GET_LAUNCH_DETAILS } from '../pages/launch';
import Button from '../components/button';

const CANCEL_TRIP = gql`
  mutation cancel($launchId: ID!) {
    cancelTrip(launchId: $launchId) {
      success
      message
      launches {
        id
        isBooked
      }
    }
  }
`;

export default function ActionButton({ isBooked, id, isInCart }) {
  const [mutate, { loading, error }] = useMutation(
    isBooked ? CANCEL_TRIP : TOGGLE_CART,
    {
      variables: { launchId: id },
      refetchQueries: [
        {
          query: GET_LAUNCH_DETAILS,
          variables: { launchId: id },
        },
      ]
    }
  );

  if (loading) return <p>Loading...</p>;
  if (error) return <p>An error occurred</p>;

  return (
    <div>
      <Button
        onClick={mutate}
        isBooked={isBooked}
        data-testid={'action-button'}
      >
        {isBooked
          ? 'Cancel This Trip'
          : isInCart
          ? 'Remove from Cart'
          : 'Add to Cart'}
      </Button>
    </div>
  );
}
```

この例では `isBooked` の値を用いて、どの mutation を実行するかを決めています。また remote mutation の場合と同じく、local mutation の場合にも `useMutation` hook を用いるだけです。

> In this example, we're using the `isBooked` prop passed into the component to determine which mutation we should fire. Just like remote mutations, we can pass in our local mutations to the same `useMutation` hook.

---

お疲れ様でした！これで名実ともに Apollo platform のチュートリアルの卒業です！

> Congratulations! 🎉 You've officially made it to the end of the Apollo platform tutorial. In the final section, we're going to recap what we just learned and give you guidance on what you should learn next.
