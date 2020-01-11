---
title: '2. データソースをつなぎこむ'
description: REST や SQL data を graph に組み込む
---

Time to accomplish: _10 Minutes_

Schema の作成が終わりましたので、次にデータソースを GraphQL API につなぎこむ必要があります。GraphQL API は非常に柔軟性があるため、どんなサービス、どんなビジネスロジック、どんな REST API、データベース、その他 gRPC といったサービスを組み込むことが可能です。

> Now that we've constructed our schema, we need to hook up our data sources to our GraphQL API. GraphQL APIs are extremely flexible because you can layer them on top of any service, including any business logic, REST APIs, databases, or gRPC services.

Apollo は data source API を用いることで、これらのサービスを graph に簡単に組み込むことができます。**Apollo data source** は、特定のサービスに対する、データのフェッチ、キャッシング、データの重複排除といったロジックをカプセル化するクラスです。Apollo data sources を用いてサービスを graph API に接続することで、コードを構造化するためのベストプラクティスに則ることができます。

> Apollo makes connecting these services to your graph simple with our data source API. An **Apollo data source** is a class that encapsulates all of the data fetching logic, as well as caching and deduplication, for a particular service. By using Apollo data sources to hook up your services to your graph API, you're also following best practices for organizing your code.

ここからは REST API と SQL データベースのための data sources を構築していきます。　そしてその data sources を Apollo server に接続します。これらサービスの技術に詳しくなくても心配ありません。深い理解がなくとも、チュートリアルを進めることができます。

> In the next sections, we'll build data sources for a REST API and a SQL database and connect them to Apollo Server. Don't worry if you're not familiar with either of those technologies, you won't need to understand them deeply in order to follow the examples. 😀

## Connect a REST API

まずは [Space-X v2 REST API](https://github.com/r-spacex/SpaceX-API) を graph に追加しましょう。そのために必要な `apollo-datasource-rest` パッケージをインストールしてください。

> First, let's connect the [Space-X v2 REST API](https://github.com/r-spacex/SpaceX-API) to our graph. To get started, install the `apollo-datasource-rest` package:

```bash
npm install apollo-datasource-rest --save
```

このパッケージが提供する `RESTDataSource` クラスは、REST API からデータをフェッチする役割を担当します。REST API のための data source を構築するためには `RESTDataSource` クラスを extend し、`this.baseURL` を定義します。

> This package exposes the `RESTDataSource` class that is responsible for fetching data from a REST API. To build a data source for a REST API, extend the `RESTDataSource` class and define `this.baseURL`.

今回は `baseURL` に `https://api.spacexdata.com/v2/` を設定します。では `src/datasources/launch.js` に以下のコードをコピーアンドペーストして `LaunchAPI` を作成しましょう。

> In our example, the `baseURL` for our API is `https://api.spacexdata.com/v2/`. Let's create our `LaunchAPI` data source by adding the code below to `src/datasources/launch.js`:

_src/datasources/launch.js_

```js
const { RESTDataSource } = require('apollo-datasource-rest');

class LaunchAPI extends RESTDataSource {
  constructor() {
    super();
    this.baseURL = 'https://api.spacexdata.com/v2/';
  }
}

module.exports = LaunchAPI;
```

Apollo `RESTDataSource` はなんの設定をしなくても、REST リソースからのレスポンスをインメモリにキャッシュする機能を提供します。この機能を **partial query caching** と呼んでいます。この機能がいいのは、REST API が提供するキャッシングロジックをそのまま再利用できる点です。（訳注：英語的にはそう書いてあるが、技術的には意味がよくわからない。正確には次に添付される記事を読んでいただきたい。）Partial query caching に興味がある場合には次の [our blog post](https://blog.apollographql.com/easy-and-performant-graphql-over-rest-e02796993b2b) をチェックしてください。

> The Apollo `RESTDataSource` also sets up an in-memory cache that caches responses from our REST resources with no additional setup. We call this **partial query caching**. What's great about this cache is that you can reuse existing caching logic that your REST API exposes. If you're curious to learn more about partial query caching with Apollo data sources, please check out [our blog post](https://blog.apollographql.com/easy-and-performant-graphql-over-rest-e02796993b2b).

### Write data fetching methods

次に `LaunchAPI` へのメソッドを追加します。これは graph API へ、対応する query が投げられた際に実行し、値をフェッチするためのメソッドです。既に定義した Schema で、全ての発射予定を取得する query が定義されていましたので、これに対応する `getAllLaunches` というメソッドを `LaunchAPI` クラスに追加しましょう。

> The next step is to add methods to the `LaunchAPI` data source that correspond to the queries our graph API needs to fetch. According to our schema, we'll need a method to get all of the launches. Let's add a `getAllLaunches` method to our `LaunchAPI` class now:

_src/datasources/launch.js_

```js
async getAllLaunches() {
  const response = await this.get('launches');
  return Array.isArray(response)
    ? response.map(launch => this.launchReducer(launch))
    : [];
}
```

Apollo REST data sources には、HTTP が実行可能な `GET` や `POST`　に対応するヘルパーメソッドが用意されています。例えば上記コード `this.get('launches')` を実行すると `GET` request を `https://api.spacexdata.com/v2/launches`　に対して行います。取得した結果は `response` 変数に保存されています。その後 `getAllLaunches` メソッドは取得した結果を `this.launchReducer` を使って必要な形状に変換します。もし取得した結果に全く発射予定がない場合には空の配列を返します。

> The Apollo REST data sources have helper methods that correspond to HTTP verbs like `GET` and `POST`. In the code above, `this.get('launches')`, makes a `GET` request to `https://api.spacexdata.com/v2/launches` and stores the returned launches in the `response` variable. Then, the `getAllLaunches` method maps over the launches and transforms the response from our REST endpoint with `this.launchReducer`. If there are no launches, an empty array is returned.

`launchReducer` メソッドはまだ書いていませんでしたのでこれを作りましょう。このメソッドは取得した launch を schema が要求する形状に変えるためのメソッドです。このように値をフェッチするメソッドとビジネスロジックを分離する方法を推奨します。ではまずは schema で定義した `Launch` がどのようなものであったか思い出すことにしましょう。

> Now, we need to write our `launchReducer` method in order to transform our launch data into the shape our schema expects. We recommend this approach in order to decouple your graph API from business logic specific to your REST API. First, let's recall what our `Launch` type looks like in our schema. You don't have to copy this code:

_src/schema.js_

```graphql
type Launch {
  id: ID!
  site: String
  mission: Mission
  rocket: Rocket
  isBooked: Boolean!
}
```

では schema を確認できたので `launchReducer` 関数を書いていきます。以下のコードを `LaunchAPI` に複写してください。

> Next, let's write a `launchReducer` function to transform the data into that shape. Copy the following code into your `LaunchAPI` class:

_src/datasources/launch.js_

```js
launchReducer(launch) {
  return {
    id: launch.flight_number || 0,
    cursor: `${launch.launch_date_unix}`,
    site: launch.launch_site && launch.launch_site.site_name,
    mission: {
      name: launch.mission_name,
      missionPatchSmall: launch.links.mission_patch_small,
      missionPatchLarge: launch.links.mission_patch,
    },
    rocket: {
      id: launch.rocket.rocket_id,
      name: launch.rocket.rocket_name,
      type: launch.rocket.rocket_type,
    },
  };
}
```

このように API からのフェッチとその値の整形を別のメソッドに分けることで、整形する値の形式が変更された場合にも、`getAllLaunches` へ変更を加える必要がなくなりますし、また `getAllLaunches` が簡潔になります。また `LaunchAPI` のテストも容易になります。テストの件に関してはまた後ほど取り上げます。

> With the above changes, we can easily make changes to the `launchReducer` method while the `getAllLaunches` method stays lean and concise. The `launchReducer` method also makes testing the `LaunchAPI` data source class easier, which we'll cover later.

では次は発射予定を ID で指定して取得する機能を作って行きましょう。ここでは `getLaunchById` と `getLaunchesByIds` の二つのメソッドを `LaunchAPI` クラスに追加します。

> Next, let's take care of fetching a specific launch by its ID. Let's add two methods, `getLaunchById`, and `getLaunchesByIds` to the `LaunchAPI` class.

_src/datasources/launch.js_

```js
async getLaunchById({ launchId }) {
  const response = await this.get('launches', { flight_number: launchId });
  return this.launchReducer(response[0]);
}

getLaunchesByIds({ launchIds }) {
  return Promise.all(
    launchIds.map(launchId => this.getLaunchById({ launchId })),
  );
}
```

`getLaunchById` メソッドは、単一のフライト番号を引数として受け取り特定の単一の発射予定を返すメソッドです。それに対して `getLaunchesByIds` は launchID を複数受け取って、複数の発射予定を返すメソッドです。

> The `getLaunchById` method takes in a flight number and returns the data for a particular launch, while `getLaunchesByIds` returns several launches based on their respective `launchIds`.

REST API へのつなぎこみができたので、次はデータベースを接続してみましょう。

> Now that we've connected our REST API successfully, let's connect our database!

## Connect a database

今まで使ってきた REST API はデータの読み取りしかできないものでしたので、ユーザー情報を保存したり読み出したりするためのデータベースを用意する必要があります。このチュートリアルでは SQLite をデータベースとして使用することとします。そして OR Mapper として Sequelize を使用します。`package.json` に既にこれらの機能のためのパッケージが書かれているので、このチュートリアルの序盤に実行した `npm install` によって既にインストールされています。またこのセクションには SQL のためのコードが含まれていますが、Apollo data sources を理解するためには、これら SQL のためのコードを理解する必要はありません。これらの機能に関するコードは既に `src/datasources/user.js` に `UserAPI` data source として用意しておきました。そのファイルに移動してから以下の説明を読んで概念を理解してください。

> Our REST API is read-only, so we need to connect our graph API to a database for saving and fetching user data. This tutorial uses SQLite for our SQL database, and Sequelize for our ORM. Our `package.json` already included these packages, thus they were installed in the first part of this tutorial with `npm install`. Also, since this section contains some SQL-specific code that isn't necessary to understanding Apollo data sources, we've already built a `UserAPI` data source for you in `src/datasources/user.js`. Please navigate to that file so we can explain the overall concepts.

### Build a custom data source

Apollo は SQL data source を使用するための便利機能は提供されていません。（もしあなたがそういう機能を作ってくれるのであれば歓迎しています。）なのでデータベースのためのカスタムデータソースを汎用 Apollo data source class を extend して自前で作成していきます。そのためには `apollo-datasource` パッケージを使用します。

> Apollo doesn't have support for a SQL data source yet (although we'd love to help guide you if you're interested in contributing), so we will need to create a custom data source for our database by extending the generic Apollo data source class. You can create your own with the `apollo-datasource` package.

次のリストが自前でデータソースを作る場合の考え方です。

> Here are some of the core concepts for creating your own data source:

- `initialize` メソッドについて: このクラスに設定オプションを渡したい場合には、このメソッドを実装します。今回は graph API の context を使用するためにこのメソッドの実装が必要です。（訳注：今回の context にはユーザーからのリクエストの内容が含まれる。例えば header の内容等。今回はこれを参照して DB のユーザーを特定したりするために、この context をこのクラス内から参照する必要があるため、このメソッドを実装している。）
- `this.context` メソッドについて: A graph API の context は全ての GraphQL resolver で共有されているオブジェクトです。context については次のセクションで詳しく説明するつもりですが、今の所は context はユーザー情報を取得するために便利だという理解をしておいてください。
- Caching について: REST data source には最初からキャッシュ機能がありますが、ここで使っている汎用 data source にはありません。ですが [our cache primitives](https://www.apollographql.com/docs/apollo-server/features/data-sources/#using-memcached-redis-as-a-cache-storage-backend) を使って自前のキャッシュを実装することはできます。

> - The `initialize` method: You'll need to implement this method if you want to pass in any configuration options to your class. Here, we're using this method to access our graph API's context.
> - `this.context`: A graph API's context is an object that's shared among every resolver in a GraphQL request. We're going to explain this in more detail in the next section. Right now, all you need to know is that the context is useful for storing user information.
> - Caching: While the REST data source comes with its own built in cache, the generic data source does not. You can use [our cache primitives](https://www.apollographql.com/docs/apollo-server/features/data-sources/#using-memcached-redis-as-a-cache-storage-backend) to build your own, however!

では具体的に `src/datasources/user.js` に作成されているコードをみていきましょう。ここに書かれたメソッドによってデータベースからデータを取得したり、データを更新したりします。これらのメソッドは次のセクションで実行することになるので、このコードを参照しながら次のセクションを読んでください。

> Let's go over some of the methods we created in `src/datasources/user.js` to fetch and update data in our database. You will want to reference these in the next section:

- `findOrCreateUser({ email })`: 与えられた email を用いてデータベースから user を探すか、もしくは作成します。
- `bookTrips({ launchIds })`: `launchIds` の配列を受け取って、それをログイン中のユーザーのために予約します。
- `cancelTrip({ launchId })`: `launchId` を受け取ってログイン中ユーザーのためにそれをキャンセルします。
- `getLaunchIdsByUser()`: ログイン中のユーザーの全ての発射予約を表示します。
- `isBookedOnLaunch({ launchId })`: ログイン中のユーザーが特定の ID の発射を予約しているかどうかを返します。

> - `findOrCreateUser({ email })`: Finds or creates a user with a given `email` in the database
> - `bookTrips({ launchIds })`: Takes an object with an array of `launchIds` and books them for the logged in user
> - `cancelTrip({ launchId })`: Takes an object with a `launchId` and cancels that launch for the logged in user
> - `getLaunchIdsByUser()`: Returns all booked launches for the logged in user
> - `isBookedOnLaunch({ launchId })`: Determines whether the logged in user booked a certain launch

## Add data sources to Apollo Server

`LaunchAPI` data source を構築し REST API と接続をしました。また `UserAPI` data source を構築し SQL データベースに接続しました。次は graph API にこれらを組み込みます。

> Now that we've built our `LaunchAPI` data source to connect our REST API and our `UserAPI` data source to connect our SQL database, we need to add them to our graph API.

data source を graph API に組み込むのは簡単です。`dataSources` プロパティを `ApolloServer` インスタンスを作成する際のオプション用オブジェクトに追加し、そのプロパティにインスタンス化した data source を含むオブジェクトを返す「関数」を与えます。では `src/index.js` にそのためのコードを加えましょう。

> Adding our data sources is simple. Just create a `dataSources` property on your `ApolloServer` that corresponds to a function that returns an object with your instantiated data sources. Let's see what that looks like by navigating to `src/index.js` and adding the code below:

_src/index.js_

```js{3,5,6,8,12-15}
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');
const { createStore } = require('./utils');

const LaunchAPI = require('./datasources/launch');
const UserAPI = require('./datasources/user');

const store = createStore();

const server = new ApolloServer({
  typeDefs,
  dataSources: () => ({
    launchAPI: new LaunchAPI(),
    userAPI: new UserAPI({ store })
  })
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

`createStore` 関数はデータベースのセットアップのために必要ですので読み込みます。また `LaunchAPI` と `UserAPI` も読み込みます。そして `createStore` を使ってデータベースを作成します。最後に `dataSources` 関数を ApolloSever に与え、`LaunchAPI` と `UserAPI` を graph に追加します。その際に `UserAPI` には先ほど作成したデータベースを渡します。

> First, we import our `createStore` function to set up our database, as well as our data sources: `LaunchAPI` and `UserAPI`. Then, we create our database by calling `createStore`. Finally, we add the `dataSources` function to our `ApolloServer` to connect `LaunchAPI` and `UserAPI` to our graph. We also pass in our database we created to the `UserAPI` data source.

`this.context` をデータソースで使用している場合には、`dataSources` 関数の中で新しいインスタンスを作成するようにし、単一のインスタンスを共有しないことがとても重要です。そうしないと、`initialize` が他のユーザーによって実行された際に、`this.context` が別のものに上書きされてしまうからです。

> If you use `this.context` in your datasource, it's critical to create a new instance in the `dataSources` function and to not share a single instance. Otherwise, `initialize` may be called during the execution of asynchronous code for a specific user, and replace the  `this.context` by the context of another user.

これで Apollo Server に data source を組み込むことができましたので、次は resolver からこれらのデータソースを呼び出す方法を学びます。

> Now that we've hooked up our data sources to Apollo Server, it's time to move on to the next section and learn how to call our data sources from within our resolvers.
