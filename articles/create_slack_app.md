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

![](/images/create_slack_app/スクリーンショット%202025-09-14%20163007.png)


#### Scope設定

アプリ画面左タブの「OAuth & Permissions」の`Scopes`→`Bot Token Scopes`からBotに与えるスコープを設定します。

メンションを読み取り、メッセージを送る為のスコープを付けました。

![](/images/create_slack_app/スクリーンショット%202025-09-15%20223701.png)

#### OAuth Token発行

アプリ画面左タブの「OAuth & Permissions」の`OAuth Tokens`からBot User OAuth Tokenを発行します。

![](/images/create_slack_app/スクリーンショット%202025-09-14%20163149.png)

このTokenは`.env`ファイルに記載し、メッセージを返信する際に使います。

```txt:.env
SLACK_BOT_USER_OAUTH_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

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

```bash
$ ngrok http <port>
```


### Event Subscriptions URL検証

検証ではRequestで送られてくるパラメータについて、`challenge`を返す形で通します。

```json:検証request body
{
    "token": "",
    "challenge": "",
    "type": "url_verification"
}
```

```python:app.py(一部抜粋)
async def slack_events(request: Request, background_tasks: BackgroundTasks):
    body_str = (await request.body()).decode("utf-8")
    body = json.loads(body_str)

    ...

    # Event Subscriptions URL検証の場合はchallengeを返す
    if body.get("type") == "url_verification":
        logger.info("Slack URL verification request received")
        return body["challenge"]
```

できました！

![](/images/create_slack_app/スクリーンショット%202025-09-14%20173038.png)


### 返信を返す部分の作成

検証後、@メンションを飛ばすと実際にイベントが送られてくるのでそこからテキストメッセージを抽出、それを質問文とし[conversation API](https://home.quark-hardcore.com/personal-knowledge-base/api/v1/docs#/conversation/conversation_with_rag_api_v1_conversation_post)を叩き返信を受け取ります。

また、設定したURLはSlack以外からもリクエストできるためSlackから送られてきているかどうかの実装も必要です。

https://docs.slack.dev/authentication/verifying-requests-from-slack/

![](/images/create_slack_app/20250923.png)
*Basic Information→Signing Secretを利用*

`.env`ファイルに以下を追加し上記画像赤枠に記載されている文字列を入れておきます。

```txt:.env
SLACK_SIGNING_SECRET=xxxxxxxxxxxxxxxxxxxxxxxx
```

```json:イベント発火で送られるrequest body(一部文字削除)
{
    "token": "",
    "team_id": "",
    "api_app_id": "",
    "event": {
        "user": "",
        "type": "app_mention",
        "ts": "",
        "client_msg_id": "",
        "text": "<@U09F0U255FV> ちょりーっす",
        "team": "",
        "blocks": [
            {
                "type": "rich_text",
                "block_id": "",
                "elements": [
                    {
                        "type": "rich_text_section",
                        "elements": [
                            {
                                "type": "user",
                                "user_id": ""
                            },
                            {
                                "type": "text",
                                "text": " ちょりーっす"
                            }
                        ]
                    }
                ]
            }
        ],
        "channel": "",
        "event_ts": ""
    },
    "type": "event_callback",
    "event_id": "",
    "event_time": 1757838453,
    "authorizations": [
        {
            "enterprise_id": None,
            "team_id": "",
            "user_id": "",
            "is_bot": True,
            "is_enterprise_install": False
        }
    ],
    "is_ext_shared_channel": False,
    "event_context": ""
}
```

「Slackからきたメッセージかどうか」をSigning Secretを使って確認しつつリクエストを処理します。
また、[ドキュメント](https://docs.slack.dev/authentication/verifying-requests-from-slack/)にあるようにリクエストタイムスタンプが一定古いものについては対応しないようにします。

```python:app.py(APIリクエスト部分)
import hashlib
import hmac
import json
import time

from fastapi import BackgroundTasks, Request


@api_router.post("/slack/events", tags=["conversation"])
async def slack_events(request: Request, background_tasks: BackgroundTasks):
    body_str = (await request.body()).decode("utf-8")
    body = json.loads(body_str)
    headers = dict(request.headers)
    slack_signature = headers.get("x-slack-signature")
    slack_timestamp = headers.get("x-slack-request-timestamp", "0")
    slack_signing_secret = os.getenv("SLACK_SIGNING_SECRET")
    base_str = f"v0:{slack_timestamp}:{body_str}"
    signature = f"""v0={hmac.new(
            bytes(slack_signing_secret, "UTF-8"),
            bytes(base_str, "UTF-8"),
            hashlib.sha256
        ).hexdigest()}"""
    logger.info(f"Slackからの質問文 : {body}")

    # Event Subscriptions URL検証の場合はchallengeを返す
    if body.get("type") == "url_verification":
        logger.info("Slack URL verification request received")
        return body["challenge"]

    if slack_signing_secret is None:
        logger.error("x-slack-signatureヘッダーがありませんでした")
        return {"status": "ok"}

    if abs(time.time() - int(slack_timestamp)) > 60 * 5:
        logger.info(f"Slackリクエストのタイムスタンプが古すぎます: {slack_timestamp}")
        return {"status": "ok"}

    if (body["event"].get("type") == "app_mention") & (hmac.compare_digest(signature, slack_signature)):
        logger.info(f"Slack app mention received: {body['event'].get('text')}")
        # Slack APIは3秒以内にレスポンスを返さないと失敗扱いになるので、バックグラウンドで処理する
        background_tasks.add_task(process_slack_message, body["event"])

    return {"status": "ok"}

def process_slack_message(event_data: dict):
    """バックグラウンドでSlackメッセージを処理"""
    question = event_data.get("text")
    channel = event_data.get("channel")
    thread = event_data.get("ts")

    try:
        # conversation APIとして実装しているメソッドを内部から呼び出す
        conv_request = ConversationRequest(question=question)
        conv_response = conversation_with_rag(conv_request)
        answer = conv_response.answer

        # 返信として結果を送る
        requests.post(
            "https://slack.com/api/chat.postMessage",
            json={"channel": channel, "thread_ts": thread, "text": answer},
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {os.getenv('SLACK_BOT_USER_OAUTH_TOKEN')}",
            },
        )
    except Exception as e:
        logger.error(f"Slack処理エラー: {e}")
```

Slack アプリは[イベントを受け取って3秒以内にHTTP 200 OKで応答をしないといけない](https://docs.slack.dev/apis/events-api/#responding)ので、Fast [APIのBackgroundTasks](https://fastapi.tiangolo.com/ja/tutorial/background-tasks/)を利用して処理はバックグラウンドで行い、先にHTTP 200 OKを返すようにしておきます。

うまくいけばこれでSlackからメンションを行うと、返信が返ってきます！

![](/images/create_slack_app/スクリーンショット%202025-09-23%20130057.png)

## Deploy後のEvent Subscriptions URL修正

開発時はngrokのURLを使ってSlackの応答を試しましたが、本番環境用のURLに変更する必要があります。

Deploy後にURLを変更し、適切に検証と応答が来るかを試します。

![](/images/create_slack_app/スクリーンショット%202025-09-23%20130446.png)
*実際に使うURLに変更して挙動を確認*


## 参考

https://zenn.dev/katzumi/articles/introduction-of-slack-bot-with-app-manifest

https://qiita.com/hoshiume11/items/4d0e37b07bca5bbb3fb5
