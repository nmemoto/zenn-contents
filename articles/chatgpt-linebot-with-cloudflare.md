---
title: "ChatGPT APIとCloudflareを使って過去の会話を覚えてるLINEボットを構築する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Cloudflare", "Cloudflare Workers", "queues", "d1", "chatgpt"]
published: true
---

ChatGPT APIの[Chat Completion API](https://platform.openai.com/docs/api-reference/chat)を用いて、チャットの入力に対してその回答をレスポンスで返してくれます。
このチャットの入力に過去のチャットの内容を含めることで、過去の内容を前提とした回答を行うことができますが、これを実現するには、過去のチャットの内容を永続化しておく必要があります。

ユーザーインターフェースとしてLINE(LINE Messaging API)、LINEからの処理受付とChatGPTへのリクエスト、チャット内容の永続化をCloudflareを使って、過去の会話を覚えてるLINEボットを実現することができました。
本記事では、Cloudflare側の構成について紹介します。

[[ChatGPT API][AWSサーバーレス]ChatGPT APIであなたとの会話・文脈を覚えてくれるLINEボットを作る方法まとめ](https://dev.classmethod.jp/articles/chatgpt-api-line-bot-aws-serverless/)のCloudflare版の内容になります。(ただし、この記事内で言及されているLINEプラットフォーム署名検証についてはできていません。)

Cloudflare側の構成では、LINEからの処理受付とChatGPTへのリクエストでCloudflare Workers、チャット内容の永続化でCloudflare D1を利用します。
さらに、この構成ではCloudflare Queuesを用いています。
2023/3/5現在、Cloudflare Queuesの利用には[Workers Paid Plan](https://developers.cloudflare.com/workers/platform/pricing/)が必要になりますので、参考にされる方はご注意ください。

## なぜ Cloudflare Queues を使用したか

LINE Messaging API のwebhookを利用していますが、ボットサーバーは1秒以内にレスポンスを返す必要がありそうでした。
https://developers.line.biz/ja/docs/messaging-api/receiving-messages/#check-error-reason

webhookのリクエストを受けたWorkerで直接レスポンスを返そうとすると、ChatGPT APIからのレスポンスに時間がかかる場合があるため、worker内で行おうとしているクライアント側(webhook側)からリクエストがキャンセルされる事象を確認しました。
そのためwebhookからのWorkerChatGPT APIからのレスポンスにやD1への参照・登録操作等のworkerの処理途中で処理が終了していました。

そのため、webhookからのリクエストを受けるWorkerでは、Queueに必要なデータを送りステータスコード200のレスポンスだけ処理とし、それとは別のWorkerでQueueからデータを取り出し必要な処理を行う構成としています。

[[ChatGPT API][AWSサーバーレス]ChatGPT APIであなたとの会話・文脈を覚えてくれるLINEボットを作る方法まとめ](https://dev.classmethod.jp/articles/chatgpt-api-line-bot-aws-serverless/) では、Amazon APIGateway + AWS Lambdaの構成でこのような話は出ていなかったですが、おそらくLINE−APIGateway間の接続が終了してもAPIGateway-Lambda間の接続は終了しておらずLambda側の処理が継続できたからなのではと推測しています。
([AWS Lambda Functions URL](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/lambda-urls.html)を使った構成でどうなるか、確認してみたいです。)

## LINEの設定と動作検証

LINEの設定と、その検証のためのCloudflare Workersの構成は、[Cloudflare Worker + D1 + Hono + OpenAIでLINE Botを作る](https://zenn.dev/razokulover/articles/4d0ba10083524e#%E6%9C%80%E4%BD%8E%E9%99%90%E3%81%AE%E6%8C%99%E5%8B%95%E3%82%92%E3%81%99%E3%82%8Bline-bot%E3%82%92%E4%BD%9C%E3%82%8B) を参考にしました。

上記と同じですが、後述するCloudflare Workerでは`POST /api/webhook`を受ける構成としており、webhook urlとして`[ベースURL]/api/webhook`を指定しています。

ただし、最終的なCloudflare Workersの構成ではhonoを使用しませんでした。これは、Queuesを使用するため、Producer WorkerとConsumer Workerを以下のように構成する必要があると考えたためです。

```typescript
export default {
  async fetch(req: Request, env: Environment): Promise<Response> {
    ・・・・・・
  },
  async queue(batch: MessageBatch<Error>, env: Environment): Promise<void> {
    ・・・・・・
  },
};
```
上記は[ここ](https://developers.cloudflare.com/queues/get-started/#5-create-your-consumer-worker)からの一部引用。

honoを使ってもQueueを使う構成を取れるか、知見をお持ちの方教えていただきたく。

## D1の構成

公式のGet startedが詳しいです。
https://developers.cloudflare.com/d1/get-started/

### D1の作成

下記を実行して、Cloudflare上にD1のリソースを作成します。

```shell-session
npx wrangler d1 create cloudflare-linebot-chatgpt-api-db
```

### wrangler.tomlに追加

上記で作成した内容を追加します。bindingは、Cloudflare Workers内で使用するD1のリソースにアクセスするために使用する設定になります。

```
[[ d1_databases ]]
binding = "DB"
database_name = "cloudflare-linebot-chatgpt-api-db"
database_id = "<UUID>"
```

### テーブル構成と適用

今回は下記のようにDDLを作成しました。

```sql:schema.sql
DROP TABLE IF EXISTS messages;
CREATE TABLE messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  role TEXT NOT NULL,
  content TEXT NOT NULL
);
```

下記でDDLを適用します。

```shell-session
npx wrangler d1 execute cloudflare-linebot-chatgpt-api-db --file=./schema.sql
```

## Queueの構成

公式のGet started guideが詳しいです。
https://developers.cloudflare.com/queues/get-started/

### キューの作成

wranglerを使ってキューを作成します。

```shell-session
npx wrangler queues create cloudflare-linebot-chatgpt-api-queue
```

### wrangler.tomlに追加

キューにメッセージを送るProducer Workerにキューをバインドする設定と、キューからメッセージを取り出すConsumer Workerの設定を追加します。

https://developers.cloudflare.com/queues/platform/configuration/

```toml
[[queues.producers]]
 queue = "cloudflare-linebot-chatgpt-api"
 binding = "QUEUE"

[[queues.consumers]]
 queue = "cloudflare-linebot-chatgpt-api"
 max_batch_size = 10   // キュー内のメッセージの数が10になったらメッセージを取り出す
 max_batch_timeout = 1 // キュー内のメッセージの数がmax_batch_sizeになっていなくても、1秒おきにキュー内のメッセージを確認するようにする
```

## Workersの構成

### Workersで参照している秘匿情報の設定

LINEに投稿するために必要なトークンと、ChatGPT APIにリクエストを投げるために必要なシークレットキーを登録します。
https://developers.cloudflare.com/workers/wrangler/commands/#secret 

```shell-session
npx wrangler secret put CHANNEL_ACCESS_TOKEN

npx wrangler secret put OPENAI_API_KEY
```


### Workersのソースコード

Consumer Worker(`async queue(batch: MessageBatch<Error>, env: Environment): Promise<void> {...}`)の処理は以下のようになっています。

1. キューのメッセージ(ユーザーの入力内容)を取り出す
    複数のメッセージを処理する可能性があるので、for文でぐるぐる回して以下の処理をする

1. ユーザーの入力内容をD1に登録
    ```typescript
    await env.DB.prepare(
        `insert into messages(user_id, role, content) values (?, "user", ?)`
      )
        .bind(userId, content)
        .run();
    ```
1. lineのユーザーIDでテーブルからこれまでのチャットの内容を抽出
    ```typescript
    const { results } = await env.DB.prepare(
        `select role, content from messages where user_id = ?1 order by id`
      )
        .bind(userId)
        .all<ChatGPTRequestMessage>();
    ```
1. ChatGPT APIにリクエスト
    ```typescript
    const res = await fetch("https://api.openai.com/v1/chat/completions", {
          method: "post",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${env.OPENAI_API_KEY}`,
          },
          body: JSON.stringify({
            model: "gpt-3.5-turbo",
            messages: chatGPTcontents,
          }),
        });
    const body = await res.json<ChatGPTResponse>();
    ```
1. ChatGPT APIのレスポンスをDBに登録
    ```typescript
    await env.DB.prepare(
          `insert into messages(user_id, role, content) values (?, "assistant", ?)`
        )
          .bind(userId, body.choices[0].message.content)
          .run();
    ```
1. ChatGPT APIの回答をLINE側に登録する
    ```typescript
    await fetch("https://api.line.me/v2/bot/message/reply", {
          body: JSON.stringify({
            replyToken: replyToken,
            messages: [response],
          }),
          method: "POST",
          headers: {
            Authorization: `Bearer ${accessToken}`,
            "Content-Type": "application/json",
          },
        });
    ```
Workersの処理全体は以下に記載しました。

:::details Workersのソースコード
```typescript::src/index.tsx
import { TextMessage, WebhookEvent } from "@line/bot-sdk";

type Environment = {
  DB: D1Database;
  QUEUE: Queue;
  CHANNEL_ACCESS_TOKEN: string;
  OPENAI_API_KEY: string;
};

type Role = "user" | "system" | "assistant";

type RequestBody = {
  events: WebhookEvent[];
};

type QueueData = {
  userId: string;
  content: string;
  replyToken: string;
};

type QueueMessage = {
  body: QueueData;
  timestamp: string;
  id: string;
};

type ChatGPTRequestMessage = {
  role: Role;
  content: string;
};

type ChatGPTResponse = {
  id: string;
  object: "chat.completion";
  created: number;
  model: string;
  usage: {
    prompt_token: number;
    completion_token: number;
    total_tokens: number;
  };
  choices: {
    message: {
      role: "assistant";
      content: string;
    };
    finish_reason: string;
    index: number;
  }[];
};

export default {
  async fetch(req: Request, env: Environment): Promise<Response> {
    // Request Check
    const { pathname } = new URL(req.url);
    if (pathname !== "/api/webhook") {
      return new Response("path error", { status: 400 });
    }
    const method = req.method;
    if (method.toLowerCase() !== "post") {
      return new Response("path error", { status: 400 });
    }
    // Extract From Request Body
    const data = await req.json<RequestBody>();
    const event = data.events[0];
    if (event.type !== "message" || event.message.type !== "text") {
      return new Response("body error", { status: 400 });
    }
    const { source, replyToken } = event;
    if (source.type !== "user") {
      return new Response("body error", { status: 400 });
    }
    const { userId } = source;
    const { text } = event.message;
    const queueData = {
      userId,
      content: text,
      replyToken,
    };
    await env.QUEUE.send(queueData);
    return new Response("Success!");
  },
  async queue(batch: MessageBatch<Error>, env: Environment): Promise<void> {
    let messages = JSON.stringify(batch.messages);
    const queueMessages = JSON.parse(messages) as QueueMessage[];
    for await (const message of queueMessages) {
      const { userId, content, replyToken } = message.body;
      // DBに登録する
      await env.DB.prepare(
        `insert into messages(user_id, role, content) values (?, "user", ?)`
      )
        .bind(userId, content)
        .run();
      // DBを参照する
      const { results } = await env.DB.prepare(
        `select role, content from messages where user_id = ?1 order by id`
      )
        .bind(userId)
        .all<ChatGPTRequestMessage>();
      const chatGPTcontents = results ?? [];
      try {
        const res = await fetch("https://api.openai.com/v1/chat/completions", {
          method: "post",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${env.OPENAI_API_KEY}`,
          },
          body: JSON.stringify({
            model: "gpt-3.5-turbo",
            messages: chatGPTcontents,
          }),
        });
        const body = await res.json<ChatGPTResponse>();
        // DBに登録する
        await env.DB.prepare(
          `insert into messages(user_id, role, content) values (?, "assistant", ?)`
        )
          .bind(userId, body.choices[0].message.content)
          .run();
        const accessToken: string = env.CHANNEL_ACCESS_TOKEN;
        const response: TextMessage = {
          type: "text",
          text: body.choices[0].message.content,
        };
        await fetch("https://api.line.me/v2/bot/message/reply", {
          body: JSON.stringify({
            replyToken: replyToken,
            messages: [response],
          }),
          method: "POST",
          headers: {
            Authorization: `Bearer ${accessToken}`,
            "Content-Type": "application/json",
          },
        });
      } catch (error) {
        if (error instanceof Error) {
          console.error(error);
        }
      }
    }
  },
};

```
:::

### デプロイ

以下のようにpackage.jsonに設定しておき

```json::package.json
{
  "scripts": {
    ・・・・
    "deploy": "wrangler publish src/index.ts",
    ・・・・
  },
  ・・・・
}
```

以下のコマンドで実施

```shell-session
npm run deploy
```


## リポジトリ

https://github.com/nmemoto/chatgpt-linebot-with-cloudflare

## まとめ

Cloudflare Workers/D1/QueuesとChatGPT APIのChat Completion APIを用いて、過去の会話を覚えてくれるLINEボットを作りました。

## 参考

- [[ChatGPT API][AWSサーバーレス]ChatGPT APIであなたとの会話・文脈を覚えてくれるLINEボットを作る方法まとめ](https://dev.classmethod.jp/articles/chatgpt-api-line-bot-aws-serverless/)
- [Cloudflare Worker + D1 + Hono + OpenAIでLINE Botを作る](https://zenn.dev/razokulover/articles/4d0ba10083524e)