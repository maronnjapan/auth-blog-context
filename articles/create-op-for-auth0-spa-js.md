---
title: "Auth0提供のライブラリで構築されたSPAと自作OPで、トークンが発行されるまでの流れを実装する"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Auth0","SPA","Javascript","OpenID Connect","OAuth2"]
published: true
published_at: 2024-12-23 08:00
---
この記事はDigital Identity技術勉強会 #iddance Advent Calendar 2024 23日目の記事となります。
https://qiita.com/advent-calendar/2024/iddance
## 結論
大人しく、Auth0提供のライブラリを使用するときはAuth0を使用しましょう。
## 出来上がったアプリケーションの挙動
![spa-oidc-demo.gif](/images/create-op-for-auth0-spa-js/spa-oidc-demo.gif)
## はじめに
世に存在するIDaaSの一つに、[Auth0](https://auth0.com/jp)があります。
このAuth0ですが、多くのライブラリも提供しています。
その中の一つに、[auth0-spa-js](https://github.com/auth0/auth0-spa-js)が存在します。
今回は、上記ライブラリを使用して、OpenID ConnectのAuthorization Code Flowを構築しているSPAのOPを、自作アプリにして動かしてみようという記事です。
## 記事の内容に入る前に
この記事は/authorizeへのアクセスから、受け取ったID Tokenの検証を完了するという流れを完了させるということを第一としています。
そのため、以下の要件が達成できておりません
- 認可コードを渡す方法がクエリかフラグメントかを分けるための判定
- スコープのチェック
- 同意・ユーザー認証画面の搭載
- Authorization Code Flow以外のフローへの対応
- 適切なアクセストークンの発行
- リフレッシュトークンの対応
- stateやnonceなどが使用できる有効期限の設定
- コンフィデンシャルクライアントによるリクエスト
- エラーの判定と、適切なエラーコード・メッセージの送信
- …etc

このように、対応できていないことが数多あります。
そのため、自分でやると色々と考えることが多くて大変そうだなということだけ伝わればと思います。
## クライアントの作成
以下のQuickStartを元に、アプリケーションを作成します。
https://auth0.com/docs/quickstart/spa/react/01-login
Reactで作成していますが、以下[Auth0が想定しているアプリケーション](https://auth0.com/docs/quickstart/spa)であればどれでも大丈夫なはずです。
![image.png](/images/create-op-for-auth0-spa-js/image.png)
上記アプリケーションによるクイックスタートはhttps://github.com/auth0/auth0-spa-jsをラップしたライブラリを使用しているためです。
今回はReactのクイックスタートを元に、App.tsxに以下の記載をしています。
```
import React from 'react';
import { useAuth0 } from '@auth0/auth0-react';
function App() {
  const { isLoading, isAuthenticated, error, user, loginWithRedirect, logout } =
    useAuth0();
  if (isLoading) {
    return <div>Loading...</div>;
  }
  if (error) {
    return <div>Oops... {error.message}</div>;
  }
  if (isAuthenticated) {
    return (
      <div >
        Hello {user.name}
      </div>
    );
  } else {
    return <button onClick={() => loginWithRedirect()}>Log in</button>;
  }
}
export default App;
```
そして、それを以下のように呼び出しています。
```bash
import React from "react";
import ReactDOM from "react-dom/client";
import "./index.css";
import App from "./App";
import { Auth0Provider } from "@auth0/auth0-react";
const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <React.StrictMode>
    <Auth0Provider
      domain="identity-provider-by-next-js.vercel.app"
      clientId="d654d2fc-118b-8592-020a-f5b13c4eafbe"
      authorizationParams={{
        redirect_uri: "http://localhost:3000",
      }}
    >
      <App />
    </Auth0Provider>
  </React.StrictMode>
);
```
これでクライアント側の準備は完了です。
## クライアント側がどのようなリクエストを飛ばすかを確認
OPの実装に入る前に、クライアント側がどのようなリクエストを飛ばすかを確認します。
以下の画像から、クライアント側はトークンを受け取るために、赤枠部分のリクエストをOPに対して行っていることが分かります。
![2024-12-21_17h00_31.png](/images/create-op-for-auth0-spa-js/2024-12-21_17h00_31.png)
[Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow) より引用(一部画像を改変しました)
では、これら二つにおけるリクエストの値を見ていきます。
まずは②の/authorizeにアクセスする時の値です。
![2024-12-21_17h08_09.png](/images/create-op-for-auth0-spa-js/2024-12-21_17h08_09.png)
画像に「Query String Parameters」と書かれている通り、URLクエリとして設定します。
クエリを確認する限り、client_id、response_typeなど必須のクエリに加え、state・nonce・code_challengeなどより安全にするためのクエリも設定されていそうです。
なので、トークンの発行を完了するにはstateやnonceによる値比較、PKCEの流れも搭載する必要があります。
なお、auth0ClientについてはAuth0がログを取るために設定している独自のクエリなので、Authorization Code Flowには一切関係がないです。
次に⑥の認可コードを元に、トークンを取得するためのリクエスト値を確認します。
![2024-12-21_17h13_17.png](/images/create-op-for-auth0-spa-js/2024-12-21_17h13_17.png)
トークンを取得するリクエストはPOSTなので、Form Dataになっていますね。
stateについては、[こちら](https://openid-foundation-japan.github.io/rfc6749.ja.html#CSRF)の記載があるように、~~認可コードをOPからもらった後、適切なクライアントかを判定~~認可レスポンスを受け付けるにあたって適切なセッションかどうかを判定するために使用します。
そのため、トークンを取得するためのリクエストにstateは含めません。
PKCEについては、OP側での検証となるのでリクエストに含まれています。
その他、[仕様書](https://openid-foundation-japan.github.io/rfc6749.ja.html#token-req)に記載がある通り、必須のcodeとgrant_typeが存在します。
さらに、今回は/authorizeへのアクセスにredirect_uriを設定しているのと、クライアント認証が完了していないので、client_idとredirect_uriも設定されています。
以上クライアントからのリクエストについて確認しました。
それでは、実際にOPを作成していきます。
## OPの作成
### コード全体
以下のリポジトリに全体のコードは記載しております。
[https://github.com/maronnjapan/identity-provider-by-next-js](https://github.com/maronnjapan/identity-provider-by-next-js)
### 使用技術
フレームワーク
- Next.js(App Router)

ストレージ
- [Upstash For Redis](https://upstash.com/)

デプロイ先
- Vercel

とにかくサクッと構築できることを第一としたので、上記にしました。
### /authorizeの構築
まずはクライアントが認可リクエストを行うためのURLを用意します。
app/authorizeにpage.tsxを作成し、以下のコードを記載します。
```bash
import { getClientById } from "@/lib/services/client.service";
import { isBadRequestQuery, OptionalAuthorizeQuery, RequiredAuthorizeQuery } from "./_validate/check-bad-request";
import { redirect } from "next/navigation";
import { randomUUID } from "crypto";
import { storeAuth } from "@/lib/services/auth.service";
import { storeCode } from "@/lib/services/token.service";
export default async function Page({ searchParams }: { searchParams: Promise<RequiredAuthorizeQuery & OptionalAuthorizeQuery> }) {
    const { client_id, response_type, redirect_uri, scope, state, nonce, code_challenge, code_challenge_method, audience } = await searchParams
    const client = getClientById(client_id);
    if (!client) {
        return <div>
            <p>不正なURLです</p>
        </div>
    }
    if (isBadRequestQuery({ client_id, response_type, redirect_uri, scope, state, nonce, audience, code_challenge, code_challenge_method })) {
        return <div>
            <p>不正なURLです</p>
        </div>
    }
    const code = randomUUID()
    await storeCode(code)
    const redirectUrlQuery = new URLSearchParams({ state, code }).toString()
    const redirectUrl = redirect_uri ?? client.allowRedirectUrls[0]
    const codeChallengeObj = code_challenge && code_challenge_method ? { code_challenge, code_challenge_method } : undefined
    await storeAuth(code + client.clientId, { clientId: client.clientId, isPublishIdToken: !!scope?.includes('openid') && response_type === 'code', nonce, codeChallengeObj })
    return redirect(redirectUrl + `?${redirectUrlQuery}`)
```
ユーザー認証や同意画面への遷移は考えていないので、クエリが全て適切であればクライアントにリダイレクトするようにしています。
リダイレクトするURLは、クエリに値があればそれを使用し無ければ予め登録したクライアントが持つ最初のURLを使用するようにしました。
後は、トークンの発行の際に使用する認可コード(code)、PKCE関連のcode_challenge・code_challenge_method、nonceを永続化しています。
永続化が完了したら、トークンの発行リクエストに必要な認可コードと認可コードの横取りを防ぐためにクライアントが使用するstateをクエリとして付与したURLにリダイレクトします。
ちなみに、クエリのチェックを行っているisBadRequestQueryの実装は以下の通りです。
```bash
import { getClientById } from "@/lib/services/client.service";
import { Permutation } from "@/utils/util-type";
export type RequiredAuthorizeQuery = {
    client_id: string;
    response_type: 'code';
    state: string
}
export type OptionalAuthorizeQuery = {
    scope?: string;
    audience?: string;
    redirect_uri?: string;
    nonce?: string;
    code_challenge?: string;
    code_challenge_method?: 'S256' | 'plain';
}
const requiredQueryNames: Permutation<keyof RequiredAuthorizeQuery> = ['client_id', 'response_type', 'state']
export const isBadRequestQuery = (query: RequiredAuthorizeQuery & OptionalAuthorizeQuery) => {
    const isExistRequiredQueries = requiredQueryNames.every(name => !!query[name]);
    if (!isExistRequiredQueries) {
        return true;
    }
    if (query.response_type !== 'code') {
        return true;
    }
    const client = getClientById(query.client_id)
    if (!client) {
        return true;
    }
    if (query.redirect_uri && !client.isAllowUrl(query.redirect_uri)) {
        return true;
    }
    return false
}
```
必須のクエリが存在するかをチェックし、存在する場合はclient_idに一致するクライアントが存在するかを確認しています。
そして、そのクライアントが持つリダイレクト先として許可しているURLがクエリの値と一致するかを確認しています。
また、今回はresponse_typeはcodeのみを許容しています。
本当はcode以外もあるのですが、auth0-spa-jsは[こちら](https://github.com/auth0/auth0-spa-js/blob/f2e566849efa398ca599daf9ebdfbbd62fcb1894/src/Auth0Client.ts#L203)に記載があるように、scopeに`openid` を含めることでID Tokenの発行を依頼しています。
なので、response_typeが`code`でかつ、scopeが`openid`を含む場合のみID Tokenを発行させるために、code以外を禁止としています。
これで、/authorizeのエンドポイントが作成できました。
次に、トークンの発行リクエストを受け付けるAPIを作成します。
### トークン発行エンドポイント
まず、auth0-spa-jsが想定しているトークン発行のエンドポイントを確認します。
[https://github.com/auth0/auth0-spa-js/blob/f2e566849efa398ca599daf9ebdfbbd62fcb1894/src/api.ts#L23](https://github.com/auth0/auth0-spa-js/blob/f2e566849efa398ca599daf9ebdfbbd62fcb1894/src/api.ts#L23)
上記ソースを見る限り、基本的に/oauth/tokenにリクエストを投げています。
なので、OPとしては/app/oauth/tokenでディレクトリを作成し、そこにroute.tsを作成すれば良さそうです。
ただ、Next.jsの都合上api系の処理はapp/api配下にまとめることが推奨されているので、今回はapp/api/oauth/tokenの階層でディレクトリを作りました。
そして、api用にroute.tsを作成しています。
このままではauth0-spa-jsからのリクエストを受け取ることができないので、next.config.tsで以下rewriteの設定を付与しています。
```bash
async rewrites() {
    return [
      {
        source: '/oauth/token',
        destination: '/api/oauth/token'
      }
    ]
  },
```
これでauth0-spa-jsからのリクエストも、自作OPが想定するエンドポイントを叩いてくれるようになります。
なお、next.config.tsにはrewriteの設定に加え、/api/oauth/tokenのエンドポイントは外部からのリクエストとなるので、以下のCORS対応も定義しています。
```bash
  async headers() {
    return [
      {
        source: "/api/oauth/token",
        headers: [
          { key: "Access-Control-Allow-Origin", value: "*" },
          { key: "Access-Control-Allow-Methods", value: "GET,OPTIONS,PATCH,DELETE,POST,PUT" },
          { key: "Access-Control-Allow-Headers", value: "*" },
        ]
      },
      {
        source: "/oauth/token",
        headers: [
          { key: "Access-Control-Allow-Origin", value: "*" },
          { key: "Access-Control-Allow-Methods", value: "GET,OPTIONS,PATCH,DELETE,POST,PUT" },
          { key: "Access-Control-Allow-Headers", value: "*" },
        ]
      }
    ]
  }
```
今回のケースでいうと/oauth/tokenだけで十分ですが、念のために/api/oauth/tokenも設定しています。
apiを呼ぶための準備はできたので、実際にAPIの実装を以下に展開します。
```bash
import { deleteAuth, getAuth } from "@/lib/services/auth.service";
import { getClientById } from "@/lib/services/client.service";
import { validateChallenge } from "@/lib/services/pcke.service";
import { getState } from "@/lib/services/state.service";
import { deleteCode, generateIdToken, IdTokenPayload, validCode } from "@/lib/services/token.service";
import { createHash } from "crypto";
import { NextRequest, NextResponse } from "next/server";
export async function POST(request: NextRequest) {
    const bodyText = await request.text()
    const bodyList = bodyText.split('&').map(item => item.split(/=(.+)/, 2))
    const body = bodyList.reduce<{ code: string, code_verifier?: string, client_id: string, grant_type?: 'authorization_code', redirect_uri?: string }>((acc, [key, value]) => {
      return { ...acc, [key]: value }
    }, { code: '', code_verifier: undefined, client_id: '', grant_type: undefined, redirect_uri: undefined });
    
    const { code, code_verifier, client_id, grant_type, redirect_uri } = body
    if (!code) {
        return NextResponse.json({ message: 'Bad Request ' }, { status: 400 })
    }
    const client = getClientById(client_id)
    if (!client) {
        return NextResponse.json({ message: 'Bad Request  ' }, { status: 400 })
    }
    const auth = await getAuth(code + client.clientId)
    if (!auth) {
        return NextResponse.json({ message: 'Bad Request  ' }, { status: 400 })
    }
    const digestBufferOrString = code_verifier && auth.codeChallengeObj?.code_challenge_method === 'S256' ?
        await crypto.subtle.digest(
            { name: 'SHA-256' },
            new TextEncoder().encode(code_verifier)
        )
        : code_verifier
    const digest = typeof digestBufferOrString === 'string' ? digestBufferOrString : bufferToBase64UrlEncoded(digestBufferOrString)
    const isValidCodeVerifier = digest ? await validateChallenge(digest) : true;
    const isValidCode = await validCode(code)
    const isValidClientId = client_id === auth.clientId
    const isValidGrantType = grant_type === 'authorization_code'
    const isValidRedirectUri = redirect_uri ? client.isAllowUrl(decodeURIComponent(redirect_uri)) : true
    if (!isValidCodeVerifier || !isValidCode || !isValidClientId || !isValidGrantType || !isValidRedirectUri) {
        return NextResponse.json({ message: 'Bad Request   ' }, { status: 400 })
    }
    deleteCode(code)
    deleteAuth(code + client.clientId)
    const apiUrl = new URL(request.url)
    const nonceObj = auth.nonce ? { nonce: auth.nonce } : {}
    const idTokenPayload: IdTokenPayload = {
        iss: `${apiUrl.origin}/`,
        sub: '1234567890',
        name: 'John Doe',
        email: 'john.doe@example.com',
        iat: Date.now(),
        exp: Date.now() + 3600,
        aud: auth.clientId,
        auth_time: Date.now(),
        ...nonceObj
    }
    const idToken = auth.isPublishIdToken ? generateIdToken(idTokenPayload) : undefined
    return NextResponse.json({ access_token: 'opaque', token_type: "Bearer", expires_in: 3600, id_token: idToken }, {
        status: 200, headers: {
            'Cache-Control': 'no-store',
            'Pragma': 'no-cache',
            'Content-Type': 'application/json',
        }
    })
}
const bufferToBase64UrlEncoded = (input?: ArrayBuffer) => {
    if (!input) {
        return undefined
    }
    const ie11SafeInput = new Uint8Array(input);
    return urlEncodeB64(
        btoa(String.fromCharCode(...Array.from(ie11SafeInput)))
    );
};
const urlEncodeB64 = (input: string) => {
    const b64Chars: { [index: string]: string } = { '+': '-', '/': '_', '=': '' };
    return input.replace(/[+/=]/g, (m: string) => b64Chars[m]);
};
```
それぞれのポイントは以下の通りです。
- リクエスト値の受け取り方
- 各種値に紐づくデータが存在するか
- code_verifierのハッシュ化とcode_challengeとの比較
- ID Tokenの発行

まず、リクエスト値ですが[OAuth2のRFC](https://openid-foundation-japan.github.io/rfc6749.ja.html#token-req)で記載があるように、Cotent-Typeは`application/x-www-form-urlencoded`となります。
そのため、リクエスト値を`await request.json()`で受け取るとエラーになります。
これを回避するために、まずtext()で値を受け取り、そこから以下のようにオブジェクトとしてつめなおしています。
```bash
const bodyText = await request.text()
const bodyList = bodyText.split('&').map(item => item.split(/=(.+)/, 2))
const body = bodyList.reduce<{ code: string, code_verifier?: string, client_id: string, grant_type?: 'authorization_code', redirect_uri?: string }>((acc, [key, value]) => {
   return { ...acc, [key]: value }
}, { code: '', code_verifier: undefined, client_id: '', grant_type: undefined, redirect_uri: undefined })
```
リクエスト値をオブジェクトに変換したら、その値を元に保存したデータと一致するものを取得しています。
そして、各種リクエスト値が許可している値であるかを比較しています。
なお、code_verifierについてはcode_challengeにした上で、保存したデータと一致しているかを確認する必要あります。
[RFC7636](https://www.rfc-editor.org/rfc/rfc7636)を確認すると、`code_challenge = BASE64URL-ENCODE(SHA256(ASCII(code_verifier)))` という関係性であることが分かります。
そのため、以下の関数で受け取ったcode_verifierを上記を満たすために変換しています。
```bash
const bufferToBase64UrlEncoded = (input?: ArrayBuffer) => {
    if (!input) {
        return undefined
    }
    const ie11SafeInput = new Uint8Array(input);
    return urlEncodeB64(
        btoa(String.fromCharCode(...Array.from(ie11SafeInput)))
    );
};
const urlEncodeB64 = (input: string) => {
    const b64Chars: { [index: string]: string } = { '+': '-', '/': '_', '=': '' };
    return input.replace(/[+/=]/g, (m: string) => b64Chars[m]);
};
```
これで生成された値を、code_challengeと比較することでPKCEの処理が完了します。
各種値の検証が通ったら、保存したデータを削除し、トークンの作成を行います。
今回は、アクセストークンをリソースサーバーが使用することまでは考えないので、アクセストークンについては無意味な文字列を渡しています。
後は、[openid-connect-core-1.0](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#TokenResponse)が定めているように、有効期限を示すexpires_inとtoken_typeとid_tokenを渡しています。
id_tokenの形式については、[openid-connect-core-1.0](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#IDToken)が定めているものに従って、生成しています。（auth0-spa-js用に、余分なものもつけてはいますが…)
なお、署名の検証については[opeid-connect-core-1.0](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#IDTokenValidation)が言及しているように、TLS Serverの確認を署名の検証の代わりにできるという記載があります。
そして、auth0-spa-jsは以下の実装で、httpsでの処理しか認めないようにしています。
[https://github.com/auth0/auth0-spa-js/blob/f2e566849efa398ca599daf9ebdfbbd62fcb1894/src/utils.ts#L221](https://github.com/auth0/auth0-spa-js/blob/f2e566849efa398ca599daf9ebdfbbd62fcb1894/src/utils.ts#L221)
そのため、今回のケースでいうと署名はあってもなくても良いので、適当な鍵で署名したものを返しています。(実際には絶対にやってはいけないことですが、鍵の提供なども考えると果てしないことになりそうだったので、ご容赦ください。)
以上でトークンを発行する準備が完了です。
後は、冒頭のデモの操作を行うと/oauth/tokenのレスポンスに以下が表示されます。
![2024-12-22_14h27_40.png](/images/create-op-for-auth0-spa-js/2024-12-22_14h27_40.png)
各種プロパティと、クライアントが行うID Tokenの検証も無事に終わって、画面にはユーザー名が表示されます。
めでたしめでたしです。
ちなみに、ID Token内のissですが、末尾に`/`をつけないとエラーとなります。
ようやく出来たと思ったタイミングでの上記エラーなので、結構落ち込みました。
皆様もお気を付けください。
## 実際に作ってみてどうだったか
最後にここまで作ってみた感想を軽く書きます。
最初に思ったのは、ちゃんと実装するのはあまりにも難しいということです。
今回の記事でも全然仕様に沿えていない部分が多々あります。
如何に自分が感覚でOAuth2やOpenID Connectを捉えていたかを痛感しました。
OAuth2やOpenID Connectの基礎部分は、なにもわからないだと思っていたのですが、完全に理解したの状態でした。
https://note.com/teched/n/n555f4f5f9344
ただ、今回改めて実装してみて良かったと思います。
stateやPKCEの発行、検証タイミングやnonceの役割など、多くの学びを得たのは良かったです。
全てを勉強のために作るのは難しいので、こういったライブラリやIDaaSの一部代替という形で実装していこうと思いました。
最後に、OAuth2・OpenID Connect周りについてはIDaaSや実績あるOSSを使う必要性を改めて痛感しました。
考えることが多すぎるため、個人で作ったものはおろか、会社で作ったものですら独自のアプリケーションは漏れがないようにするのは相当困難ですね。
適切なツールの選択のために、知識は必要ですが実際の適用はIDaaSを第一に考え、厳しい場合は実績のあるOSSを選択していこうと思いました。
ここまで読んでいただきありがとうございました。