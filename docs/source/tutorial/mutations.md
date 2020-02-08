---
title: '7. Mutaion を用いてデータを更新する'
description: useMutation hook を使ってデータを更新する
---

Time to accomplish: _12 Minutes_

Apollo client を用いた graph API データの更新は、関数を実行するのと同じくらいシンプルにおこなえます。また Apollo Client のキャッシュ機能は非常に優秀で、大抵の場合は更新された値を自動的に追従します。このセクションでは `useMutation` を用いたユーザーにログインをさせる機能を実装していきます。

> With Apollo Client, updating data from a graph API is as simple as calling a function. Additionally, the Apollo Client cache is smart enough to automatically update in most cases. In this section, we'll learn how to use the `useMutation` hook to login a user.

## What is the useMutation hook?

`useMutation` は Apollo アプリケーションを構築する重要な要素の一つです。これはReact の [Hooks API](https://reactjs.org/docs/hooks-intro.html) に則って作られており、GraphQL mutation を実行する関数を取得することができます。さらに、ロードしているかどうか、完了したかどうか、エラーが発生したかどうか、といった値を取得することもできます。

> The `useMutation` hook is another important building block in an Apollo app. It leverages React's [Hooks API](https://reactjs.org/docs/hooks-intro.html) to provide a function to execute a GraphQL mutation. Additionally, it tracks the loading, completion, and error state of that mutation.

`useMutation` を使ってデータを更新するやり方は、`useQuery` を使って値をフェッチするやり方とほとんど同じです。大きな違いは `useMutation` の返り値は tuple であり、その最初の要素が **mutate function** である点です。この関数を実行することで mutation を実行できます。tuple の二番目の要素には「結果」オブジェクトが返ってきます。この「結果」オブジェクトには、loading と error の状態、それから mutation で返ってくる value が含まれています。では例をみてみましょう。

> Updating data with a `useMutation` hook from `@apollo/react-hooks` is very similar to fetching data with a `useQuery` hook. The main difference is that the first value in the `useMutation` result tuple is a **mutate function** that actually triggers the mutation when it is called. The second value in the result tuple is a result object that contains loading and error state, as well as the return value from the mutation. Let's see an example:

## Update data with useMutation

最初のステップは GraphQL mutation を定義することです。`src/pages/login.js` に移動して以下のコードを複写し、ログイン画面を作っていきましょう。

> The first step is defining our GraphQL mutation. To start, navigate to `src/pages/login.js` and copy the code below so we can start building out the login screen:

_src/pages/login.js_

```js
import React from 'react';
import { useApolloClient, useMutation } from '@apollo/react-hooks';
import gql from 'graphql-tag';

import { LoginForm, Loading } from '../components';

const LOGIN_USER = gql`
  mutation login($email: String!) {
    login(email: $email)
  }
`;
```

以前と同じように `gql` 関数で GraphQL mutation を囲み、AST へとパースします。また次のセクションで使うことになるコンポーネントもいくつかインポートしています。では次にこの mutation を `useMutation` hook に渡すことで、コンポーネントに紐付けましょう。

> Just like before, we're using the `gql` function to wrap our GraphQL mutation so it can be parsed into an AST. We're also importing some components that we'll use in the next steps. Now, let's bind this mutation to our component by passing it to the `useMutation` hook:

_src/pages/login.js_

```jsx
export default function Login() {
  const [login, { data }] = useMutation(LOGIN_USER);
  return <LoginForm login={login} />;
}
```

こうすることで `useMutation` は mutate function (`login`) を返します。それからここでは data オブジェクトを、tuple の二番目の要素から分解して取得しています。最後に取得した mutate function を `LoginForm` コンポーネントに渡しています。

> Our `useMutation` hook returns a mutate function (`login`) and the data object returned from the mutation that we destructure from the tuple. Finally, we pass our login function to the `LoginForm` component.



> To create a better experience for our users, we want to persist the login between sessions. In order to do that, we need to save our login token to `localStorage`. Let's learn how we can use the `onCompleted` handler of `useMutation` to persist our login:

### Expose Apollo Client with useApolloClient

React Apollo の主要な機能の一つに、`ApolloClient` instance を React の context へと注入する機能があります。`useApolloClient` を使って `ApolloClient` にアクセスし、`@apollo/react-hooks` コンポーネントでは提供されていないメソッドを直に実行することが可能です。

> One of the main functions of React Apollo is that it puts your `ApolloClient` instance on React's context. Sometimes, we need to access the `ApolloClient` instance to directly call a method that isn't exposed by the `@apollo/react-hooks` helper components. The `useApolloClient` hook can help us access the client.

では `useApolloClient` を実行して client instance を取得しましょう。そして `onCompleted` コールバックを `useMutation` に渡します。ここで渡したコールバックは mutation が完了した場合に、その返り値を用いて一度だけ実行されます。

> Let's call `useApolloClient` to get the currently configured client instance. Next, we want to pass an `onCompleted` callback to `useMutation` that will be called once the mutation is complete with its return value. This callback is where we will save the login token to `localStorage`.

In our `onCompleted` handler, we also call `client.writeData` to write local data to the Apollo cache indicating that the user is logged in. This is an example of a **direct write** that we'll explore further in the next section on local state management.

_src/pages/login.js_

```jsx{2,6-9}
export default function Login() {
  const client = useApolloClient();
  const [login, { loading, error }] = useMutation(
    LOGIN_USER,
    {
      onCompleted({ login }) {
        localStorage.setItem('token', login);
        client.writeData({ data: { isLoggedIn: true } });
      }
    }
  );

  if (loading) return <Loading />;
  if (error) return <p>An error occurred</p>;

  return <LoginForm login={login} />;
}
```

### Attach authorization headers to the request

We're almost done completing our login feature! Before we do, we need to attach our token to the GraphQL request's headers so our server can authorize the user. To do this, navigate to `src/index.js` where we create our `ApolloClient` and replace the code below for the constructor:

_src/index.js_

```js
const client = new ApolloClient({
  cache,
  link: new HttpLink({
    uri: 'http://localhost:4000/graphql',
    headers: { // highlight-line
      authorization: localStorage.getItem('token'), // highlight-line
    },
  }),
});

cache.writeData({
  data: {
    isLoggedIn: !!localStorage.getItem('token'),
    cartItems: [],
  },
});
```

Specifying the `headers` option on `HttpLink` allows us to read the token from `localStorage` and attach it to the request's headers each time a GraphQL operation is made.

In the next section, we'll add the `<Login>` form to the user interface. For that, we need to learn how Apollo allows us to manage local state in our app.
