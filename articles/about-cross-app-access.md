---
title: "Cross App Accessをざっくりと解説して、Oktaで試してみる"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Okta", "CrossAppAccess", "XAA", "MCP"]
published: true
---
この記事は[Digital Identity技術勉強会 #iddance Advent Calendar 2025](https://qiita.com/advent-calendar/2025/iddance)の15日目の記事です。
https://qiita.com/advent-calendar/2025/iddance

## はじめに
現在AIエージェントをはじめとした、Non-HumanのID管理というのが求められています。
その中で、認可管理をエンタープライズで行うための仕様が今議論されています。
それが、[Cross App Access](https://oauth.net/cross-app-access/)(XAA)です。
Cross App Accessを用いることで、権限の管理をOktaなどのIdPで行うことが可能になります。
今回はこのCross App Accessの概要と内部技術について解説します。
主に以下の人を対象にしています。
- OAuthやOpenID Connectなどは理解していてかつ、Cross App Accessの雰囲気をつかみたい人

逆にこの記事は以下の内容を期待する人の需要を満たすことはできません。
- Cross App Accessの仕様を正確に理解したい人
- Cross App Accessを実務に活かしたいと思っている人
- 「OAuth? OpenID Connect? SAML? なんだそれ？」という状態だけどCross App Accessを理解したい人

もし対象の方であれば、続きを読んでいただけますと幸いです。
## Cross App Accessって何？どんな使い方ができるの？
### Cross App Accessの目的
ここでは、Cross App Accessが何のために存在するかを確認します。
主に以下の二つの目的があると考えています。
- 企業で使っているIdPで中央集権的にアプリの接続情報を制御、可視化
- OAuthの同意疲れ回避

それぞれ確認していきます。
#### 目的①：企業で使っているIdPで中央集権的にアプリの接続情報を制御、可視化
:::details 一文要約
企業でアプリ接続をユーザーの負荷を下げつつ、可視化や制御することが目的です。
:::
まず一つ目の目的について確認します。
例として以下のような、ClaudeとMCPが様々接続できるケースを考えます。
![](/images/about-cross-app-access/agent-and-mcp.png)
[# MCP（Model Context Protocol）の紹介 – 現代のAIアシスタントのための新しい共通プロトコル](https://josysnavi.jp/2025/introduction-to-mcp) より引用
人間ではないClaudeが複数のMCPサーバーと接続していると捉えてください。
この場合、ITの管理者がユーザーごとに利用可能MCPサーバーを設定する方法を考えてみてください。
以下のようなフローであれば、一応管理ができるかと思います。
1. 管理者が利用可能なMCPをエクセルにまとめる
2. ユーザーにはそのエクセルを参照する
3. 自身が利用可能なMCPを確認し、接続する

しかし、このフローはユーザーがきちんとエクセルを確認してくれるというユーザーの善意に依存しています。
これで適切な管理ができているとするのは少々心許ないです。
ユーザーの善意だけでなく、システムとしてきちんと権限を割り当てることが必要だからです。
この問題を解決するために、Cross App Accessがあります。
Cross App Accessに基づいていれば、ログイン情報を使うことでMCP接続に必要な権限を取得できます。
権限を取得するイメージは以下の通りです。
1. ユーザーがあるMCPの接続を管理者に要求する
2. 管理者は要求してきたユーザー情報と利用可能なMCPをエクセルで参照する
3. 対象のMCPが利用可能リストに含まれていれば、利用可能の許可証を渡す
4. ユーザーがMCPに許可証を渡し、MCPは利用可能券を発行する。
5. MCPとのやり取りは4の利用可能券を渡すことで行う

フローの中身については後ほど見ていきますので、一旦は上記のような処理をシステム的に行っているイメージでいてください。
システム的に行うので、ユーザーは一度ログインするだけで権限の割り当てができます。
そして、管理者もユーザーやグループに合わせて権限を付与するだけで、管理ができます。
ログが残るので、権限の把握だけでなく、権限が実際に割り当てられたタイミングも把握できます。
以上のような、管理者が管理できる形でアプリ接続を構築するのがCross App Accessの目的の一つとなります。
#### 目的②：OAuthの同意疲れ解消
:::details 一文要約
ユーザーに都度同意を求めないようにして、快適かつ安全な接続を可能にするのが目的です。
:::
次に同意疲れの解消について確認します。
同意疲れとはアプリの連携を許可する作業が多く、連携の許可に疲れてしまうことです。
この同意疲れは以下の問題を孕んでいます。
- 同意する作業が多く、ユーザーにストレスを与える
- 付与する権限をきちんと確認しないまま許可を与えてしまう

最初の欠点ももちろん問題ですが、二つ目の問題はより多くの危険があります。
過剰すぎる権限を与えることに気づきにくくなるためです。
過剰すぎる権限は意図しない情報へのアクセスを許可し、最悪の場合データが悪用されてしまいます。
本来同意画面というのは、そういった過剰な権限の付与を未然に防ぐためにあります。
しかし、同意疲れが起きているとチェックをしなくなるので、防ぐことが難しくなります。
上記を体感するために、以下のアプリを作ったので試してみてください。
https://sample-many-const-app.maronn.workers.dev/
このアプリはAIとのチャットとMCPの接続を疑似的に再現したものです。
テキストエリアの内容を送信すると、複数のMCPと接続し作業を行います。
MCPと接続する時同意画面を表示します。
全てのMCPに対して、同意、拒否をすると回答が返ってきます。
以上がアプリの流れです。
それでは、実際に試してみてください。
https://sample-many-const-app.maronn.workers.dev/
サンプルアプリくらいの連携数であればギリギリ許容できるかもしれません。
しかし、連携数がもっと増えたり、同意する場面が多岐にわたるとげんなりします。
付与する権限についてはどうでしょうか。
特に違和感なく権限を与えたでしょうか。
実は一部過剰な権限を付与するようになっていました。
それが、以下3つ目の同意画面における権限です。
![](/images/about-cross-app-access/consent-screen.png)
最後の権限「参加者が持つ全ての個人情報へのアクセス」は中々危険です。
MCP経由で住所などのプライバシーに関わる内容が取得できてしまいます。
以上が同意疲れ問題についてでした。
Cross App Accessはこの問題をユーザーの同意を省くことで発生させないようにします。
予め付与する権限を定義し、システム同士のやり取りで連携できるようにします。
システム間で連携が完了するため、ユーザーが同意することはありません。
そのため、Cross App Accessを使えばユーザーは同意疲れを起こさず、適切な権限での連携が可能となります。
これがCross App Accessの目的の一つである同意疲れの解消です。
以上Cross App Accessの目的を確認しました。

## Cross App Accessのざっくりとした解説
:::details 一文要約
Cross App AccessはIDトークンもしくはSAMLアサーションの取得、Token Exchange、JWT Profile for OAuth 2.0が構成要素です。
:::
先ほどCross App Accessの目的は見たので、ここでは技術的な概要を試してみます。
なお、ここで解説する内容は[draft-ietf-oauth-identity-assertion-authz-grant-01](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/)を元にしています。
内容は以下の通りです。
- Cross App Accessの全体像
- User SSO→ID Tokenで把握しておくこと
- Token Exchangeの概要とCross App Accessでの役割
- JWT Profile for OAuth 2.0の概要とCross App Accessでの役割

まずは全体像をざっくりと説明します。
### Cross App Accessの全体像
Cross App Acessの全体像は以下の通りです。
![](/images/about-cross-app-access/flow.png)
凄いざっくり言うと、認証後システムだけでアクセストークンを取得するイメージです。
もうちょっとだけ段階を分けて書くと以下の通りです。
1. 認証したことを示すトークン(IDトークンやSAMLアサーション)を取得
2. 認証したことを示すトークンをアクセストークンを取得するためのJWTに交換
3. アクセストークン取得用のJWTを渡してアクセストークンを使用する

肝となるのはシステム間通信で構築されていることです。
システム間通信で完結することで、IdPの管理者は抜け漏れなく権限を設定できます。
また、ユーザー側も何度も権限付与に同意する必要がなくなります。
Cross App Accessの全体像と肝になる部分を説明しました。
次から、各フローについて簡単にみていきます。
## フロー１ 認証を示すトークンの取得
:::details 一文要約
IDトークンかSAMLアサーションを取得します。
:::
最初のフローは認証を示す値を取得するためのフローです。
認証方式に一応制限はなく、どのような方法を用いても問題ありません。
ただし、認証したことを示すトークンを安全に取得できることが望ましいです。
そのため、よっぽどの事情がない限りはOpenID ConnectによるIDトークンか、SAMLによるSAMLアサーションを取得することが望ましいと思います。
また、後続のトークンを交換する部分の仕様においても、SAMLやOpenID Connectのケースが用意されています。
以上から、少し乱暴ですがOpenID ConnectかSAMLを用いた認証をすると理解するとイメージがわきやすいです。
## フロー2 トークンの交換
:::details 一文要約
IDトークンかSAMLアサーションを認可サーバーからアクセストークンを取得できるためのJWTに変換します。
:::
先ほどの認証を示すトークンから、最終的に権限情報（アクセストークン）を取得するのがCross App Accessの流れです。
ですが、IDトークンやSAMLアサーションを使って、アクセストークンを取得するリクエストは実行できません。
そのため、IDトークンやSAMLアサーションの情報を活かしつつ、アクセストークンの取得リクエストに使えるものへ変換する必要があります。
そこで使うのが、[Token Exchange](https://www.rfc-editor.org/rfc/rfc8693.html)という仕様です。
Token Exchangeは受け取ったトークンから、権限の委譲や別のユーザーへのなりすましを行うトークンを返す仕様です。
詳細な説明は以下川崎さんの記事が詳しいので、適宜参照していただければと思います。
https://qiita.com/TakahikoKawasaki/items/d9be1b509ade87c337f2
Cross App Accessでは、権限の委譲を行う目的でToken Exchangeを利用しています。
やっていることは、IDトークンやSAMLアサーションを使い、アクセストークンを取得するための情報が詰まったJWTを返却することです。
IdPが「指定サービスのアクセストークンを発行することを認めました」とするのが、Token Exchangeによるトークンの交換が行っていることだと考えています。
OAuth2のAuthorization Code Flowでいうと、認可コードの発行とクライアントへの返却までかなと思っています。
## フロー3 アクセストークンの発行
:::details 一文要約
JWTを使ってOAuthにおけるトークンエンドポイントにリクエストして、アクセストークンを発行します。
:::
ここはトークンエンドポイントにリクエストを行っています。
2で発行したJWTなどの情報から必要な権限を付与した上で、アクセストークンを発行しています。
OAuth2に馴染みがある人はClient Credentialフローによるアクセストークンを発行とおなじような感じです。
もちろん、リクエストで必要な値は全く違いますのでその点はご留意ください。
システム間通信でアクセストークンを発行している部分だけ共通点として見出してください。
やっていることは以上です。
ちなみに、[以下記載](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/#:~:text=The%20Resource%20Authorization%20Server%20SHOULD%20NOT%20return%20a%20Refresh%20Token%0A%20%20%20when%20an%20Identity%20Assertion%20JWT%20Authorization%20is%20exchanged%20for%20an%0A%20%20%20Access%20Token%20per%20Section%205.2%20of%20%5BI%2DD.ietf%2Doauth%2Didentity%2Dchaining%5D.)から、アクセストークンを発行する時リフレッシュトークンは発行すべきではないです。
>The Resource Authorization Server SHOULD NOT return a Refresh Token
   when an Identity Assertion JWT Authorization is exchanged for an
   Access Token per Section 5.2 of [I-D.ietf-oauth-identity-chaining].

そのため、アクセストークンの再発行は再度Cross App Accessのフローを行う必要があります。
以上がCross App Accessの大まかな全体像です。
内部のリクエストまで見るともっと複雑ですが、全体像としては
- IDトークン or SAMLアサーションを取得する
- 上記トークンを**IdP**を経由して、JWTに変換する
- JWTを使っていい感じの権限をもったアクセストークンを取得する
に集約されます。

間にIdPを挟むことで、誰がどんな権限のアクセストークンを取得するかのログをIdPに集約できます。
これによって、IdPの管理者が様々なサービスとの接続を管理できるようになります。

## Cross App Accessを動かす
実際にCross App AccessをOktaを使って試してみます。
手順は以下の記事を参考にしています。
https://developer.okta.com/blog/2025/09/03/cross-app-access
なお、上記記事の中ではAWSのBedrockを使っています。
ですが、従量課金が怖いので以下の記事をベースにOpen Router経由で実行するようにします。
https://zenn.dev/maronn/articles/exec-auth0-ai-agent-free
https://zenn.dev/asap/articles/5cda4576fbe7cb

もし、AWSのBedrockを使うのが全然問題なければ、参考記事をもとに構築してもらっても大丈夫です。
また、以下のことは前提としているので、前提を満たしていない方は適宜用意をお願いします。
- OktaのアカウントとOrg作成（[こちらのスクラップ](https://zenn.dev/maronn/scraps/5b345c5165c231)から用意できるかと思います）
- [Okta Verify](https://help.okta.com/en-us/content/topics/mobile/okta-verify-overview.htm)のインストール
- Open Routerのアカウント作成とAPIキーの生成（[手順参考記事](https://zenn.dev/asap/articles/5cda4576fbe7cb)）
- [サンプルプロジェクト](https://github.com/oktadev/okta-cross-app-access-mcp)のクローン
- Dockerの起動とDevContainerの実行環境設定

以上の前提を踏まえ、以下の内容で記載していきます。
- OktaでCross App Accessの設定
- 検証ユーザーをOktaで作成と設定
- AWS BedrockからOpen Router経由での実行に変更
- アプリの設定
- ログの確認

### Okta側の設定
まずはOktaでCross App Accessを使えるようにします。
Oktaにアクセスし、サイドメニューの設定→機能にアクセスします。
ページ内の早期アクセス内にある、「Cross App Access」を有効にします。
![](/images/about-cross-app-access/cross_app_access_enable.png)
有効後はサイドメニューのアプリケーション→アプリケーションに移動し、「アプリカタログを参照」ボタンをクリックします。
![](/images/about-cross-app-access/apps_create_btn.png)
検索欄で「Cross App Access」と検索し、表示される「Agent0」から始まるCross App Access用のアプリを作成します。
![](/images/about-cross-app-access/cr_agent0.png)
選択後、統合を追加して任意の名前を決めたらアプリを作成します。
同様にTodo0から始まるCross App Access用のアプリを作成します。
作成したら、以下のように二つのアプリが存在していればよいです。
![](/images/about-cross-app-access/l_apps.png)
以降は、上記画像にあるSampleAgent0、SampleTodo0というアプリの名前を使って説明します。
SampleAgent0にアクセスし、「接続の管理」タブに遷移します。
そこで、「同意を​付与された​アプリ」と「同意を​提供する​アプリ」にそれぞれSampleTodo0を追加します。
以下のような状態になっていれば良いです。
![](/images/about-cross-app-access/link_apps.png)
これでCross App Accessに使うアプリの準備は完了です。
### ユーザーの作成、設定
次にサンプルアプリでログインするためのユーザーを作成します。
サイドメニューのディレクトリ→ユーザーへアクセスし、「ユーザーを追加」ボタンをクリックします。
表示されるモーダルに適宜値を入力しますが、ユーザー名とメインのメールアドレスは**必ず**`bob@tables.fake`にしてください。
![](/images/about-cross-app-access/m_cr_user.png)
厳密に言えば、後続で使用するサンプルプロジェクトの中に含まれているユーザーのメールアドレスであれば良いです。
ただ、分かりやすくするために`bob@tables.fake`を作成するよう記載しました。
後は、検証用のユーザーなのでモーダル下部で初期パスワードを設定すると、追加の作業がなくて楽かと思います。
私は以下のようにして作成しました。
![](/images/about-cross-app-access/s_user.png)
ユーザー作成後は、サイドメニューのアプリケーション→アプリケーションから遷移し、SampleAgent0にアクセスします。
その中の、割り当てタブを選び、割り当てボタンから「ユーザーに割り当て」を選択します。
モーダルが表示されるので、そこで先ほど作成したユーザーを追加してください。
最終的に保存し、以下のようになっていれば手順は適切に行われています。
![](/images/about-cross-app-access/s_add_user.png)
SampleTodo0側にも同様の手順で同じユーザーを割り当てます。
以上でユーザーの設定も完了です。
### アプリの設定
予めクローンしたアプリをコードエディタで開き、以下の対応をします。
- SampleAgent0とSampleTodo0のClient IDとClient Secretを環境変数に設定
- Open Routerの環境変数設定
- AWS Bedrock部分の差し替え

#### Oktaに関する設定
まず、Oktaのアプリケーション→アプリケーションからSampleAgent0、SampleTodo0にアクセスします。
その後、サインオンタブの設定項目から、Client IDとClient Secretにコピペします。
![](/images/about-cross-app-access/ci_cl.png)
コピーしたら、SampleAgent0の値はサンプルプロジェクトのpackages/authorization-server/.env.agent.defaultにある`CUSTOMER1_`から始まる値に以下のように設定します。
```
CUSTOMER1_EMAIL_DOMAIN="tables.fake"
CUSTOMER1_AUTH_ISSUER="対象OrgのオリジンURL"
CUSTOMER1_CLIENT_ID="コピーしたClient ID"
CUSTOMER1_CLIENT_SECRET="コピーしたClient Secret"
```
なお、`CUSTOMER1_AUTH_ISSUER`については、管理画面のオリジンURLではなく、ダッシュボードのオリジンURLを設定してください。
つまり、管理画面のオリジンURLが`https://xxxx-admin.okta.com`の場合、`CUSTOMER1_AUTH_ISSUER`には`https://xxxx.okta.com`を設定してください。
SampleTodo0の値はpackages/authorization-server/.env.todo0.defaultへ設定してください。
設定するオリジンURLは一緒のものです。
ここまで終われば、Okta側の設定は完了です。
#### Open Routerの設定
次に、OpenRouterの設定を行います。
packages/agent0/.env.defaultに以下の環境変数を追加します。
```
OPENROUTER_API_KEY="Open Routerで取得したAPIキー"
OPENROUTER_MODEL="https://openrouter.ai/models?fmt=cards&max_price=0&supported_parameters=toolsで選んだ無料モデル"
OPENROUTER_BASE_URL="https://openrouter.ai/api/v1"
```
モデル名については、上記に記載しているURLに飛んで任意のものを選択します。
選択したら、以下画像の赤枠部分を環境変数に設定します。
![](/images/about-cross-app-access/m_name.png)
次に、アプリ側の差し替えを行います。
packages/agent0/src配下に任意のTSファイルを作り、以下の実装をします。（今回はmcp-openrouter-client.tsで作成しました）
```typescript
// OpenRouter版 MCPクライアント（検証用・最小限実装）
// mcp-bedrock-client.tsの代わりにこのファイルを使用してください

import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';
import { SYSTEM_PROMPT } from './config.js';
import { logger } from './logger.js';
import { ChatMessage, MCPServerConfig } from './types.js';

export class MCPBedrockClient {
    private apiKey: string;
    private baseURL: string = 'https://openrouter.ai/api/v1';
    private mcpClients: Map<string, Client> = new Map();
    private modelId: string;
    private conversationHistory: ChatMessage[] = [];

    constructor(config: any = {}) {
        this.apiKey = process.env.OPENROUTER_API_KEY || '';
        this.modelId = process.env.OPENROUTER_MODEL || 'deepseek/deepseek-chat-v3-0324:free';

        if (!this.apiKey) {
            throw new Error('OPENROUTER_API_KEY is required');
        }

        logger.info('Initialized Agent0 MCP Client with OpenRouter', {
            modelId: this.modelId,
        });
    }

    /**
     * Connect to an MCP server
     */
    async connectToMCPServer(serverConfig: MCPServerConfig): Promise<void> {
        try {
            logger.loading(`Connecting to MCP server: ${serverConfig.name}`);

            const transport = new StdioClientTransport({
                command: serverConfig.command,
                args: serverConfig.args || [],
                env: serverConfig.env,
            });

            const client = new Client(
                {
                    name: 'agent0',
                    version: '1.0.0',
                },
                {
                    capabilities: {},
                }
            );

            await client.connect(transport);
            this.mcpClients.set(serverConfig.name, client);

            logger.success(`Successfully connected to MCP server: ${serverConfig.name}`);

            const tools = await client.listTools();
            logger.mcpEvent(serverConfig.name, 'tools_discovered', {
                count: tools.tools?.length || 0,
                tools: tools.tools?.map((t) => t.name) || [],
            });
        } catch (error) {
            logger.error(`Failed to connect to MCP server ${serverConfig.name}`, error);
            throw error;
        }
    }

    /**
     * Disconnect from an MCP server
     */
    async disconnectFromMCPServer(serverName: string): Promise<void> {
        const client = this.mcpClients.get(serverName);
        if (client) {
            await client.close();
            this.mcpClients.delete(serverName);
            logger.success(`Disconnected from MCP server: ${serverName}`);
        }
    }

    /**
     * List all available tools from connected MCP servers
     */
    async listAllTools(): Promise<Record<string, any[]>> {
        const allTools: Record<string, any[]> = {};

        for (const [serverName, client] of this.mcpClients) {
            try {
                const result = await client.listTools();
                allTools[serverName] = result.tools || [];
            } catch (error) {
                logger.error(`Error listing tools from ${serverName}`, error);
                allTools[serverName] = [];
            }
        }

        return allTools;
    }

    /**
     * Call a tool on a specific MCP server
     */
    async callTool(serverName: string, toolName: string, args: any): Promise<any> {
        const client = this.mcpClients.get(serverName);
        if (!client) {
            throw new Error(`MCP server "${serverName}" not connected`);
        }

        try {
            logger.toolCall(serverName, toolName, args);

            const result = await client.callTool({
                name: toolName,
                arguments: args,
            });

            logger.toolResult(serverName, toolName, result);
            return result;
        } catch (error) {
            logger.error(`Error calling tool ${toolName} on ${serverName}`, error);
            throw error;
        }
    }

    /**
     * Send a message to OpenRouter and process any tool calls
     */
    async sendMessage(message: string): Promise<string> {
        try {
            // Add user message to conversation history
            this.conversationHistory.push({
                role: 'user',
                content: message,
                timestamp: new Date(),
            });

            // Get available tools
            const allTools = await this.listAllTools();
            const toolsDescription = this.formatToolsForClaude(allTools);

            // Prepare the prompt with tool information
            const systemPrompt = `${SYSTEM_PROMPT}\n\n${toolsDescription}`;

            // Prepare conversation for Claude
            const conversationText = this.conversationHistory
                .map((msg) => `${msg.role}: ${msg.content}`)
                .join('\n\n');

            const fullPrompt = `${systemPrompt}\n\nConversation:\n${conversationText}\n\nassistant:`;

            // Call OpenRouter API
            const response = await fetch(`${this.baseURL}/chat/completions`, {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${this.apiKey}`,
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    model: this.modelId,
                    messages: [
                        {
                            role: 'user',
                            content: fullPrompt,
                        },
                    ],
                    max_tokens: 4000,
                }),
            });

            if (!response.ok) {
                const errorText = await response.text();
                throw new Error(`OpenRouter API error: ${response.status} ${errorText}`);
            }

            const responseBody = await response.json() as { choices: { message: { content: string } }[] };
            let assistantMessage = responseBody.choices[0].message.content;

            // Check if Claude wants to use a tool
            try {
                // First try to parse the entire message as JSON
                const toolRequest = JSON.parse(assistantMessage);
                if (toolRequest.action === 'tool_call') {
                    console.log(`Calling tool: ${toolRequest.tool} on server: ${toolRequest.server}`);

                    const toolResult = await this.callTool(
                        toolRequest.server,
                        toolRequest.tool,
                        toolRequest.arguments
                    );

                    // Send tool result back to Claude
                    const followUpPrompt = `The tool "${toolRequest.tool}" returned: ${JSON.stringify(
                        toolResult,
                        null,
                        2
                    )}\n\nPlease provide a natural language response based on this information.`;

                    const followUpResponse = await fetch(`${this.baseURL}/chat/completions`, {
                        method: 'POST',
                        headers: {
                            'Authorization': `Bearer ${this.apiKey}`,
                            'Content-Type': 'application/json',
                        },
                        body: JSON.stringify({
                            model: this.modelId,
                            messages: [
                                {
                                    role: 'user',
                                    content: followUpPrompt,
                                },
                            ],
                            max_tokens: 4000,
                        }),
                    });

                    if (followUpResponse.ok) {
                        const followUpBody = await followUpResponse.json() as { choices: { message: { content: string } }[] }
                        assistantMessage = followUpBody.choices[0].message.content;
                    }
                }
            } catch (error) {
                // If full message isn't JSON, try to extract JSON from the message
                try {
                    const jsonMatch = assistantMessage.match(/\{[\s\S]*\}/);
                    if (jsonMatch) {
                        const toolRequest = JSON.parse(jsonMatch[0]);
                        if (toolRequest.action === 'tool_call') {
                            console.log(`Calling tool: ${toolRequest.tool} on server: ${toolRequest.server}`);

                            const toolResult = await this.callTool(
                                toolRequest.server,
                                toolRequest.tool,
                                toolRequest.arguments
                            );

                            // Send tool result back to Claude
                            const followUpPrompt = `The tool "${toolRequest.tool}" returned: ${JSON.stringify(
                                toolResult,
                                null,
                                2
                            )}\n\nPlease provide a natural language response based on this information.`;

                            const followUpResponse = await fetch(`${this.baseURL}/chat/completions`, {
                                method: 'POST',
                                headers: {
                                    'Authorization': `Bearer ${this.apiKey}`,
                                    'Content-Type': 'application/json',
                                },
                                body: JSON.stringify({
                                    model: this.modelId,
                                    messages: [
                                        {
                                            role: 'user',
                                            content: followUpPrompt,
                                        },
                                    ],
                                    max_tokens: 4000,
                                }),
                            });

                            if (followUpResponse.ok) {
                                const followUpBody = await followUpResponse.json() as { choices: { message: { content: string } }[] }
                                assistantMessage = followUpBody.choices[0].message.content;
                            }
                        }
                    }
                } catch (innerError) {
                    // Not a tool call, continue with normal response
                }
            }

            // Add assistant response to conversation history
            this.conversationHistory.push({
                role: 'assistant',
                content: assistantMessage,
                timestamp: new Date(),
            });

            return assistantMessage;
        } catch (error) {
            logger.error('Error sending message to OpenRouter', error);
            throw error;
        }
    }

    /**
     * Format available tools for Claude's understanding
     */
    private formatToolsForClaude(allTools: Record<string, any[]>): string {
        let description = '';

        for (const [serverName, tools] of Object.entries(allTools)) {
            if (tools.length > 0) {
                description += `\nServer: ${serverName}\n`;
                for (const tool of tools) {
                    description += `- ${tool.name}: ${tool.description || 'No description'}\n`;
                    if (tool.inputSchema) {
                        description += `  Parameters: ${JSON.stringify(tool.inputSchema, null, 2)}\n`;
                    }
                }
            }
        }

        return description || 'No tools available.';
    }

    /**
     * Clear conversation history
     */
    clearHistory(): void {
        this.conversationHistory = [];
    }

    /**
     * Get conversation history
     */
    getHistory(): ChatMessage[] {
        return [...this.conversationHistory];
    }

    /**
     * Disconnect from all MCP servers
     */
    async disconnect(): Promise<void> {
        const disconnectPromises = Array.from(this.mcpClients.keys()).map((serverName) =>
            this.disconnectFromMCPServer(serverName)
        );
        await Promise.all(disconnectPromises);
    }
}

```
厳密にはもっと綺麗にできるかもしれませんが、とりあえず動くコードを目指しました。
なので、コードは読んで得られるものはないので、何も考えずにコピペしてもらえると幸いです。
作成したTSコードを使うために、packages/agent0/src/web-server.tsで以下のように変更します。
```typescript
// 略

// import { MCPBedrockClient } from './mcp-bedrock-client.js';
import { MCPBedrockClient } from './mcp-openrouter-client.js';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

//略
```
なお、なぜかプロジェクトのサンプルコードそのままでは`__dirname`が使えませんでした。
なので、生成AIに聞いてとりあえずアプリが起動するためのコードを設定しました。
また、サンプルプロジェクトのpackages/agent0/src/server/controllers/oidc.tsではid-assert-authz-grant-clientディレクトリへの相対パスを指定していましたが、その場合でも動きませんでした。
なので、packages/agent0/package.jsonのdependenciesに以下のパッケージを追加しています。
```json
  "dependencies": {
    "id-assert-authz-grant-client": "workspace:*"
  }
```
また、パッケージをライブラリとしてインストールしたので、当該部分は以下のように変更しています。
```typescript
// 元々 import { AccessTokenResult, exchangeIdJwtAuthzGrant, ExchangeTokenResult, requestIdJwtAuthzGrant } from '../../../../id-assert-authz-grant-client';

import { AccessTokenResult, exchangeIdJwtAuthzGrant, ExchangeTokenResult, requestIdJwtAuthzGrant } from 'id-assert-authz-grant-client';

```
ここまでできれば、プロジェクト直下で`yarn setup:env && yarn bootstrap`を実行します。
途中でデータベースをリセットしてよいか確認されますが、`y`を入力してそのままリセットさせてください。
ここまで来たらアプリ側の設定も完了です。
実際に動かしみましょう。
### アプリの動作確認
プロジェクト直下で`yarn start`を実行します。
実行後、`http://localhost:3001`にアクセスします。
すると、以下のような画面が表示されるので、フォームに`bob@tables.fake`を入力し送信します。

すると、Oktaのログイン画面で表示されるので、ユーザー作成の際に設定したパスワードを入力します。
Okta Verifyの設定が求められると思うので、その設定を行い認証します。
認証後、`http://localhost:3000`にアクセスします。
サイドメニューの「initialize」ボタンクリックし、成功したらその下のCoonect to IDPをクリックします。
![](/images/about-cross-app-access/i_mcp.png)
成功するとログに以下のような記載が確認できます。
![](/images/about-cross-app-access/s_t_e.png)
認証の時にToken Exchangeが行われ、JWTを取得するのが確認できます。（ちなみにOktaとToken Exchangeしている結果を直接見たい場合は、packages/authorization-server/jwt-authorization-grant-token-exchange.jsにconsole.logを仕込むと結果が確認できます）
そして、その後アクセストークンを取得するログも出ます。
Okta側にもJWT Profile for OAuth 2.0でアクセストークンを取得するログを確認できます。
![](/images/about-cross-app-access/okta_iga.png)
これで、`http://localhost:3000`は`http://localhot:3001`のAPIを実行できるアクセストークンが取得できます。
実際に、「Create Sample Task」と入力し送信すると、`http://localhot:3001`にタスクができるようになります。
以上で動作確認を終わります。
## 調べて、試した感想
最後にこの記事を書くにあたって個人的に感じたことをダラダラと書いていきます。
### 個人開発、小規模アプリでは使わないな
触ってみて改めてエンタープライズ向けだなと思います。
Cross App Accessが求められることになったのが、エンタープライズ向けの認可なので当たり前ことを言っている自覚はあります。
なのですが、実際には必要としないケースが多いかなと思います。
会社として統一した認証基盤を持っていないと、Cross  App Accessまで至ることはなさそうです。
### 何やっているか分かりにくい
ユーザビリティを損なわないために、システム間通信でアクセストークンを取得するため内部で何をしているかが理解しにくいです。
分かりにくいと、一連のフローがきちんと実装できているかのチェックが個人的には難しいです。
ユーザーの操作が少ない分、実装できたかチェックしようにもコードみないとなんとも...となりそうです。
ただ、特にリソースサーバー用の認可サーバー（フロー図でいうとResource Application Authorization Server）のチェックが難しいかなと思います。
とはいえ、今回のIdPもAuthorization Serverも全てOktaに集約するとそれはそれで関係性が見えにくいです。
認証もトークンの発行もOktaに集約されるので、何がIdPで何が認可サーバーを示しているかの把握が難しいかなと思います。
結局は慣れな気もしますが、Cross App Accessをきちんとできているという担保は悩ましいです。
### 仕様は分かりやすい
超個人的な話ですが、仕様は読んでて把握しやすいです。
RFCの分量もそこまで多くなく、かつ元になっている仕様も個人的には読破しやすい分量です。
Cross App Accessの仕様を読むだけで、Token Exchange、JWT Profile for OAuth 2.0なども把握できます。
一粒で二度美味しいってやつです。
この辺が個人的に感じたことです。
## おわりに
今回はCross App Accessについて概要をざっくりと見て、Oktaで実際に試してみました。
やりたいことは、IdPに管理を集約させるということでエンタープライズ向けだということを実感できます。
会社として、接続できるサービスをきちんと管理したい。だけど、ユーザビリティを落として現場に負担をかけたくないと思っている場合は検討してみる価値はあると思います。
まだまだ、仕様もdraftですが必要となれば私も自社に取り入れられないか考えていきたいです。
ここまで読んでいただきありがとうございました。