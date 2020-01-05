---
title: '1. Build a schema'
description: graph データの設計図を作ろう
---

graph API をつくるこの旅の最初のステップは、**schema** を作ることです。schema は、graph を通してアクセスできる全てのデータの「設計図」といってもいいでしょう。このセクションを通して Apollo を用いて graph の schema を作る方法と、その構造を探求する方法を学習します。

> The first step on our journey toward building our graph API is constructing its **schema**. You can think of a schema as a blueprint for all of the data you can access in your graph. Throughout this section, you'll learn how to build and explore your graph's schema with Apollo.

## Set up Apollo Server

Schema を書く前に、graph API のサーバーを立てる必要があります。**Apollo Server**　はプロダクションに即使える graph API の構築を助けてくれるライブラリです。これを使えばどんなデータソースにもアクセスすることができます。もちろん REST API にもデータベースにも。さらにこれをつかうことで Apollo の開発ツールともシームレスに連携することができます。

> Before we write our schema, we need to set up our graph API's server. **Apollo Server** is a library that helps you build a production-ready graph API over your data. It can connect to any data source, including REST APIs and databases, and it seamlessly integrates with Apollo developer tooling.

では開発ディレクトリのルートで必要なパッケージをインストールしていましょう。

> From the root, let's install our project's dependencies:

```bash
cd start/server && npm install
```

Apollo Server の開発を始めるのに必要なパッケージは、`apollo-server` and `graphql` です。これはすでに package.json に記録しておきましたので上記コマンドを実行すればインストールが完了します。では

> The two packages you need to get started with Apollo Server are `apollo-server` and `graphql`, which we've already installed for you. Now, let's navigate to `src/index.js` so we can create our server. Copy the code below into the file.

_src/index.js_

```js
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');

const server = new ApolloServer({ typeDefs });
```

To build our graph API, we need to import the `ApolloServer` class from `apollo-server`. We also need to import our schema from `src/schema.js`. Next, let's create a new instance of `ApolloServer` and pass our schema to the `typeDefs` property on the configuration object.

Before we can start the server, we need to write our schema first.

## Write your graph's schema

Every graph API is centered around its schema. You can think of a schema as a blueprint that describes all of your data's types and their relationships. A schema also defines what data we can fetch through queries and what data we can update through mutations. It is strongly typed, which unlocks powerful developer tooling.

Schemas are at their best when they are designed around the needs of the clients that are consuming them. Since a schema sits in between your clients and your underlying services, it serves as a perfect middle ground for frontend and backend teams to collaborate. We recommend that teams practice **Schema First Development** and agree upon the schema first before any API development begins.

Let's think about the data we will need to expose in order to build our app. Our app needs to:

- Fetch all upcoming rocket launches
- Fetch a specific launch by its ID
- Login the user
- Book launch trips if the user is logged in
- Cancel launch trips if the user is logged in

Our schema will be based on these features. In `src/schema.js`, import `gql` from Apollo Server and create a variable called `typeDefs` for your schema. Your schema will go inside the `gql` function (between the backticks in this portion: <code>gql\`\`</code>).

_src/schema.js_

```js
const { gql } = require('apollo-server');

const typeDefs = gql`
  # Your schema will go here
`;

module.exports = typeDefs;
```

### Query type

We'll start with the **Query type**, which is the entry point into our schema that describes what data we can fetch.

The language we use to write our schema is GraphQL's schema definition language (SDL). If you've used TypeScript before, the syntax will look familiar. Copy the following SDL code between the backticks where the `gql` function is invoked in  `src/schema.js`

_src/schema.js_

```graphql
type Query {
  launches: [Launch]!
  launch(id: ID!): Launch
  # Queries for the current user
  me: User
}
```

First, we define a `launches` query to fetch all upcoming rocket launches. This query returns an array of launches, which will never be null. Since all types in GraphQL are nullable by default, we need to add the `!` to indicate that our query will always return data. Next, we define a query to fetch a `launch` by its ID. This query takes an argument of `id` and returns a single launch. Finally, we will add a `me` query to fetch the current user's data. Above the `me` query is an example of a comment added to the schema.

How do we define what properties are exposed by `Launch` and `User`? For these types, we need to define a GraphQL object type.

### Object & scalar types

Let's define what the structure of `Launch` looks like by creating an **object type**.  Once again, copy the following SDL code inside the backticks where the `gql` function is invoked within  `src/schema.js`:

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

The `Launch` type has **fields** that correspond to object and scalar types. A **scalar type** is a primitive type like `ID`, `String`, `Boolean`, or `Int`. You can think of scalars as the leaves of your graph that all fields resolve to. GraphQL has many scalars built in, and you can also define [custom scalars](https://www.apollographql.com/docs/apollo-server/schema/scalars-enums/) like `Date`.

The `Mission` and `Rocket` types represent other object types. Let's define the fields on `Mission`, `Rocket`, and `User`:

_src/schema.js_

```graphql
type Rocket {
  id: ID!
  name: String
  type: String
}

type User {
  id: ID!
  email: String!
  trips: [Launch]!
}

type Mission {
  name: String
  missionPatch(size: PatchSize): String
}

enum PatchSize {
  SMALL
  LARGE
}
```

You'll notice that the field `missionPatch` takes an argument of `size`. GraphQL is flexible because any fields can contain arguments, not just queries. The `size` argument corresponds to an **enum type**, which we're defining at the bottom with `PatchSize`.

There are some other less common types you might also encounter when building your graph's schema. For a full list, you can reference this handy [cheat sheet](https://devhints.io/graphql#schema).

### Mutation type

Now, let's define the **Mutation type**. The `Mutation` type is the entry point into our graph for modifying data. Just like the `Query` type, the `Mutation` type is a special object type.

_src/schema.js_

```graphql
type Mutation {
  # if false, booking trips failed -- check errors
  bookTrips(launchIds: [ID]!): TripUpdateResponse!

  # if false, cancellation failed -- check errors
  cancelTrip(launchId: ID!): TripUpdateResponse!

  login(email: String): String # login token
}
```

Both the `bookTrips` and `cancelTrip` mutations take an argument and return a `TripUpdateResponse`. The return type for your GraphQL mutation is completely up to you, but we recommend defining a special response type to ensure a proper response is returned back to the client. In a larger project, you might abstract this type into an interface, but for now, we're going to define `TripUpdateResponse`:

_src/schema.js_

```graphql
type TripUpdateResponse {
  success: Boolean!
  message: String
  launches: [Launch]
}
```

Our mutation response type contains a success status, a corresponding message, and the launch that we updated. It's always good practice to return the data that you're updating in order for the Apollo Client cache to update automatically.

## Run your server

Now that we have scoped out our app's schema, let's run the server by calling `server.listen()`.

_src/index.js_

```js
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');

const server = new ApolloServer({ typeDefs });

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

In your terminal, run `npm start` to start your server! 🎉 Apollo Server will now be available on port 4000.

### Explore your schema

By default, Apollo Server supports [GraphQL Playground](https://www.apollographql.com/docs/apollo-server/features/graphql-playground/). The Playground is an interactive, in-browser GraphQL IDE for exploring your schema and testing your queries. Apollo Server automatically serves GraphQL Playground in development only.

The GraphQL Playground provides the ability to introspect your schema. **Introspection** is a technique used to provide detailed information about a graph's schema. To see this in action, check out the right hand side of GraphQL Playground and click on the `schema` button.

<div style="text-align:center">
  <img src="../images/schematab.png" alt="Schema button">
</div>

You can quickly have access to the documentation of a GraphQL API via the `schema` button.

<div style="text-align:center">
  <img src="../images/moredetailsonatype.png" alt="More details on a Schema Type">
</div>

That's all for building our schema. Let's move on to the next part of our tutorial.
