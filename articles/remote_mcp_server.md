---
title: "Cloudflare リモートMCPサーバーを利用してAPIを叩くMCPツールを作る"
emoji: "📞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "cloudflare"]
published: false
---

## はじめに

この記事は[【LLM+RAG】自分自身と会話できるナレッジベースシステムを作ってみた](https://zenn.dev/nijigen_plot/articles/personal_knowledge_base)のナレッジベースシステムの`search` API をMCPから叩けるようにした記事です。

画像赤枠の部分について書いています。

![](/images/remote_mcp_server/architecture_mcp.png)

### できること

- MCPからsearch APIを通じてナレッジベースにアクセスし、情報を返す
- リモートMCPサーバーをホストすることで誰でもアクセスできる


## リモートMCPサーバーのホスティング

[MCP](https://modelcontextprotocol.io/docs/getting-started/intro)ですが、サーバーについては[ローカル環境で構築する](https://modelcontextprotocol.io/docs/develop/build-server)ことは簡単ですが、環境に捉われず誰でも利用できるMCPサーバーを立てたいです。

CloudflareがHTTPで接続しMCPを利用できるサービスを提供しているのでそれを使うことにしました。

以下ページに詳しい内容が載っています。

https://blog.cloudflare.com/remote-model-context-protocol-servers-mcp/

### 初期設定

Cloudflare Docsの[Build a Remoter MCP server](https://developers.cloudflare.com/agents/guides/remote-mcp-server/)に沿って作っていきます。

[対象のリポジトリ](https://github.com/nijigen-plot/personal-knowledge-base)で以下を実行します。

```bash
$ npm create cloudflare@latest -- mcp-server
```

対話形式で設定を確認されるため以下のように進めていきます。

```
├ In which directory do you want to create your application?
│ dir ./mcp-server
│
╰ What would you like to start with?
  ● Hello World example <-選択
  ○ Framework Starter
  ○ Application Starter
  ○ Template from a GitHub repo
  ◁ Go back


╰ Which template would you like to use?
  ○ Worker only
  ○ Static site
  ○ SSR / full-stack app
  ● Worker + Durable Objects <-選択
  ○ Worker + Durable Objects + Assets
  ○ Workflow
  ○ Scheduled Worker (Cron Trigger)
  ○ Queue consumer & producer Worker
  ○ API starter (OpenAPI compliant)
  ◁ Go back

╰ Which language do you want to use?
  ● TypeScript <-選択
  ○ JavaScript
  ○ Python (beta)
  ◁ Go back

┐ Installing dependencies ..


╰ Do you want to use git for version control?
  Yes / No <- 既に対象リポジトリがGit管理のためNo

╰ Do you want to deploy your application?
  Yes / No <- 後程設定してdeployしたいのでNo
```

必要なライブラリをインストールします

```bash
$ npm install agents @modelcontextprotocol/sdk zod
```

### MCPツールの実装

ナレッジベースの[search API](https://home.quark-hardcore.com/personal-knowledge-base/api/v1/docs#/search/search_documents_api_v1_search_post)を叩き、結果を返す内容を実装します。

以下のページをとても参考にしました。

https://azukiazusa.dev/blog/cloudflare-mcp-server/


```typescript:./src/MyMcp.ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { McpAgent } from 'agents/mcp';
import { z } from 'zod';

export class MyMCP extends McpAgent {
	server = new McpServer({
		name: 'MyMCP Server',
		version: '0.1.0',
	});

	async init() {
		this.server.tool(
			// ツールの名前
			'quarkgabber-knowledge-base-search',
			// ツールの説明
			'QuarkgabberナレッジベースAPI searchにアクセスしレスポンスを受け取ります。',
			// ツールの引数のスキーマ
            { query: z.string().describe('検索クエリ')},
			// ツールの実行関数
            async (args: any) => {
                const { query } = args;
                // search APIをPOSTリクエストする
                const response = await fetch('https://home.quark-hardcore.com/personal-knowledge-base/api/v1/search', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        query: query,
                        k: 10
                    })
                });

                const data = await response.json();
                // 結果を返す
                return {
                    content: [{ type: 'text', text: JSON.stringify(data, null, 2)}]
                };
        });
    }
}
```

```typescript:./src/index.ts
import { MyMCP } from './MyMcp.js';

// Durable Objects のエクスポート
export { MyMCP };

export default {
	fetch(request: Request, env: Env, ctx: ExecutionContext) {
		const url = new URL(request.url);

		// /sse エンドポイントの場合は SSE で応答する
		if (url.pathname === '/sse' || url.pathname === '/sse/message') {
			// @ts-ignore
			return MyMCP.serveSSE('/sse').fetch(request, env, ctx);
		}

		// /mcp エンドポイントの場合は Streamable HTTP で応答する
		if (url.pathname === '/mcp') {
			// @ts-ignore
			return MyMCP.serve('/mcp').fetch(request, env, ctx);
		}

		return new Response('Not found', { status: 404 });
	},
};
```

```json:wrangler.jsonc
{
	"$schema": "node_modules/wrangler/config-schema.json",
	"name": "mcp-server",
	"main": "src/index.ts",
	"compatibility_date": "2025-09-02",
	"compatibility_flags": ["nodejs_compat"],
	"migrations": [
		{
			"new_sqlite_classes": [
				"MyMCP"
			],
			"tag": "v1"
		}
	],
	"durable_objects": {
		"bindings": [
			{
				"class_name": "MyMCP",
				"name": "MCP_OBJECT"
			}
		]
	}
}

```


### 動作チェック

まずはローカル環境で確認します。

サーバーを立ち上げます。

```bash
$ npm run dev
[wrangler:info] Ready on http://localhost:8787
```


ブラウザからMCPサーバーに接続してツールを呼び出せる[MCP Inspector](https://github.com/modelcontextprotocol/inspector)を実行します。

```bash
$ npx @modelcontextprotocol/inspector@latest
```

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20160906.png)
*ブラウザからこのような画面が開く*

デフォルトでは`localhost:6274`でブラウザが開き、リクエストは`localhost:6277`でプロキシしているみたいです。


- `Transport Type -> SSE`
- `URL -> http:localhost:8787/sse`

と設定しconnectを押します。

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20160623.png)


右上の`Tools`→`List Tools`ボタンを押します。

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20161004.png)

queryに質問文を入れて、`Run Tool`を押します。

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20161059.png)

成功しました！

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20161230.png)

> `Transport Type -> SSE`
> `URL -> http://localhost:8787/sse`
> と設定しconnectを押します。

こちらの設定ですが、リモートMCPでは[SSE(Server-Sent Events)かStreamable HTTPで通信](https://developers.cloudflare.com/agents/model-context-protocol/transport/)します。


- `Transport Type -> Streamable HTTP`
- `URL -> http://localhost:8787/mcp`

でも同様に成功することを確認します。

![](/images/remote_mcp_server/20251915.png)

成功したら変更を対象リポジトリにPushしておきます。

### CloudflareにDeploy

[Cloudflare](https://www.cloudflare.com)にアカウントを作ります。

`npm run deploy`コマンドを`wrangler.jsonc`があるディレクトリで実行します。初回はOAuthブラウザ認証が入ります。

```bash
$ npm run deploy
Total Upload: 1237.91 KiB / gzip: 198.40 KiB
Worker Startup Time: 30 ms
Your Worker has access to the following bindings:
Binding                     Resource
env.MCP_OBJECT (MyMCP)      Durable Object

Uploaded mcp-server (4.35 sec)
Deployed mcp-server triggers (0.30 sec)
  https://mcp-server.quark-hardcore.workers.dev
```

Cloudflareのダッシュボード「コンピューティング」→「Workers & Pages」をみるとWorkerが作られています。

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20165846.png)

[Workers AI LLM Playground](https://playground.ai.cloudflare.com/)でDeployしたMCPサーバーを試すことができます。

`npm run deploy`で発行されたURL + /sse を「Enter MCP server URL」にいれてconnectします。(URLはCloudflare対象workerの設定タブからも確認できます。)

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20170429.png)

このMCPはsearch APIでOpenSearchから類似コンテンツを返すもので、その後の解釈はAI行うので、**AIの能力に依存**します。

Workers AI LLM Playgroundで提供されているモデルでは、あまり思ったような返事にはなりませんでした。

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20171612.png)
*GPT5やSonnet 4ではtimestampを使って答えてくれる*

## Claude DesktopからリモートMCPサーバーに接続して使う

[Claude Desktop](https://claude.ai/download)はMCPに対応しているのでそちらで使ってみます。

「設定」→「コネクタ」→「カスタムコネクタを追加」をクリック

![](/images/remote_mcp_server/202509153.png)

名前をURLを設定します。

![](/images/remote_mcp_server/202509154.png)

検索とツールでツールを有効にします。

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20172341.png)

チャットから誕生日を聞いてみます

![](/images/remote_mcp_server/スクリーンショット%202025-09-15%20172555.png)

うまくいきました！

## 参考

https://azukiazusa.dev/blog/cloudflare-mcp-server/

https://speakerdeck.com/syumai/cloudflare-workersdejin-merurimotomcphuo-yong

https://dev.classmethod.jp/articles/mcp-server-without-local-credentials-cloudflare/
