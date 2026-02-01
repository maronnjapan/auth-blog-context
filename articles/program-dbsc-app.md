---
title: "Device Bound Session Credentials(DBSC)をNext.jsで実装して検証してみた"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DeviceBoundSessionCredentials","DBSC","Next.js","Cookie"]
published: true
---
## はじめに
先日以下のツイートを見かけました。
https://x.com/agektmr/status/1915182340319699097
以前[記事](https://zenn.dev/maronn/articles/about-dbsc-infomation)も書いており、密かに待ち望んでいたDevice Bound Session Credentials(以降DBSC)のオリジントライアルが始まったとのことです。
なので、早速実装してみたのがこの記事となります。
なお、発表したのはオリジントライアルですが、試したのはローカル環境となります。
オリジントライアルしているのに、デプロイしてないのかいという点はご容赦いただけますと幸いです。
また、DBSCを完璧に理解している人間が書いたものではないので、誤り・不足があるかと思います。
その際はコメントいただけますと大変助かります。
## DBSCについて
:::message alert
注：あまりDBSCを私の中で咀嚼しきれていない箇所があります。
なので、記事一番下の[参考資料](https://zenn.dev/maronn/articles/program-dbsc-app#%E5%8F%82%E8%80%83%E8%B3%87%E6%96%99)にて概要を把握いただけますと幸いです。
:::
実装を見ていく前に、DBSCの概要について触れていきます。
DBSCは仕様を確認すると、以下の記載があります。
> Device Bound Sessions Credentials (DBSC) aims to prevent hijacking via cookie theft by building a protocol and infrastructure that allows a user agent to assert possession of a securely-stored private key.
> 

ユーザーエージェントが持っている安全な秘密鍵の所有を証明できるプロトコルとインフラを構築することで、Cookie Theftによるハイジャックを防止するものみたいです。
ログインして、サーバーでセッション張ってはい終わりってわけではなく、セッションをデバイスに紐づけることで固有のブラウザのみセッションを張れるようにし、それによってCookie Theftを予防できるそうです。
フロー全体は以下の通りです。
![[**Device Bound Session Credentials explainer](https://github.com/w3c/webappsec-dbsc?tab=readme-ov-file#high-level-overview) より引用**](/images/program-dbsc-app/image.png)
[**Device Bound Session Credentials explainer](https://github.com/w3c/webappsec-dbsc?tab=readme-ov-file#high-level-overview) より引用**
色々とフローが書いてありますが、着目するのはセッションを張る前に必ずデバイスのTPMに存在する秘密鍵で署名したJWTを検証し、それがOKであることをチェックしている点です。
これによって、異なるデバイスでも同じセッションを設定するという行為が出来なくなります。
また、TPMはデバイスにあるものなので、仮に秘密鍵を抜き取って悪用するマルウェアに感染させようにも端末でマルウェアを動かす必要があるため、検知などがしやすくなります。
すごいザックリとした説明で恐縮ですが、とりあえずアプリケーションのセッション管理にデバイス内の秘密鍵を紐づけることができるようになったというのがDBSCの一番の肝かと個人的には思います。
それでは実際に動かしてみて、DBSCを体感しようと思います
## 事前準備
まず、ローカルでDBSCを動かすための設定をします。
ここが分からなくて、結構苦労しました。
結論正解は[chrome://flags/#enable-standard-device-bound-session-credentials](chrome://flags/#enable-standard-device-bound-session-credentials)を`Enabled - Without Origin Trial tokens`とし、[chrome://flags/#enable-standard-device-bound-sesssion-refresh-quota](chrome://flags/#enable-standard-device-bound-sesssion-refresh-quota)を`Disabled`とする形でした。
画像としては、以下の通りです。
![2025-04-28_11h06_30.png](/images/program-dbsc-app/2025-04-28_11h06_30.png)
なお、[アナウンス記事](https://developer.chrome.com/blog/dbsc-origin-trial?hl=ja)では`chrome://flags/#device-bound-session-credentials`　とありますがこれでアクセスしても当該部分は見つけることはできないです。
設定としては、[こちらのイシュー](https://github.com/w3c/webappsec-dbsc/wiki/Testing-early-versions-of-DBSC)を参考にして行うと上手く行きました。
では早速実装の中身を解説します。
## DBSCを実装したコードを見る
:::message
2025/07/25時点の仕様では、DBSCの各種ヘッダー名の接頭辞が`Secure-`に変更されています。  
ですが、2025/07/25においてChromeでは、ヘッダー名が`Secure-`では動かず、`Sec-`でないと動きません。    
将来的には、`Secure-`になっていくと思いますが、この記事では動くことを優先して`Sec-`を用いて実装しています。  
:::
コード全体は以下リポジトリにあります。
https://github.com/maronnjapan/sample-id-app/tree/no-authorization-dbsc
全体像や動き自体は上記リポジトリをクローンしてもらうとして、この記事ではDBSCに関わるコードを見ていきます。
前提として、DBSCを行うにはアプリケーション側にも追加で実装が必要です。
具体的には以下の内容です。
- セッションを張るタイミングで、`Sec-Session-Registration`レスポンスヘッダーを付与する
- TPMによって署名されたJWSを検証し、新規で有効期限が短いCookieを発行するエンドポイント
- Cookieの有効期限が切れたときに、再度Cookieを発行するエンドポイント

これらは各種アプリケーションにないと動きません。
余談ですが、[Googleの記事](https://developer.chrome.com/docs/web-platform/device-bound-session-credentials?hl=ja#:~:text=%E6%97%A2%E5%AD%98%E3%81%AE%E3%82%A8%E3%83%B3%E3%83%89%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88%E3%81%AE%E3%81%BB%E3%81%A8%E3%82%93%E3%81%A9%E3%81%AF%E5%A4%89%E6%9B%B4%E4%B8%8D%E8%A6%81%E3%81%A7%E3%81%99%E3%80%82DBSC%20%E3%81%AF%E3%80%81%E8%BF%BD%E5%8A%A0%E5%9E%8B%E3%81%A7%E4%B8%AD%E6%96%AD%E3%81%AE%E3%81%AA%E3%81%84%E6%96%B9%E6%B3%95%E3%81%A7%E8%A8%AD%E8%A8%88%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%81%BE%E3%81%99%E3%80%82)で既存のアプリケーションを大きく変えることなく、DBSCは搭載できるとありましたが、まさにその通りだと思います。
基本的に、エンドポイントやヘッダーの追加だけで、動かせそうなので導入する際に実装の難しさはそれほどないと感じています。
今回は以下のようにDBSCに特化したエンドポイントを作っていますが、既存のアプリケーションに組み込む時も同じような形になるかと思います。
![2025-05-05_12h28_15.png](/images/program-dbsc-app/2025-05-05_12h28_15.png)
各種エンドポイントについて見ていきます。
### start-dbsc-flow
DBSCの利用開始ブラウザ側に伝えるエンドポイントです。
以下のように、アプリケーションサーバーで認証を行った際、このセッションはDBSCを使用することをブラウザ側に伝えます。
![[**Device Bound Session Credentials explainer](https://github.com/w3c/webappsec-dbsc/blob/main/README.md#high-level-overview) より引用**](/images/program-dbsc-app/2025-05-05_13h26_25.png)
[**Device Bound Session Credentials explainer](https://github.com/w3c/webappsec-dbsc/blob/main/README.md#high-level-overview) より引用**
ブラウザ側はこの開始の合図をもって後続の内容を実行します。
エンドポイントの実装は以下の通りです。
```tsx
export async function GET() {
    const cookieList = await cookies()
    const challenge = randomUUID()

    /** 後続のフローでSec-Session-Responseのpayloadに含まれるjtiと一致するかを確認するためにchallengeをDBに保存する。 */
    await prisma.challenge.create({
        data: {
            challenge
        }
    })

    /** 
     * DBSCが失敗した場合に使用する長期間保持できるCookie 
     * 参考資料(https://developer.chrome.com/docs/web-platform/device-bound-session-credentials?hl=ja#caveats_and_fallback_behavior)
     * ただし、DBSCを進めるために必須のものではない
     * */
    cookieList.set('auth_cookie', 'cookie', {
        domain: 'localhost',
        sameSite: 'lax',
        path: '/',
        maxAge: 2592000,
    })

    /**
     * DBSCを開始するための値
     * 最初に署名に使用する鍵のアルゴリズム
     * pathに署名したトークンを渡すエンドポイントの設定
     * トークンを識別するためのchallenge
     * challengeはデバイスで署名したJWTのjtiに使用される
     * https://w3c.github.io/webappsec-dbsc/#header-sec-session-registration
     */
    const secSessionRegistration = `(ES256 RS256);path="register-dbsc-cookie";challenge="${challenge}"`

    return NextResponse.json({}, {
        /** DBSCを開始するために、レスポンスに「Sec-Session-Registration」を含める*/
        headers: {
            "Sec-Session-Registration": secSessionRegistration,
        }
    })
}
```
本来は認証用のエンドポインですが、サンプルなので認証の判定などは行っていません。
エンドポイントを実行すれば、等しく認証できたとみなします。
重要なのは、`Sec-Session-Registration`ヘッダーに指定する値です。
ここでは最低限以下の設定を行います。
- 署名を行う鍵のアルゴリズム
- 署名したトークンを渡すエンドポイントのパス
- チャレンジ

具体的な形式については、[仕様](https://w3c.github.io/webappsec-dbsc/#header-sec-session-registration)の確認をお願いします。
このレスポンスヘッダーをブラウザに渡すことで、デバイスで秘密鍵・公開鍵が作られ、公開鍵を含んだJWSを含んだリクエストがDBSCを開始するエンドポイントに行きます。
なお、鍵の作成・保存からエンドポイントまでのリクエストは、Chromeが自動で行ってくれます。
アプリケーションとして意識する必要がないのは大変ありがたいですね。
### register-dbsc-flow
デバイスから公開鍵を受け取り、セッションをスタートさせるエンドポイントです。
以下のように、ブラウザから公開鍵を受け取りそれをセッションと紐づけつつ保存し、セッションを張るようにします。
![[**Device Bound Session Credentials explainer](https://github.com/w3c/webappsec-dbsc/blob/main/README.md#high-level-overview) より引用**](/images/program-dbsc-app/2025-05-05_13h56_29.png)
[**Device Bound Session Credentials explainer](https://github.com/w3c/webappsec-dbsc/blob/main/README.md#high-level-overview) より引用**
ここで重要なのは、まず公開鍵を保存することです。
この後の更新エンドポイントで受け取る公開鍵はここでの公開鍵と同じものになります。
なので、公開鍵が一致するかを検証するためにもこの値は保持しておく必要があります。
また、ここでのフローはCookieを再度発行するのですが、このCookieの有効期限は短くする必要があります。
DBSCは有効期限を短くしてもユーザビリティを損なわない仕組みなので、Cookieの安全性を高めるためにも有効期限は短くすることが求められます。
発行したCookieに対するCookie Theftを防ぐ仕組みはDBSCには定義されていないように感じます。
[イシュー](https://github.com/w3c/webappsec-dbsc/issues/136#issuecomment-2806070969)にも、上記のような記載があります。
> Theft of the short-term cookie (or signature) isn't prevented, that's correct.
> 

ただ、発行する際にはデバイスに紐づいた形でしかできないようになっているので、有効期限を短くし、頻繫に発行・更新を行うことがDBSCを使う際はもとめられるのかなと思います。
ポイントは記載したので、実装を確認します。
```tsx
export async function POST(req: NextRequest) {
    /** 公開鍵やチャレンジの値を含むJWSが設定されているヘッダーから値を取得 */
    const token = req.headers.get("Sec-Session-Response")
    if (!token) {
        return NextResponse.json(JSON.stringify({ message: 'Unauthorized' }), {
            status: 401,
            statusText: 'Unauthorized',
        })
    }

    /** トークン内の公開鍵を使用して、署名を検証する */
    const isValidPublicKey = await verifySecSessionResponseToken(token)
    if (!isValidPublicKey) {
        /** 署名が無効な場合は401エラーとする */
        return NextResponse.json(JSON.stringify({ message: 'Unauthorized' }), {
            status: 401,
            statusText: 'Unauthorized',
        })
    }

    const [_, payload] = token.split('.')
    const payloadJson = JSON.parse(base64UrlDecode(payload))
    const sessionId = randomUUID()
    const publicKey = getPublicKey(payload)

    /** DBに保存されているchallengeと一致するかを確認する */
    /** challengeはデバイスで署名したJWTのjtiに使用される */
    await prisma.challenge.findUniqueOrThrow({
        where: {
            challenge: payloadJson.jti
        }
    })

    /** 
     * DBにセッションIDとそれに紐づく公開鍵を保存する
     * 更新エンドポイントが実行された時、公開鍵が同じものかを検証するために使用
     */
    await prisma.dbscSession.create({
        data: {
            sessionId,
            publicKey
        }
    })

    const cookieList = await cookies()
    /** 
     * 有効期限が短いCookieをセットし直す 
     * 今回は検証しやすいように10秒で設定しているが、仕様に明確な秒数は指定されていない
     * ただし、長すぎるとセキュリティ上の問題があるため、短い時間で設定することが推奨されている
     * */
    cookieList.set('auth_cookie', randomUUID(), {
        maxAge: 10,
        domain: 'localhost',
        sameSite: 'lax',
        path: '/'
    })
    
    return NextResponse.json({
        "session_identifier": sessionId,
        /** セッションを更新する際にブラウザが実行するエンドポイントを設定 */
        "refresh_url": "/api/refresh-dbsc-cookie",
        "scope": {
            "origin": "http://localhost:3000",
            "include_site": true,
            "scope_specification": [
            ]
        },
        "credentials": [{
            "type": "cookie",
            "name": "auth_cookie",
            "attributes": "Domain=localhost; SameSite=Lax; Path=/"
        }]
    }
    )
}
```
受け取ったJWS形式のトークンを内部の公開鍵で検証し、問題なければ更新エンドポイント用にセッションIDと紐づけた形で公開鍵を保存します。
保存したら、有効期限の短いCookieを発行し、レスポンスの値に登録したセッションの種類を示す値をJSON形式で返却します。
`session_identifier`は後続の更新エンドポイントを叩く際に、対象のセッションを見つけるために使用されます。
`refresh_url`はCookieの有効期限が切れた際に実行される更新エンドポイントを設定します。
`scope`と`credentials`については、あまりちゃんと理解できていないのでスルーします。すみません…。
なお、DBSCとは関係ないですがトークン内の公開鍵を使用して署名を検証する処理についても記載しておきます。
```tsx
/**
 * 文字列をWeb Crypto APIで使用するArrayBufferに変換する
 */
export const stringToArrayBuffer = (str: string) => {
    const buffer = new ArrayBuffer(str.length);
    const bufferView = new Uint8Array(buffer);
    for (let i = 0; i < str.length; i++) {
        bufferView[i] = str.charCodeAt(i);
    }
    return buffer;
}
export const base64UrlDecode = (str: string) => {
    /**  URLで使用可能な形式にエンコードされた文字列を元のBase64で使用される文字列に置換 */
    const replaceStr = str.replace(/-/g, '+').replace(/_/g, '/');
    /**
     * Base64の文字数が4の倍数になるようにパディング文字列を追加
     * パディング文字列は'='である
     * パディングについての記事
     * https://qiita.com/yagaodekawasu/items/bd8a1db4529cfc921bba
     */
    const padding = Array(replaceStr.length * 8 % 6).map(() => '=').join('');
    return atob(`${replaceStr}${padding}`);
}

export const getPublicKey = (payload: string) => {
    return JSON.parse(base64UrlDecode(payload)).key as {
        "crv": "P-256",
        "kty": "EC",
        "x": string,
        "y": string
    }
}

export const verifySecSessionResponseToken = async (token: string) => {
    const [header, payload, signature] = token.split('.')

    const publicKey = getPublicKey(payload)
    const keyOption = {
        name: 'ECDSA',
        namedCurve: "P-256",
        hash: 'SHA-256'
    }
    const encoder = new TextEncoder()
    /**
     * Web Crypto APIで使用するための公開鍵をインポート
     * 署名検証に使用するために、公開鍵をインポートする
     * https://developer.mozilla.org/ja/docs/Web/API/SubtleCrypto/importKey
     */
    const publicKeyData = await crypto.subtle.importKey('jwk', publicKey, keyOption, true, ['verify']);
    /**
     * 署名を検証
     * 署名はURLのクエリで使用可能な形式にエンコードされているので元のバイナリに戻す
     * 署名に使用したheaderとpayloadを結合したものははURLのパスで使用可能な形式にエンコード後、
     * TextEncoderのencodeでArrayBufferViewに変換されたものとなっている。
     * なので、検証に使うときは単純にTextEncoderのencodeでArrayBufferViewに変換したものを使う
     */
    return await crypto.subtle.verify(keyOption, publicKeyData, stringToArrayBuffer(base64UrlDecode(signature)), encoder.encode(`${header}.${payload}`))
} 
```
コードは[src/utils.ts](https://github.com/maronnjapan/sample-dbsc-app/blob/main/src/util.ts)に書いています。
### refresh-dbsc-cookie
Cookieが有効期限切れになった後、ブラウザが自動でリクエストをするエンドポイントです。
先程の登録エンドポイントで、セッション情報をブラウザに渡しているため、クライアント側で何かする必要はないです。
フローは以下の部分です。
![[**Device Bound Session Credentials explainer](https://github.com/w3c/webappsec-dbsc/blob/main/README.md#high-level-overview) より引用**](/images/program-dbsc-app/2025-05-05_15h54_23.png)
[**Device Bound Session Credentials explainer](https://github.com/w3c/webappsec-dbsc/blob/main/README.md#high-level-overview) より引用**
結構更新エンドポイントはやることが多いです。
まず、最初は`Sec-Session-Id`ヘッダーに登録したセッションIDが保持された状態でブラウザからリクエストされます。
最初にリクエストされるときは、基本的にCookieの有効期限が切れたタイミングなので、401エラーを返します。※
しかし、ただ401エラーを返すのではなく、`Sec-Session-Challenge`レスポンスヘッダーにチャレンジとチャレンジ対象のsessionIDを記載した形で返します。
これによって、ブラウザ側はデバイスが持っている鍵を使用して署名付きのトークンを作成し、それを更新エンドポイントに再度リクエストする処理を行います。
再度更新エンドポイントにリクエストが来たタイミングで、登録の時に行ったのと同じような検証を行い、問題が無ければ再度Cookieを発行します。
ただし、検証は署名が有効なだけでなく、トークンに含まれている公開鍵が登録の時と同じかも判定する必要があります。
それらを含めた実装が以下の通りです。
```tsx
export async function POST(req: NextRequest) {
    const token = req.headers.get("Sec-Session-Response")
    const sessionId = req.headers.get("Sec-Session-Id")
    const cookieList = await cookies()

    if (!token) {
        const challengeValue = randomUUID()
        const challenge = `"${challengeValue}";id="${sessionId}"`
        /**
         * 更新エンドポイントはリトライされるのが前提
         * リトライされたタイミングで、チャレンジの一致を検証するために保存
         */
        await prisma.challenge.create({
            data: {
                challenge: challengeValue
            }
        })
        
        /**
         * 2025/05/05時点では、401エラーがリトライのトリガーになる
         * が、リトライのトリガーが401エラーから403エラーに変更となるプルリクエストがマージされている
         * https://github.com/w3c/webappsec-dbsc/pull/141
         * なので、将来的には403エラーを返すようにしないと動かなくなる可能性がある
         */
        return NextResponse.json(JSON.stringify({ message: 'Unauthorized' }), {
            status: 401,
            statusText: 'Unauthorized',
            headers: {
                "Sec-Session-Challenge": challenge,
            }
        })
    }

    const isValidPublicKey = await verifySecSessionResponseToken(token)
    if (!isValidPublicKey) {
        return NextResponse.json(JSON.stringify({ message: 'Unauthorized' }), {
            status: 401,
            statusText: 'Unauthorized',
        })
    }

    const [_, payload] = token.split('.')
    const publicKey = getPublicKey(payload)
    const payloadJson = JSON.parse(base64UrlDecode(payload))

    await prisma.challenge.findUniqueOrThrow({
        where: {
            challenge: payloadJson.jti
        }
    })

    /** セッションの存在確認 */
    const session = await prisma.dbscSession.findUnique({
        where: {
            sessionId: sessionId
        }
    })

    const publicKeyX = (session?.publicKey as {
        x: string,
        y: string
    })?.x
    const publicKeyY = (session?.publicKey as {
        x: string,
        y: string
    })?.y

    /**
     * 登録時に受け取った公開鍵と一致するかを確認する
     * 一致しない場合、異なるデバイスからのリクエストとなるはずなので、エラーとする
     */
    if (!session || publicKeyX !== publicKey.x || publicKeyY !== publicKey.y) {
        return NextResponse.json(JSON.stringify({ message: 'Unauthorized' }), {
            status: 401,
            statusText: 'Unauthorized',
        })
    }

    /** 
     * 有効期限が短いCookieをセットし直す 
     * 今回は検証しやすいように10秒で設定しているが、更新の場合も仕様に明確な秒数は指定されていない
     * ただし、長すぎるとセキュリティ上の問題があるため、短い時間で設定することが推奨されている
     * */
    cookieList.set('auth_cookie', randomUUID(), {
        maxAge: 10,
        domain: 'localhost',
        sameSite: 'lax',
        path: '/'
    })

    return NextResponse.json({
        /** 
         * session_identifierは登録時と同じにすること
         * 同じにしないと登録したセッションと異なるようになってしまい、繰り返し更新してくれなくなる。
         * */
        "session_identifier": sessionId,
        "refresh_url": "/api/refresh-dbsc-cookie",
        "scope": {
            "origin": "http://localhost:3000",
            "include_site": true,
            "scope_specification": [
            ]
        },
        "credentials": [{
            "type": "cookie",
            "name": "auth_cookie",
            /** 
             * このattributesはかならずセットしたCookieと同じ設定にする 
             * 同じにしないと、更新エンドポイントをブラウザが無限に叩き続ける
             * */
            "attributes": "Domain=localhost; SameSite=Lax; Path=/"
        }]
    })
}
```
注意点として、`session_identifier`は登録時と同じ値にする必要があります。
異なる値にすると、Cookieが再度有効期限切れになっても更新処理が上手く走ってくれません。
また、credentialsのattributesは必ずセットしたCookieと同じ設定にしてください。
これをしないとおそらくブラウザ側が、新しくセットしたCookieが更新処理によるものなのか上手く判別ができず、無限に更新エンドポイントを叩きつづけます。
以上までが実装されていれば、セッションの登録から更新がいい感じに動くのではないかと思います。
セッションの停止についてはできていないので、折を見てやっていけたらなと思います。
:::message
※現在は401エラーを返していますが、以下プルリクエストがマージされたため、将来的に403エラーが再度リクエストを行うために必要なエラーになりそうです。
https://github.com/w3c/webappsec-dbsc/pull/141
ただ、まだ2025/05/05時点ではChromeが対応していないので、アプリケーションのコードとしては401エラーを返すようにしています。
:::
## 実装した感想 兼 おわりに
最後にDBSCを実装した感想を軽く書きます。
個人的には、もっとちゃんとした実装例が増えればスタンダードなものになるかなと感じています。
今回いい感じのサンプルがないので、実装に苦労しましたが終わってみればそこまで既存のアプリケーションへの導入に大きな障壁はなさそうでした。
基本追加実装だけで問題ないので、短いCookieの扱いをどうするかを各プロジェクトが考えないといけないですが、それ以外はサンプルがあればそこまで大変なものではなさそうです。
IDaaSがまずDBSCを課金機能として導入し、そこから徐々に広まっていくのかなと妄想しています。
また、DBSCは秘密鍵をデバイスに保存し、それを署名に使用する形ですが鍵の扱いについては全く意識する必要がありませんでした。
これはありがたい反面、ちゃんと動いているのかが少し不安になります。
やはりもうちょっとサンプルというか実例出てきてほしいなと感じます。
後は、DBSCのフローで発行した有効期限が短いCookie自体は盗まれたら使いまわせると思っているのですが、その辺の理解はあっているのか不安に思っています。
結構APIを実行するための判定材料として核となる部分なので、有効期限が短いとはこのCookieが盗まれると結局Cookie Theftはされてしまう気がしています。
なので、DBSCはCookie Theftの防止策というよりは緩和策くらいのイメージになってしまうのですが、あっているのか不安に感じています。
ここは特にちゃんと知りたいなので、調査できたらなと思います。
以上理解がふわっとしている部分はありつつも、将来を感じさせる機能でした。
ここまで読んでいただきありがとうございました。
## 参考資料
https://developer.chrome.com/blog/dbsc-origin-trial?hl=ja
https://developer.chrome.com/docs/web-platform/device-bound-session-credentials?hl=ja
https://github.com/w3c/webappsec-dbsc
https://w3c.github.io/webappsec-dbsc/