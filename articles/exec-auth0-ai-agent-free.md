---
title: "10ドルで従量課金にビクビクせずAuth0 for AI Agentsを試す"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Auth0","OpenRouter","zennfes2025infra"]
published: true
---

## はじめに
先日Auth0から、Auth0 for AI Agentsでました。
早速試そうと思い、クイックスタートを見てサンプルコードを確認しました。
https://auth0.com/ai/docs/get-started/overview
ただ、コードを見る限り実行するには、LLMモデルのAPIキーが必要そうでした。
従量課金怖い勢には中々試すハードルが高いです。
なので、10$である程度試すことをできる環境を作るためのTipsを共有します。
なお、サンプルアプリケーションの対象はTypescriptのみで、Vercel AIとLangchainとなっています。
ただ、手順は簡単なので、Pythonでもサクッと搭載はできると思います。
## 手順
手順を簡単に記載します。
なお、この手順は以下のクイックスタートを前提にしていますが、他のクイックスタートでも似たようなコードがあるので、応用は効くと思います。
https://auth0.com/ai/docs/get-started/call-your-apis-on-users-behalf
- 以下のURLを参考に、[Open Router](https://openrouter.ai/)の設定を行います。
https://zenn.dev/asap/articles/5cda4576fbe7cb
- サンプルコードの.env.exampleに以下の変数を追加します。
```
OPENAI_MODEL="google/gemini-2.0-flash-exp:free"
OPENAI_BASE_URL="https://openrouter.ai/api/v1"
```
- Vercel AIを使ったサンプルコードの場合は、src/app/api/chat/route.tsのコードを以下のように変更します。
```typescript:route.ts
import { NextRequest } from 'next/server';
import { streamText, UIMessage, createUIMessageStream, createUIMessageStreamResponse, convertToModelMessages, stepCountIs } from 'ai';
import { createOpenAI } from '@ai-sdk/openai';
import { setAIContext } from '@auth0/ai-vercel';

const date = new Date().toISOString();

const AGENT_SYSTEM_TEMPLATE = `You are a personal assistant named Assistant0. You are a helpful assistant that can answer questions and help with tasks. You have access to a set of tools, use the tools as needed to answer the user's question. Render the email body as a markdown block, do not wrap it in code blocks. Today is ${date}.`;

// 外部からURLを渡せる形で定義
const openai = createOpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  baseURL: process.env.OPENAI_BASE_URL, 
});

export async function POST(req: NextRequest) {
  const { id, messages }: { id: string; messages: Array<UIMessage> } = await req.json();
  setAIContext({ threadID: id });
  const tools = {};

  const stream = createUIMessageStream({
    originalMessages: messages,
    execute: async ({ writer }) => {
      const result = streamText({
	    // モデル名を外部から渡せる形で実行させる
        model: openai(process.env.OPENAI_MODEL_NAME || 'google/gemini-2.0-flash-exp:free'),
        system: AGENT_SYSTEM_TEMPLATE,
        messages: convertToModelMessages(messages),
        stopWhen: stepCountIs(5),
        tools,
      });

      writer.merge(
        result.toUIMessageStream({
          sendReasoning: true,
        })
      );
    },
    onError: (err: any) => {
      console.log(err);
      return `An error occurred! ${err.message}`;
    },
  });

  return createUIMessageStreamResponse({ stream });
}
```
- Langchainを使ったサンプルコードの場合は、src/lib/agent.tsを以下のように変更します。
```typescript:agent.ts
import { createReactAgent, ToolNode } from '@langchain/langgraph/prebuilt';
import { ChatOpenAI } from '@langchain/openai';
import { InMemoryStore, MemorySaver } from '@langchain/langgraph';
import { Calculator } from '@langchain/community/tools/calculator';


const date = new Date().toISOString();

const AGENT_SYSTEM_TEMPLATE = `You are a personal assistant named Assistant0. You are a helpful assistant that can answer questions and help with tasks. You have access to a set of tools, use the tools as needed to answer the user's question. Render the email body as a markdown block, do not wrap it in code blocks. Today is ${date}.`;

// 外部からモデル名やURLを渡せる形で定義
const llm = new ChatOpenAI({
  model: process.env.OPENAI_MODEL || 'gpt-4o-mini',
  temperature: 0,
  configuration: process.env.OPENAI_BASE_URL ? {
    baseURL: process.env.OPENAI_BASE_URL,

  } : undefined,
});


const tools = [
  new Calculator(),
];

const checkpointer = new MemorySaver();
const store = new InMemoryStore();

export const agent = createReactAgent({
  llm,
  tools: new ToolNode(tools, {
    handleToolErrors: false,
  }),
  prompt: AGENT_SYSTEM_TEMPLATE,
  store,
  checkpointer,
});
```
あとはドキュメントに従って、セットアップすればアプリを動かせるようになります。
やっていることは、モデル狙い撃ちで定義していたものを、外部から設定が与えられる処理に置き換えるということです。
これによって、モデルの定義やURLをOpen Routerのものにした上で、チャットを実行させることができます。
## 諸注意
手順は以上でとても単純です。
ただ、一応注意点だけ記載しておきます。
### 一応無料でもできるが、10ドル払うのが無難
Open Routerは無料モデルがあるので、やろうと思えば完全無料でも実行できます。
ただ、レート制限がめちゃくちゃ早いので、10ドルだけ払っておいて、Open Routerの枠を増やした方が、ストレスなく検証できます。
1500円のみなら、個人的には許容かなと思います。
### 使用する無料モデルについて
使用するモデルは無料でも動きますが、サンプルコードの実装の兼ね合いでtools機能が使えないものしか実行できません。
なので、無料モデルを探す時はtools機能があるかをチェックしてください。
今回示したサンプルモデルは対応しています。
また、別のモデルが良い場合は、以下のURLを開くと無料かつtools機能があるモデルの検索を行います。
https://openrouter.ai/models?fmt=cards&max_price=0&supported_parameters=tools
以上です。
これからどんどん活用していこうと思います！