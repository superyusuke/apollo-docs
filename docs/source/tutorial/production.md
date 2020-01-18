---
title: '4. 作成した graph を production として実行する'
description: ディプロイと開発ツールについて学ぶ
---

Time to accomplish: _15 Minutes_

ここまでの作業お疲れ様でした！私たちはもう GraphQL API を Apollo を使って開発できるようになりましたね。GraphQL API は REST や SQL といった data source につなげることも、GraphQL query を投げることもできるようになりました。Graph の構築はこれで終わりですので、最後にディプロイしましょう！

> Great job for making it this far! We've already learned how to build a GraphQL API with Apollo, connect it to REST and SQL data sources, and send GraphQL queries. Now that we've completed building our graph, it's finally time to deploy it! 🎉

Apollo 製の GraphQL API はどんなクラウドサービスにディプロイすることも可能です。例えば Heroku, AWS Lambda, Netlify といった選択肢があります。また、もし [Apollo Graph Manager](https://engine.apollographql.com/) のアカウントを持っていない場合は作成してください。ここからの作業に必要になります。

> An Apollo GraphQL API can be deployed to any cloud service, such as Heroku, AWS Lambda, or Netlify. If you haven't already created an [Apollo Graph Manager](https://engine.apollographql.com/) account, you will need to sign up for one.

## Publish your schema to Graph Manager

アプリケーションをディプロイする前に、作成した schema を Apollo Graph Manager cloud service に publish しましょう。そうすることで VSCode 用のツールや schema 変更を追跡するツールが使えるようになります。npm が JavaScript packages の保存場所であるように、Apollo Graph Manager は schema 保存場所の役割も持ち、ここから直近の schema を簡単に取得することができます。

> Before we deploy our app, we need to publish our schema to the Apollo Graph Manager cloud service in order to power developer tooling like VSCode and keep track of schema changes. Just like npm is a registry for JavaScript packages, Apollo Graph Manager contains a schema registry that makes it simple to pull the most recent schema from the cloud.

実際のアプリケーションにいおいては CI ワークフローの一部にこの publish するスクリプトを組み込んだほうがいいでしょう。いまのところはターミナルで Apollo CLI を手動で実行し schema を Graph Manager に publish することとしましょう。

> In a production application, you should set up this publishing script as part of your CI workflow. For now, we will run a script in our terminal that uses the Apollo CLI to publish our schema to Graph Manager.

### Get a Graph Manager API key

まず Apollo Graph Manager API key を取得する必要があります。[Apollo Graph Manager](https://engine.apollographql.com/) に移動しログインし、サイドバーの一番上にある _New Graph_ をクリックしましょう。するとこれからする graph に名前をつけるように促されます。それを入力したら _Create Graph_ をクリックします。すると `service:` の後ろに key が表示されるはずです。この key をコピーし、これを環境変数に保存しましょう。

First, we need an Apollo Graph Manager API key. Navigate to [Apollo Graph Manager](https://engine.apollographql.com/), login, and click on _New Graph_ on the sidebar or at the top. The prompt will instruct you to name your graph. When you're finished, click _Create Graph_. You'll see a key appear prefixed by `service:`. Copy that key so we can save it as an environment variable.

環境変数に保存するにあたって重要なのは、Graph Manager API key を Git のような version control tool へ絶対にコミットしないことです。`server/` にある `.env.example` を複製して名称を `.env` へ変更します。先ほどコピーしたキーをここに複写します。（訳注：`.env` が絶対にコミットされないように .gitignore 等に処理がされていることを確認しましょう）

> Let's save our key as an environment variable. It's important to make sure we don't check our Graph Manager API key into version control. Go ahead and make a copy of the `.env.example` file located in `server/` and call it `.env`. Add your Graph Manager API key that you copied from the previous step to the file:

```
ENGINE_API_KEY=service:<your-service-name>:<hash-from-apollo-engine>
```

The entry should basically look like this:

```
ENGINE_API_KEY=service:my-service-439:E4VSTiXeFWaSSBgFWXOiSA
```

これで環境変数の `ENGINE_API_KEY` として使用する準備ができました。

> Our key is now stored under the environment variable `ENGINE_API_KEY`.

### Check and publish with the Apollo CLI

さあ Graph Manager に schema を publish しましょう！まずサーバーを立ち上げるためにターミナルウィンドウで `npm start` を実行します。その後もう一つ別のターミナルウィンドウを開いて、以下のコマンドを実行します。

> It's time to publish our schema to Graph Manager! First, start your server in one terminal window by running `npm start`. In another terminal window, run:

```bash
npx apollo service:push --endpoint=http://localhost:4000
```

> npx is a tool bundled with npm for easily running packages that are not installed globally.

このコマンドによって schema を Apollo レジストリに publish ができます。Schema がアップロードされたら、[Apollo Graph Manager](https://engine.apollographql.com/) explorer から schema を見れるようになります。さらにここから schema をダウンロードして Apollo VSCode extension のために使うこともできるようになります。

> This command publishes your schema to the Apollo registry. Once your schema is uploaded, you should be able to see your schema in the [Apollo Graph Manager](https://engine.apollographql.com/) explorer. In future steps, we will pull down our schema from Graph Manager in order to power the Apollo VSCode extension.

さらに追加の schema を publish すると、新しい schema 古い schema を比較し、壊的な変更がないかをチェックることもできます。そのためには以下のコマンドを実行します。

> For subsequent publishes, we may first want to check for any breaking changes in our new schema against the old version. In a terminal window, run:

```bash
npx apollo service:check --endpoint=http://localhost:4000
```

### What are the benefits of Graph Manager?

Schema を Apollo Graph Manager へと publish することで、本番環境で Graph API を運用するにあたって不可欠な機能が色々と使えるようになります。

> Publishing your schema to Apollo Graph Manager unlocks many features necessary for running a graph API in production. Some of these features include:

- **Schema explorer:** Graph Manager の強力な schema レジストリのおかげで、schema の全ての type と field を調べることができ、同時に各 field の稼働統計も取ることができます。この測定によって特定の field にかかるコストがわかります。それによって例えば、この field はどれくらい負荷が高いのだろうか、この field は本当に必要なのか、といった調査が可能になります。
- **Schema history:** Apollo Graph Manager schema history を使うことで新しい Graph Schema が過去の schema と比べて破壊的なところがないのかを確認することができます。新しい schema を古い schema で field ごとにヴァリデーションをすることでそれを確認します。これによって新しい schema によって client 側のどこに破壊的変更が引き起こされるのかを精査することが可能になります。
- **Performance analytics:** Graph を実行したい際のパフォーマンス測定ができます。全ての field, resolver, operation に対する詳細な情報を得ることができます。
- **Client awareness:** クライアント情報（名前とバージョン）を取得できます。

> - **Schema explorer:** With Graph Manager's powerful schema registry, you can quickly explore all the types and fields in your schema with usage statistics on each field. This metric makes you understand the cost of a field. How expensive is a field? Is a certain field in so much demand?
> - **Schema history:** Apollo Graph Manager schema history allows developers to confidently iterate a graph's schema by validating the new schema against field-level usage data from the previous schema. This empowers developers to avoid breaking changes by providing insights into which clients will be broken by a new schema.
> - **Performance analytics:** Fine-grained insights into every field, resolvers and operations of your graph's execution
> - **Client awareness:** Report client identity (name and version) to your server for insights on client activity.

正直に申し上げると、上記で説明した機能のうち、viewing specific execution traces や validating schema changes against recent operations といった機能は有料プランでしか使えません。GraphQL を使い始めた個人開発者にとってはこれらの機能は必要ないかもしれませんが、チームで開発するとなるとこれらの機能の価値が突如として認識されるはずです。くわえて、これらの有料機能と無料の Apollo VSCode といった開発ツールの連携をより強化し、よりインテリジェントなツールになるように開発してまいります。

> We also want to be transparent that the features we just described, such as viewing specific execution traces and validating schema changes against recent operations, are only available on a paid plan. Individual developers just getting started with GraphQL probably don't need these features, but they become incredibly valuable as you're working on a team. Additionally, layering these paid features on top of our free developer tools like Apollo VSCode makes them more intelligent over time.

我々 Apollo チームは、あなたが Apollo graph API の構築と運用の手助けをミッションとしています。ですから、スキーマの公開とダウンロード、Apollo client と server のようなオープンソースライブラリ、それから Apollo VSCode や Apollo DevTools といったか初者ツールについては、永続的に常に無料であり続けます。

> We're committed to helping you succeed in building and running an Apollo graph API. This is why features such as publishing and downloading schemas from the registry, our open source offerings like Apollo Client and Apollo Server, and certain developer tools like Apollo VSCode and Apollo DevTools will always be free forever.
