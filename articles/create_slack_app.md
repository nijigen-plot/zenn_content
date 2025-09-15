---
title: "@メンションで返信を行ってくれるSlackアプリを作る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["slack", "python", "ai"]
published: false
---

## はじめに

この記事は[【LLM+RAG】自分自身と会話できるナレッジベースシステムを作ってみた](https://zenn.dev/nijigen_plot/articles/personal_knowledge_base)のナレッジベースシステムの`conversation` API をSlackメンションから叩けるようにした記事です。

画像赤枠の部分について書いています。

![](/images/create_slack_app/architecture_slack.png)

### できること

@メンションで質問を投げるとスレッドで返してくれます

![](/images/create_slack_app/スクリーンショット%202025-09-15%20185225.png)

## Slack アプリ作成

[こちらのページ](https://api.slack.com/apps/)からアプリを作り、アプリ経由でAIを介してナレッジベースにアクセスできる[conversation API](https://home.quark-hardcore.com/personal-knowledge-base/api/v1/docs#/conversation/conversation_with_rag_api_v1_conversation_post)を叩き、内容をスレッドに返信で返します。

### 初期設定

[こちらのページ](https://api.slack.com/apps/)から「Create New App」をクリックし新しいアプリを作ります。

![](/images/create_slack_app/202509155.png)

`From scratch`を選択しUIから設定を行います。

[manifest](https://docs.slack.dev/app-manifests/configuring-apps-with-app-manifests/)でコードベースの作成もできるようです。

#### BOTのメンション名変更

アプリ画面左タブの「App Home」に`App Display Name`があるのでそこから変更します。

![](/images/create_slack_app/スクリーンショット%202025-09-15%20185225.png)

#### Scope設定

アプリ画面左タブの「OAuth & Permissions」の`Scopes`→`Bot Token Scopes`からBotに与えるスコープを設定します。

メンションを読み取り、メッセージを送る為のスコープを付けました。

![](/images/create_slack_app/スクリーンショット%202025-09-15%20223701.png)

#### OAuth Token発行

アプリ画面左タブの「OAuth & Permissions」の`OAuth Tokens`からBot User OAuth Tokenを発行します。

![](/images/create_slack_app/スクリーンショット%202025-09-14%20163149.png)

#### Event Subscriptions設定

メンションでイベントが発火するように設定します。

アプリ画面左タブの「Event Subscriptions」から`Enable Events`をONにします。

`Subscribe to bot events`にapp_mensionのイベントを追加します。

![](/images/create_slack_app/スクリーンショット%202025-09-15%20224553.png)

Request URL にSlackイベント用API開発の為ngrokで生成したURLを入れ検証を通します。ます。(詳細は[API作成](#api作成)で記載します。)

動作確認後、開発したAPIエンドポイントのURLを入れなおします。

## Slack アプリ追加

追加した対象ワークスペースの適当なチャンネルにアプリを追加します。

チャンネル右上人数アイコンをクリック→「アプリを追加する」から追加

![](/images/create_slack_app/202509156.png)


## API作成

メンションでイベントが発火するので、それを受け取るAPIエンドポイントを作成します。

開発時はローカル環境でAPIサーバーが起動する為、APIサーバーポートに対して[ngrok](https://ngrok.com/)を使い適当なURLを発行します。

```

```


### Event Subscriptions URL検証

検証ではRequestで送られてくるパラメータについて、`challenge`を返す形で通します。

```json:検証request body

```

```python:app.py

```


### 返信を返す部分の作成

検証後、@メンションを飛ばすと実際にイベントが送られてくるのでそこからテキストメッセージを抽出、それを質問文とし[conversation API](https://home.quark-hardcore.com/personal-knowledge-base/api/v1/docs#/conversation/conversation_with_rag_api_v1_conversation_post)を叩き返信を受け取ります。

```json:イベント発火で送られるrequest body

```

```python:app.py

```




## 参考

https://zenn.dev/katzumi/articles/introduction-of-slack-bot-with-app-manifest

https://qiita.com/hoshiume11/items/4d0e37b07bca5bbb3fb5
