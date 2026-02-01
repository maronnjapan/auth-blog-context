---
title: "Auth0のユーザーブロックをリアルタイム検知する仕組みをEvent Streamで作成した"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Auth0","EventStream","zennfes2025infra"]
published: true
published_at: 2025-10-21 08:30
---

## はじめに
以下のような記事が投稿されるなど、認可処理を行うためのサーバーはますます増えていくと感じています。
https://zenn.dev/layerx/articles/7e69eccac4ae51

そして、こういった認可管理を一元化するために、IdP（Identity Provider）を導入するというのは自然なアプローチかなと思います。
また、IdPへの負荷なども考えて、IdPに都度問い合わせる形ではなく、JWT形式のアクセストークンを発行してもらい、各アプリケーションサーバーで署名検証を行うことが必要な場合もあります。
Auth0はJWT形式のアクセストークンを発行でき、公開鍵の公開も行っているので、管理を一元的にしつつ検証自体は各アプリケーションサーバーにゆだねることができます。
これによって、アプリケーションサーバーが増えていっても、IdPがボトルネックになることを防げます。
しかし、このアプローチには一つ大きな課題があると感じています。
それは、アクセストークンを無効化したい時です。
Auth0にはユーザーブロック機能があり、ユーザーを即座にブロックできるようになっています。
しかし、ブロックができたとしてもJWT形式で発行したアクセストークンは無効化されません。
例えば、攻撃者がアカウントを乗っ取り、それに気づいた管理者がAuth0でそのユーザーをブロックしたとします。
ですが既に取得したトークンを使い続ける限り、アクセストークンを使ってAPIにアクセスできてしまいます。
これを防ぐためにアクセストークンの有効期限を短く設定すれば、セキュリティは向上します。
が、ユーザーは頻繁にログインを求められ、ユーザビリティが大きく低下します。
逆に、長い有効期限を設定すると、ユーザビリティは向上しますが、トークンが盗まれた際のリスクが増大します。
この問題を解消する方法の一つとして、Auth0のEvent Stream機能を活用するというのがこの記事の内容です。
具体的には、ユーザー情報がアップデートされた際に発行されるuser.updatedイベントを利用して、ユーザーがブロックされたことを各アプリケーションサーバーに通知します。
そして、Auth0のJWT形式のアクセストークンはデフォルトでユーザーIDが含まれているため、、各アプリケーションサーバーはアクセストークンの署名検証に加えて、ユーザーIDをキーにしてブロック状態をチェックします。
これにより、Auth0でユーザーをブロックした瞬間に、各アプリケーションサーバーがそれを検知し、既存のセッションを持つユーザーであってもアクセスを拒否できるようになります。
このやり方が正解ではないかもしれませんが、ユーティリティを損なわずにセキュリティを強化する一つの方法として参考になれば幸いです。
それでは、具体的な実装方法を見ていきます。

### この記事で作るものの概要

この記事では、Auth0のユーザーブロックイベントを検知して、Cloudflare Workerで署名付き通知を生成し、Next.jsアプリケーションで通知を受信してRedisにブロック状態を保存する、という一連のシステムを構築していきます。
そして、APIリクエスト時にRedisをチェックして、ブロック済みユーザーのアクセスを拒否します。
このシステムにより、Auth0でユーザーをブロックした瞬間に、各アプリケーションサーバーがそれを検知し、既存のセッションを持つユーザーであってもアクセスを拒否できるようにします。

### 使用する技術スタック

今回のシステムでは、Auth0をIdPとして使用し、Event Streams機能でイベントを配信します。
Cloudflare WorkersとHonoでWebhookを受信してJWT署名を生成し、Next.jsとAuth.jsで認証とアプリケーション本体を構築します。
ブロック状態の管理にはUpstash Redisを使用し、JWT検証にはjoseライブラリ、ES256署名生成にはWeb Crypto APIを使用します。
全体のコードについては、以下のリンクにあります。
https://github.com/maronnjapan/sample-id-app/tree/notification-blocked-user
また、このリポジトリを使ったデモを試すには[README](https://github.com/maronnjapan/sample-id-app/blob/notification-blocked-user/README.md)を参照してください。
このブログでは、この中の主要な部分を抜粋して説明していきます。
それでは、実際のシステムの流れを見ていきましょう。

## アプリのデモ
まず、今回作成したアプリケーションのデモを紹介します。
![](/images/notify-block-user-by-auth0-event-stream/notify-user-block-demo.gif)
上記のデモでは、まずAuth0を使いデモアプリケーションのログイン処理を行っています。
その後、内部でAPIを叩き、アクセストークンの検証がOKであれば値が表示されるようになっています。
特にブロックをしなければ、実行ボタンを押すたびに正常にAPIから値が返ってくることが分かります。
一方、Auth0の管理画面でユーザーをブロックした後に、再度APIを実行するとそれまで返ってきた値が返ってこなくなります。
代わりに、アクセスが拒否されたことを示すエラーメッセージが表示されます。
以上がデモとなります。
デモを確認したので、システムの内部について見ていきます。
## システムの流れ

### フロー図

ブロック通知を受信・保存するざっくりとしたフローは以下のようになります。
![](/images/notify-block-user-by-auth0-event-stream/auth0-block-notification-flow.png)
Auth0でUser Block Eventが発生すると、Event Streamを通じてuser.updatedイベントがWorkerのwebhookエンドポイントに配信されます。
Auth0のEvent StreamはWebhookエンドポイントへのリクエストの際にBearerトークンをAuthorizationヘッダーに搭載できます。
今回はこの機能を使って、Worker側でWebhookがAuth0からのリクエストであるかを検証します。
Workerは検証を行った後、JWT署名を生成し、Next.jsのapi/user-blockエンドポイントに通知を送信します。
Next.jsはJWTを検証してRedisに保存し、その後のユーザーリクエストに対してRedisをチェックして、ブロックされている場合は403を返します。
これによって、Auth0でユーザーがブロックされた瞬間から、ブロックしたユーザーがAPIを実行することを防げます。

### 各コンポーネントの役割

システムを構成する各コンポーネントの役割を整理しておきます。
Auth0はユーザー認証を担当し、Event Streams機能でuser.updatedイベントを配信します。
Cloudflare Workerは、Auth0からのWebhookを受信してBearerトークンでリクエストの検証を行い、ユーザー情報を含むJWTをES256で署名します。
さらに、JWKSエンドポイントで公開鍵を提供します。
Next.jsアプリケーションは、Auth.jsでAuth0と連携し、Workerからのブロック通知を受信してUpstash Redisにブロック状態を保存します。
そして、APIリクエスト時にブロック状態をチェックします。
Upstash Redisは、ユーザーのブロック状態を永続化し、サーバーレス環境で動作するため、Vercelとの統合が容易です。

### Auth0 Event Streams（Early Access）について

今回のシステムの核となる機能が、Auth0のEvent Streamsです。
この機能を使うことで、Auth0で発生した各種イベントをリアルタイムに外部システムへ配信できます。
今回使用するuser.updatedイベントは、ユーザー情報が更新された際に発行されます。
イベントペイロードには、user_idやblocked、emailなどの情報が含まれています。
このblockedプロパティを確認することで、ユーザーがブロックされたかどうかを判定できます。
ただし、注意点として、user.updatedイベントはブロック以外の更新でも発火します。
例えば、メールアドレスの変更やメタデータの更新などでも同じイベントが配信されます。
そのため、本来であればblockedプロパティの変化を検知するフィルタリングロジックが必要になります。
今回の実装では、シンプルさを優先して全てのuser.updatedイベントを処理していますが、本番運用では効率化のためにフィルタリングを追加することをおすすめします。

Auth0のEvent Streamsは、記事執筆時点で[Early Access](https://auth0.com/docs/customize/events/create-an-event-stream#:~:text=an%20Event%20Stream-,Events%20are%20currently%20available%20in,.,-To%20learn%20more)の機能となっています。
そのため、今後仕様が変更される可能性があることを念頭に置いてください。
最新の仕様については、Auth0の公式ドキュメントを必ず確認をお願いします。
### なるべく安全に通知するための工夫
このシステムでは、単純なリクエスト形式にしてしまうと、誰でもユーザーをブロックできてしまうので、そういったことを防ぐためにいくつかのセキュリティ対策を講じています。
Worker側では、まずBearerトークンを実装し、Auth0から送信されるWebhookのトークンが一致しない場合は401を返却します。
生成したJWTにはES256で署名し、改ざん防止を行っています。
また、有効期限を5分に設定することで、盗まれても影響を最小限にしています。
Next.js側では、WorkerのJWTについてWorkerのJWKSを使用した署名検証と有効期限のチェックを行っています。。
そして、Redisでブロック管理を行うことで、サーバーレス環境に適した永続化と高速なアクセス、簡潔なデータ構造を実現しています。
単純にブロックしたユーザーを通知するだけなら、不要な実装ですがそういうわけにはいかないので、これらの工夫を盛り込んでいます。

## Cloudflare Workerの実装（Webhook受信とJWT署名）

それでは、実際の実装を見ていきます。まずはCloudflare Worker側から説明します。

### Honoを使ったエンドポイント設計

今回、Workerのルーティングには、Honoを使用しています。
理由は単純にHonoはCloudflare Workersとの相性が非常に良く、簡単にデプロイできるからです。
Workerでは2つのエンドポイントを公開します。
1つ目は署名検証用の公開鍵をJWKS形式で公開する`GET /.well-known/jwks.json`で、2つ目はAuth0イベントを受信してJWT署名された通知を生成する`POST /webhook`です。

実装は以下のようになります。

```typescript
import { Hono } from 'hono'

const app = new Hono<{ Bindings: Bindings }>()

// JWKS公開エンドポイント
app.get('/.well-known/jwks.json', async (c) => {
  const { publicJwk } = await getSigningMaterial(c.env)
  return c.json({ keys: [publicJwk] })
})

// Webhook受信エンドポイント
app.post('/webhook', async (c) => {
  // 実装の詳細は後述
})

export default app
```

### `user.updated`イベントのペイロード処理

Webhookエンドポイントでは、Auth0から送信されたイベントペイロードを処理します。まず、Bearerトークンによる検証を行います。

```typescript
app.post('/webhook', async (c) => {
  // 1. トークンの検証
  const secrets = c.env.AUTHORIZATION_SECRETS.split(',')
    .map(s => s.trim())
    .filter(s => s !== '')
  
  const authHeader = c.req.header('Authorization')
  const authorizationToken = authHeader?.replace('Bearer ', '')
  
  if (!authHeader || !authorizationToken || !secrets.includes(authorizationToken)) {
    return c.json({ message: 'Missing or invalid Authorization header' }, 401)
  }

  // 2. Auth0イベントペイロードからuser_idとblockedを抽出
  const jsonData = await c.req.json()
  const body: UserUpdateEvent = jsonData.data.object
  
  // 以降の処理...
})
```

AUTHORIZATION_SECRETSは環境変数にカンマ区切りで複数のシークレットを保持しています。これにより、シークレットローテーション時も古いシークレットを受け入れることができます。
認証が成功したら、ペイロードからuser_idとblockedを取得します。
前述の通り、user.updatedイベントはブロック以外の更新でも発火します。
現在の実装では、全てのイベントを処理していますが、将来的には以下のようなフィルタリングを追加できます。

```typescript
// 将来的な拡張例
const body: UserUpdateEvent = jsonData.data.object

// blockedプロパティの変化のみを処理したい場合
if (jsonData.data.type === 'user' && 'blocked' in body) {
  // ブロック関連の更新のみ処理
}
```

このように、必要に応じてフィルタリングロジックを追加することで、無駄な処理を減らすことができます。（今回はシンプルさを優先しています）

### ES256によるJWT署名の実装

次に、JWTの署名処理を見ていきます。
今回はES256、つまりECDSA with P-256 and SHA-256というアルゴリズムを使用します。
ES256は楕円曲線暗号を使用した署名アルゴリズムで、RSAに比べて鍵のサイズが小さく、署名・検証の速度も高速です。
後方互換などを考えるとRS256が無難ですが、今回署名についての互換性を考慮する必要はないので、ES256にしています。
Web Crypto APIを使用した署名処理の実装は以下のようになります。

```typescript
export const signJwt = async (
  privateKey: CryptoKey,
  payload: Record<string, unknown>,
  options: SignOptions = {}
): Promise<string> => {
  const header: Record<string, string> = {
    alg: 'ES256',
    typ: 'JWT',
  }

  const issuedAt = Math.floor(Date.now() / 1000)
  const normalizedPayload: Record<string, unknown> = {
    ...payload,
    iat: issuedAt
  }
  
  if (options.expiresInSeconds) {
    normalizedPayload.exp = issuedAt + (options.expiresInSeconds ?? 300)
  }

  const encodedHeader = base64UrlEncode(stringToArrayBuffer(JSON.stringify(header)))
  const encodedPayload = base64UrlEncode(stringToArrayBuffer(JSON.stringify(normalizedPayload)))
  const signingInput = `${encodedHeader}.${encodedPayload}`

  const signature = await crypto.subtle.sign(
    { name: 'ECDSA', hash: { name: 'SHA-256' } },
    privateKey,
    stringToArrayBuffer(signingInput)
  )

  const encodedSignature = base64UrlEncode(signature)
  return `${signingInput}.${encodedSignature}`
}
```

この関数では、まずJWTヘッダーとペイロードを作成し、iat（発行日時）とexp（有効期限）を設定します。
その後、ヘッダーとペイロードをBase64URLエンコードし、Web Crypto APIで署名を生成して、署名をBase64URLエンコードして結合します。
Base64URLエンコードは通常のBase64とは異なり、URLセーフな文字列を生成します。具体的には、+を-に、/を_に置き換え、末尾の=を削除します。
署名後、以下のようにリクエスト先として定義したURLに署名した値をボディに含めて、リクエストを行っています。
```typescript
// 3. JWTに署名（5分の有効期限）
const { privateKey } = await getSigningMaterial(c.env)
const token = await signJwt(
  privateKey,
  {
    user_id: body.user_id,
    blocked: body.blocked,
  },
  {
    expiresInSeconds: 300,
  }
)

// 4. 通知先URL（Next.jsなど）にPOST
const sendUrls = c.env.NOTIFY_URLS.split(',')
  .map(s => s.trim())
  .filter(s => s !== '')

await Promise.all(sendUrls.map(url => fetch(url, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ token }),
})))

return c.json({
  message: `User ${body.user_id} updated and JWT generated.`,
  token,
})
```

### JWKS公開エンドポイントの作り方
先ほど署名した値の生成と送信についてみたので、受け取り先が署名を検証するためのエンドポイントも作成します。
検証するためには、署名に使用した公開鍵が必要です。
この公開鍵を[JWKS（JSON Web Key Set）](https://datatracker.ietf.org/doc/html/rfc7517)形式で公開します。
JWKSエンドポイントの実装は以下のようになります。

```typescript
app.get('/.well-known/jwks.json', async (c) => {
  const { publicJwk } = await getSigningMaterial(c.env)
  return c.json({ keys: [publicJwk] })
})
```

getSigningMaterial関数では、秘密鍵から公開鍵を導出し、JWK形式に変換します。

```typescript
export const getSigningMaterial = async (privateKeyPem: string): Promise<SigningMaterial> => {
  // 秘密鍵のインポート
  const privateKey = await crypto.subtle.importKey(
    'pkcs8',
    pemToArrayBuffer(privateKeyPem),
    { name: 'ECDSA', namedCurve: 'P-256' },
    true,
    ['sign']
  )

  // 公開鍵の導出
  const publicKey = await crypto.subtle.exportKey('jwk', privateKey)
  
  // Key IDの計算
  const kid = await computeKid(publicKey)
  
  const publicJwk = {
    ...publicKey,
    kid,
    use: 'sig',
    alg: 'ES256',
  }

  return { privateKey, publicJwk }
}
```

Key ID（kid）は、JWKの座標（xとy）からSHA-256ハッシュを計算して生成します。

```typescript
const computeKid = async (jwk: JsonWebKey): Promise<string> => {
  if (!jwk.x || !jwk.y) {
    throw new Error('Missing EC coordinates in JWK')
  }
  
  const xBytes = base64UrlDecode(jwk.x)
  const yBytes = base64UrlDecode(jwk.y)
  
  const combined = new Uint8Array(xBytes.length + yBytes.length)
  combined.set(xBytes, 0)
  combined.set(yBytes, xBytes.length)
  
  const digest = await crypto.subtle.digest('SHA-256', combined)
  const kidSource = new Uint8Array(digest).slice(0, 12)
  
  return base64UrlEncode(kidSource.buffer)
}
```
このkidを使用することで、複数の鍵を管理する際にどの鍵で署名されたかを識別できます。(今回は複数鍵を作る実装はありませんが...。拡張性があるということでご容赦ください)
鍵マテリアルの生成は比較的重い処理なので、キャッシュ機構を実装します。
```typescript
let cachedMaterial: {
  cacheKey: string
  promise: Promise<SigningMaterial>
} | null = null

export const getSigningMaterial = (env: Bindings): Promise<SigningMaterial> => {
  if (!env.JWT_PRIVATE_KEY) {
    throw new Error('JWT_PRIVATE_KEY secret is not configured')
  }
  
  const cacheKey = `${env.JWT_PRIVATE_KEY}`
  
  if (!cachedMaterial || cachedMaterial.cacheKey !== cacheKey) {
    cachedMaterial = {
      cacheKey,
      promise: ensureSigningMaterial(env.JWT_PRIVATE_KEY)
    }
  }
  
  return cachedMaterial.promise
}
```
このキャッシュにより、同じ秘密鍵を使用する限り、鍵マテリアルの生成は初回のみとなります。
ただ、Cloudflare Workersの特性を考えた時にこの実装が適切かは微妙です。
なので、キャッシュについては参考程度にとどめ、よりよいキャッシュ方法があればそちらを使うようにしてください。
というか、Cloudflare Workersでキャッシュする際の良い方法を教えてください。

### （参考）Auth0 Event Streamの自動セットアップスクリプト

最後に、Auth0のEvent Streamを自動的にセットアップするスクリプトについて説明します。
このスクリプトは必須ではないですが、セットアップの際に便利だったので作成しました。
ただ、アプリケーションを動かすために必須のコードではありません。
そのため、気になる方は読んでいただければ幸いですが、スキップしても問題ありません。
なお、スクリプトを実行するには、Auth0でMachine to Machine Applicationを作成し、Management APIの権限を付与しておく必要があります。（作成方法は共有したGitHubのREADMEにて記載しております）
このスクリプト（create-event-stream.ts）を実行することで、ランダムなシークレット（UUID）の生成、.envのAUTHORIZATION_SECRETSの更新（最大3つ保持）、wrangler secret putでWorkerへのシークレット設定、Auth0でのnotifyUserBlock Event Streamの作成・更新が自動的に行われます。
手で都度ポチポチ作るのが面倒だったので、こういったスクリプトを作りました。
それでは、スクリプトの主要な部分を見てみましょう。

```typescript
const main = async () => {
  const config = collectEnv()
  const client = new ManagementClient({
    domain: config.tenantDomain,
    clientId: config.clientId,
    clientSecret: config.clientSecret
  })

  // 1. ランダムなシークレットを生成
  const secret = crypto.randomUUID().replaceAll('-', '')
  writeSecretEnvFile(secret)

  // 2. Wranglerにシークレットを設定
  execSync(`echo ${secret} | wrangler secret put AUTHORIZATION_SECRETS --name notify-user-block`)

  // 3. Event Streamを作成または更新
  const eventStreams = await client.eventStreams.list()
  const existingId = eventStreams.eventStreams.find(es => es.name === 'notifyUserBlock')?.id

  if (existingId) {
    // 既存のEvent Streamを更新
    await client.eventStreams.update(existingId, {
      destination: {
        type: 'webhook',
        configuration: {
          webhook_authorization: {
            token: secret
          }
        }
      }
    })
    return
  }

  // 新規作成
  await client.eventStreams.create(buildPayload({
    webhookEndpoint: config.webhookEndpoint,
    webhookToken: secret,
    eventStreamName: 'notifyUserBlock',
    eventTypes: ['user.updated'],
    status: 'enabled'
  }))
}
```

ここで重要なのは、eventTypes: ['user.updated']の部分です。
この設定により、Auth0でユーザー情報が更新された際に、WorkerのWebhookエンドポイントへイベントが配信されます。
また、status: 'enabled'を設定することで、Event Streamが即座に有効化されます。
このスクリプトを実行するには、`pnpm handle:event-stream`コマンドを使用します。
これにより、Auth0の設定が自動的に完了し、すぐにシステムを稼働させることができます。
## Next.jsアプリケーションの実装（ブロック通知の受信と検証）

続いて、Next.jsアプリケーション側の実装を見ていきます。
### Auth.jsとAuth0の連携
まず、[Auth.js](https://authjs.dev/getting-started)を使用してAuth0との認証連携を設定します
src/auth.tsでの設定は以下のようになります。

```typescript
import NextAuth from 'next-auth'
import Auth0 from 'next-auth/providers/auth0'
import { Config } from './config'

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [Auth0({
    clientId: Config.auth0ClientId,
    clientSecret: Config.auth0ClientSecret,
    issuer: `https://${Config.auth0Domain}/`,
    authorization: {
      params: {
        audience: Config.appApiAudience
      }
    },
  })],
  callbacks: {
    jwt({ token, account }) {
      if (account?.access_token) {
        token.accessToken = account.access_token
      }
      return token
    },
    session: async ({ session, token }) => ({
      ...session,
      accessToken: token.accessToken
    }),
  },
})
```

ここで注意してほしいのが、`authorization.params.audience`の設定です。
Auth0では、audienceパラメータを指定することで、特定のAPI用のアクセストークンを取得できます。
この設定がない場合、Auth0はpayload部分を空にしたアクセストークンを発行します。
今回の実装はアクセストークンにpayloadが含まれており、その中にユーザーIDが存在する前提で実装しています。
そのため、`authorization.params.audience`の設定は必須です。
audienceを指定することで、Auth0はJWT形式のアクセストークンを発行し、それを自分のアプリケーションで検証できるようになります。

そして、Auth.jsのコールバック機能を使用して、アクセストークンをセッションに追加します。
jwtコールバックでアカウント情報からアクセストークンを取得し、sessionコールバックでセッションにアクセストークンを追加します。

```typescript
callbacks: {
  // アカウント情報からアクセストークンを取得
  jwt({ token, account }) {
    if (account?.access_token) {
      token.accessToken = account.access_token
    }
    return token
  },
  
  // セッションにアクセストークンを追加
  session: async ({ session, token }) => ({
    ...session,
    accessToken: token.accessToken
  }),
}
```

これにより、サーバーコンポーネントやサーバーアクションでauth()を呼び出すと、session.accessTokenとしてアクセストークンが取得できます。
なお、この実装は以下のドキュメントを参考にしておりますので、合わせて確認いただけますと幸いです。
https://authjs.dev/guides/integrating-third-party-backends
アプリ側の実装は以上ですが、補完が効くように、typesディレクトリを作成し、以下のようなglobal.d.tsファイルを作成します。

```typescript
// types/global.d.ts
declare module 'next-auth' {
  interface Session {
    accessToken?: string;
  }
}
```

これで、TypeScriptの型チェックも正しく機能します。
なので、auth()関数で取得した値にて、accessTokenプロパティの補完が効きます。

### ブロック通知受信APIの実装

次に、Workerからのブロック通知を受信するAPIエンドポイントを実装します。
src/app/api/user-block/route.tsでの実装は以下のようになります。

```typescript
import { jwtVerify, createRemoteJWKSet } from 'jose'
import { Redis } from '@upstash/redis'
import { Config } from '@/config'

const redis = Redis.fromEnv()

interface BlockNotification {
  token: string
}

export async function POST(request: Request) {
  // 1. リクエストボディをパース
  let body: BlockNotification
  try {
    body = await request.json() as BlockNotification
  } catch (error) {
    console.error('Failed to parse block notification payload', error)
    return Response.json({ message: '無効なリクエストです' }, { status: 400 })
  }

  if (!body?.token) {
    return Response.json({ message: 'トークンが見つかりません' }, { status: 400 })
  }

  // 以降の処理...
}
```

Workerから送信されるリクエストは、tokenというプロパティを持つJSON形式です。このトークンを検証し、ブロック情報をRedisに保存します。

### JWT検証とUpstash Redisへの保存

Workerから送信されたJWTを検証します。
joseライブラリのcreateRemoteJWKSet関数を使用することで、WorkerのJWKSエンドポイントから公開鍵を自動的に取得し、署名を検証できます。
なので、今回はそのライブラリを活用しました。

```typescript
// 2. WorkerのJWKSを使ってJWT検証
const { payload } = await jwtVerify(
  body.token,
  createRemoteJWKSet(new URL('/.well-known/jwks.json', Config.userBlockAppUrl))
)
```
公開鍵を使用して、検証が成功したら、ペイロードから情報を取得し、Redisに保存します。

```typescript
// 3. ペイロードからuser_idとblockedを取得
const userId = typeof payload.user_id === 'string' ? payload.user_id : undefined
const isBlocked = payload.blocked === true
const expiresAtSeconds = typeof payload.exp === 'number' ? payload.exp : undefined

if (!userId || !expiresAtSeconds) {
  return Response.json({ message: 'トークンの内容が不正です' }, { status: 400 })
}

// 4. Upstash Redisにブロック状態を保存
if (isBlocked) {
  await redis.set(userId, true)
  return Response.json({ message: 'ユーザーをブロックしました' }, { status: 200 })
}

await redis.set(userId, false)
return Response.json({ message: 'ユーザーのブロックを解除しました' }, { status: 200 })
```

RedisのキーとしてユーザーIDを使用し、値としてtrue（ブロック中）またはfalse（ブロック解除）を保存します。
Upstash RedisのRedis.fromEnv()は、環境変数UPSTASH_REDIS_REST_URLとUPSTASH_REDIS_REST_TOKENを自動的に読み取ります。
これにより、明示的に接続情報を渡す必要がなく、コードがシンプルになります。

### 保護されたAPIエンドポイントでのブロックチェック

最後に、実際のAPIエンドポイントでブロック状態をチェックする実装を見ていきます。
src/app/api/sample/route.tsでの実装の大枠は以下のようになります。

```typescript
import { jwtVerify, createRemoteJWKSet, errors } from 'jose'
import { Redis } from '@upstash/redis'
import { Config } from '@/config'

const redis = Redis.fromEnv()

export async function GET(request: Request) {
  // 処理の実装
}
```
GET関数内での具体的な処理は以下の通りです。
まず、リクエストヘッダーからアクセストークンを取得します。

```typescript
// 1. Authorizationヘッダーを確認
const headerValue = request.headers.get('Authorization') || request.headers.get('authorization')

if (!headerValue?.startsWith('Bearer ')) {
  return Response.json({ message: 'ログインしていません' }, { status: 401 })
}

const accessToken = headerValue.slice('Bearer '.length).trim()

if (!accessToken) {
  return Response.json({ message: '不正なトークンです' }, { status: 401 })
}
```

取得したトークンをAuth0のJWKSで署名の検証をし、値についても有効期限がきれていないかなどをチェックします。
Auth0はデフォルトでアクセストークンにユーザーIDはありますが、ユーザーIDがもしないと今回の記事の内容は上手くいかないので、チェックしています。

```typescript
try {
  // 2. Auth0のJWKSを使ってトークン検証
  const domain = Config.auth0Domain
  const audience = Config.appApiAudience

  const { payload } = await jwtVerify(
            accessToken,
            createRemoteJWKSet(new URL(`https://${Config.auth0Domain}`, `/.well-known/jwks.json`)),
            {
                issuer: `https://${domain}/`,
                audience
            }
        )

  // 3. ペイロードからユーザーID（sub）を取得
  const userId = typeof payload.sub === 'string' ? payload.sub : undefined
  
  if (!userId) {
    return Response.json({ message: 'トークンの内容が不正です' }, { status: 401 })
  }

  // 以降の処理...
} catch (error) {
  console.error('Token verification failed', error)

  if (error instanceof errors.JWTExpired) {
    return Response.json({ message: 'トークンの有効期限が切れています' }, { status: 401 })
  }

  return Response.json({ message: '不正なトークンです' }, { status: 401 })
}
```
以上の検証が成功したら、Redisでブロック状態を確認します。

```typescript
// 4. Upstash Redisでブロック状態をチェック
const isBlocked = await redis.get(userId)

if (Boolean(isBlocked) === true) {
  return Response.json({ message: 'ユーザーはブロックされています' }, { status: 403 })
}

return Response.json({ message: 'ログイン済みです', token: payload }, { status: 200 })
```

ブロックされている場合は403 Forbiddenを返し、そうでない場合は正常にレスポンスを返します。
この実装により、Auth0でユーザーがブロックされると、既存のアクセストークンを持っていてもAPIへのアクセスが拒否されるようになります。

## 補足：非同期処理について考える

ここまで、シンプルなWebhook方式でブロック通知システムを構築してきました。
しかし、本番運用を考えると、非同期処理の導入も検討が必要かなと思います。
今回の構成でいうと、Auth0 Event StreamからCloudflare Workerへの通知部分が非同期処理の候補になります。
それぞれについて軽く見ていきます。

### Auth0とAWS EventBridgeの連携（SQS + Lambda）

Auth0のEvent Streamsは、[AWS EventBridgeとの連携もサポートしています](https://auth0.com/docs/ja-jp/customize/events/create-an-event-stream#aws-eventbridge)。
![](/images/notify-block-user-by-auth0-event-stream/image-select-aws.png)
EventBridgeを使用する場合、Auth0 Event StreamからAmazon EventBridgeにイベントが送信されます。
そこからSQS（メッセージキュー）を経由してLambda（処理実行）で処理を行い、最終的にNext.js APIやRedisにデータを保存するというアーキテクチャになります。
SQS（Simple Queue Service）を使用することで、イベントを一時的にキューに保持できます。
これにより、イベントの取りこぼしを防ぐことができ、バッチ処理が可能になり、リトライ機能による耐障害性も向上します。
Webhook方式だと、エラーが起きた時のリトライや障害対応が難しいですが、この構成であればより確実にユーザーがブロックしたことを通知できるかなとおもいます。
ただ、これを用意するのが結構大変そうだったので、この記事を作るにあたっては構築していません。
### Cloudflare Queueを使った非同期処理の可能性
Cloudflareにも、[Queue](https://developers.cloudflare.com/queues/)という非同期処理のための機能があります。
Cloudflare Queueは、Worker間でメッセージを非同期に配信するための機能で、Producer Workerがキューにメッセージをsendし、Consumer Workerで処理を行うという流れになります。
Producer Workerでキューにメッセージを送信し、Consumer Workerで処理を行います。
実装イメージは以下のような形です。
```typescript
// Producer Worker
await env.MY_QUEUE.send({
  user_id: body.user_id,
  blocked: body.blocked
})

// Consumer Worker
export default {
  async queue(batch, env) {
    for (const message of batch.messages) {
      // メッセージを処理
    }
  }
}
```
こうすればWebhookとしては、キューにメッセージを送信することさえ担保できればよいので、イベントの取りこぼしを防ぎやすくなります。
また、Cloudflare QueueにはDLQ（デッドレターキュー）機能もあり、処理に失敗したメッセージを別のキューに移動させることができます。
リトライ処理もサポートされているはずなので、AWSで行うのと同様に、イベントの取りこぼしを防ぎ、耐障害性を向上させることが可能だと感じています。
ただし、Cloudflare Queueには最低月5ドルの固定費がかかります。
今回はあくまでAuth0のEvent Streamsの検証が主題なので、コスト面的に実装するのは控えました。
## まとめと今後の拡張可能性

### このシステムの利点
今回構築したシステムには、いくつかの利点があるとおもっています。
まず、リアルタイムなユーザーブロック検知が実現できます。
Auth0でユーザーをブロックした瞬間に、全てのアプリケーションサーバーに通知されるため、既存のセッションを持つユーザーであっても、即座にアクセスを拒否できます。
Auth0のエンタープライズプランであれば、Back-channel Logout機能も利用できますが、一度発行したJWT形式のアクセストークンは無効化はできないと理解しています。（ここは実際に試していないので、もし違っていたら教えてください）
一方今回の実装では、アプリケーションサーバー側でブロック状態をリアルタイムに把握できるため、ブロックしたらすぐにアクセス拒否が可能です。
もちろん、アプリ側にブロック情報を持っておき、それを使用する処理を書かないといけないという手間はあります。
とはいえ、アクセストークンの無効化ということを考慮して有効期限などを決めなくてよいのは利点だと感じています。
もう一つの利点として、ユーザーの扱いをAuth0にかなり寄せられる点があります。
アプリとAuth0の間にCloudflare Workersはありますが、Workersはあくまでもデータの中継と変換を行うだけの役割です。
そのため、ユーザーに関するロジックについてはAuth0側に集約できます。
なので、別途ユーザー情報を管理する必要がなく、Auth0の管理画面からユーザーの状態を一元的に把握できます。
この管理のしやすさは結構大きいメリットだなと感じています。

### 今後の拡張可能性
個人的に感じている、今回の内容をさらに発展させるためのアイデアを記載します。
まず、Worker側でのblockedプロパティ変化検知フィルタリングを追加できます。
現在の実装では、全てのuser.updatedイベントを処理していますが、実際にはblockedプロパティの変化のみを検知すれば十分です。
以下のようなフィルタリングロジックを追加できます。

```typescript
// Worker側でのフィルタリング例
const body: UserUpdateEvent = jsonData.data.object

// blockedプロパティが含まれている場合のみ処理
if ('blocked' in body) {
  // JWT生成と通知処理
}
```
またキャッシュ機能を追加し、前回の状態と比較して変化があった場合のみ通知するようにすれば、無駄な通知を削減できます。
そして、今回の実装はユーザー全体のブロック制御に焦点を当てていますが、場合によっては特定のアプリだけユーザーのアクセスを制御したいこともあります。
その場合は、user_metadataを使った特定アプリのみのブロック制御が考えられます。
Auth0のuser_metadataを使用することで、より細かいアクセス制御が可能になります。例えば、以下のようなメタデータを設定します。

```json
{
  "user_metadata": {
    "blocked_apps": ["app1", "app2"]
  }
}
```

Worker側では、このメタデータを含めてJWTを生成します。

```typescript
const token = await signJwt(
  privateKey,
  {
    user_id: body.user_id,
    blocked: body.blocked,
    blocked_apps: body.user_metadata?.blocked_apps || []
  },
  { expiresInSeconds: 300 }
)
```

Next.js側では、アプリケーションIDとblocked_appsを比較して、特定のアプリのみをブロックできます。
これにより、ユーザーを完全にブロックするのではなく、特定の機能やアプリケーションのみをブロックすることができます。
もちろん、この拡張は別途アプリの識別子をどうするかなどの考慮は必要になるので、必須の拡張かは微妙です。
その他にも、ログの監視などできること・やらないといけないことは色々ありますが、一旦ここまでにしておきます。

## おわりに
今回は、Auth0のEvent Streamsを使用して、ユーザーブロックをリアルタイムに検知するシステムを構築しました。
このシステムにより、アクセストークンの有効期限問題を解決しつつ、セキュリティを大幅に向上させることができます。
実装自体はシンプルですが、意外と考えることが多く大変でした。
そして、Auth0のEvent Streamsって便利だなとも感じました。
なのでぜひ皆さんのプロジェクトでも試してみてください。
そして、あわよくば今回と似たような構成でAWSでの設定方法を教えてもらえると嬉しいです。
ここまで読んでいただきありがとうございました。