---
title: "Device Bound Session Credentialsをユーザー認証に組み込んでみた（検証版）"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DeviceBoundSessionCredentials","Auth","Session","Cookie"]
published: true
---
## はじめに
最近Xにて、証券系のサイトのセッションの扱いが話題になっていました。
そのやりとりを見て、Device Bound Session Credentials（以降DBSC）の緩和策として使えるのではないかと感じていました。
今回は上記の思いが、本当にあっているかを自分の中で確かめるために、ログイン後のフローにDBSCを組み込んでみたという記事です。
もし、DBSCが気になっている方の目に止まれば幸いです。
なお、DBSCについてや試しにDBSCを組み込んでいく方法については特に記載しません。
そちらについては、以下の仕様書や、過去の記事などを参照ください。
https://developer.chrome.com/docs/web-platform/device-bound-session-credentials?hl=ja
https://w3c.github.io/webappsec-dbsc/#device-bound-session-expiration-timestamp
https://zenn.dev/maronn/articles/program-dbsc-app
https://zenn.dev/maronn/articles/about-dbsc-infomation
## 実装の前に
実装する前に、今回の記事を書くきっかけの投稿をどういう軸で見ていくかを記載します。
当該投稿をこの記事に対して都合が良いように以下の解釈をします。※
- 証券サイトが短時間で再認証を求める。
- そのため、ユーザーがフィッシングサイトであったとしても、ユーザー情報を入力してしまう可能性が上がる。
- であれば、例えば100日はセッションを維持し続ける方が良い。

つまり、短いサイクルで再認証を求めることで認証疲れを引き起こすくらいなら、長いセッションの方がましと解釈します。
認証サイクルの長い・短いどちらが良いかの対立があると投稿を捉えています。
そして、DBSCはこの対立をなくし、セッションは短いサイクルで更新されるけれど、ユーザーは何度もログイン操作が必要なくなるのではないかと考えました。
結論として、一部考慮すべきことはあるが、達成できそうという感覚です。
この点については、実装を踏まえて感じたことを改めて書きます。
では、先に進みます。
:::message
※あくまで自分の都合の良いように解釈しています。
例えば、投稿内容のセッションを長く張り続けるという内容ですが、ここでは同じセッションを100日張るかのように解釈しています。
ただ、実際にそうだと投稿者は断言していません。
というより、裏でユーザーが操作せずともセッションの更新は行いつつも、100日は再認証を求めないDBSCな挙動を想定されているように見受けられます。
ただ、今回は当該投稿を最初にみた印象をもとに解釈したことを記載しています。
投稿者の本意ではなく、私独自の解釈がふんだんに入っていることはご認識お願いします。
::::

## コード全体
以下のブランチにて、今回実装したものを置いています。
https://github.com/maronnjapan/sample-id-app/tree/check-session-dbsc
その中で、個人的なポイントだけ書いておきます。
といっても、DBSC本体部分は[以前の記事](https://zenn.dev/maronn/articles/program-dbsc-app)で書いているので、あまりその点については触れません。
### Cookieの値やSession Identifierの保存方法
発行したCookieやCookieとSession Identifierの紐づけは、[UpstashのRedis](https://upstash.com/docs/redis/overall/getstarted)を使っています。
なので、ローカルで開発する時は[@updatash/redis](https://www.npmjs.com/package/@upstash/redis)をインストールし、以下のようにDocker Composeを書いています。
```yaml
services:
  postgres:
    image: postgres:latest
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: dbsc
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
  redis:
    image: redis
    ports:
      - "6379:6379"
  serverless-redis-http:
    ports:
      - "8079:80"
    image: hiett/serverless-redis-http:latest
    environment:
      SRH_MODE: env
      SRH_TOKEN: example_token
      SRH_CONNECTION_STRING: "redis://redis:6379"
volumes:
  postgres_data:
    driver: local
```
Redisのイメージに加えて、将来的にデプロイした際もUpstash Redisを使えるようにするために[serverless-redis-http](https://github.com/hiett/serverless-redis-http)を設定しています。
PostgreSQLはユーザー情報の保存などに使うので、そのままにしています。
これが設定できたら、以下のようにRedisを使用するオブジェクトを定義します。([in-memory.ts](https://github.com/maronnjapan/sample-id-app/blob/check-session-dbsc/src/storage/in-memory.ts))
```tsx
import { Redis, SetCommandOptions } from "@upstash/redis";

const redis = new Redis({
    url: process.env.UPSTASH_REDIS_REST_URL,
    token: process.env.UPSTASH_REDIS_REST_TOKEN,
})

export const InMemoryDB = {
    async get(key: string): Promise<string | null> {
        return await redis.get(key)
    },
    async set(key: string, value: string, opts?: SetCommandOptions): Promise<void> {
        await redis.set(key, value, { ex: 60 * 5 });
    },
    async del(key: string): Promise<void> {
        await redis.del(key);
    },
    async exists(key: string): Promise<boolean> {
        const result = await redis.exists(key);
        return result > 0;
    }
};
```
これで、諸々のID情報を保存できるようにしました。
### 各種値の紐づけ
[/api/login/route.ts](https://github.com/maronnjapan/sample-id-app/blob/check-session-dbsc/src/app/api/login/route.ts#L32C5-L47C1)で、ログインした時に発行したCookieにユーザー情報を紐づけます。
```tsx
const cookieStore = await cookies();
const authValue = randomUUID();

/** Device Bound Session Credentialsが失敗したとき用に生存期間の長いCookieを設定 */
cookieStore.set("auth_cookie", authValue, {
  domain: "localhost",
  sameSite: "lax",
  path: "/",
  maxAge: 2592000,
});
/**
 * Cookieの値をInMemoryDBに保存
 * Device Bound Session Credentialsのために、user IDと紐づけて保存
 * 後で、Device Bound Session Credentialsのフローで発行したCookieやsession identifierと差し替える
 */
await InMemoryDB.set(authValue, user.id, { ex: 60 * 60 * 24 * 30 });
```
基本的にはDBSCが成功したタイミングで、紐づけは変更しますが、ユーザーを識別できるようにするために値をセットしておきます。
その後、DBSCのフローを始める際に、以下のようにCookie→DBSCが発行するSession Identifier→ユーザー情報と紐づけるようにします。
```tsx
  /**
  * 有効期限が短いCookieをセットし直す
  * 今回は検証しやすいように10秒で設定しているが、仕様に明確な秒数は指定されていない
  * ただし、長すぎるとセキュリティ上の問題があるため、短い時間で設定することが推奨されている
  * */
    const newCookieValue = randomUUID();
    cookieList.set('auth_cookie', newCookieValue, {
        maxAge: 10,
        sameSite: 'lax',
        path: '/'
    })

    /**
     * 新しいCookieの値をInMemoryDBに保存
     * Device Bound Session Credentialsで発行したsession IDとユーザーIDを紐づけて保存
     * ただ、session identifierが更新のタイミングでしか取得できない。
     * そのため、Cookieの値をキーにして、session identifierを取り出せるようにする
     * 以上より、Cookie→ session identifier → user ID の順で紐づける
     * また、本来はCookieとSession Identifierの紐づけは更新のタイミングで明示的に削除を行うべきである。
     * しかし、今回はそこまでの実装は行わないので、15秒で有効期限を設定していて無効化の対応をしている
     */
    await InMemoryDB.set(newCookieValue, sessionId, { ex: 15 });
    await InMemoryDB.set(sessionId, userId, { ex: 60 * 60 * 24 * 30 });
```
間にSession Identifierを挟むのは、DBSCを始めたことを示す識別子がSession Identifierだからです。
DBSCにおいて、リフレッシュ処理が走るのはCookieの有効期限が切れたタイミングになります。
そのため、Cookieとユーザー情報を紐づけると、更新のタイミングで、ユーザー情報と紐づいていた値が分からなくなります。
なので、設定したDBSCを識別できるSession Identifierを間にかませることで、更新のタイミングに対象のユーザーとの紐づけをなくさずに済みます。
紐づけをなくさずに済む実装については、[/api/refresh-dbsc-cookie/route.ts](https://github.com/maronnjapan/sample-id-app/blob/check-session-dbsc/src/app/api/refresh-dbsc-cookie/route.ts#L100C5-L115C1)を参照ください。
個人的ポイントは以上かと思います。
実装は色々と書いてありますが、今回紹介した部分と/api配下を見ていただければ、今回やりたかったことが掴めるかなと思います。
## 実装してみて感じたこと
個人的に実装して、動かして感じたことを書きます。
大きくは以下の二点です。
- 短いサイクルでCookieは更新できるが、ユーザー体験は良さそう
- ブラウザを閉じた時の挙動が悩ましい

まず、ユーザー体験の方です。
結構驚いたのですが、DBSCは同一ブラウザであればタブが異なってもリフレッシュが行われます。
![](/images/user-login-and-dbsc/dbsc-anther-tab-refresh-demo.gif)
上記のように、仮にDBSCで作成したCookieが有効期限切れになっても、同一ブラウザであればリフレッシュのエンドポイントが叩かれます。（Chrome環境での検証結果）
であれば、仮にログインしてしまったタブを閉じたとしても、再認証をせずにアプリケーションを使用し続けられそうです。
これは個人的に一番感動した挙動でした。
ユーザビリティがかなり良いと思います。
このように、短いサイクルでキーとなるCookieを裏で更新しつつも、ユーザーにログインを求める必要がないのは、気になっていたことが解消されると感じます。
なので、件の投稿の課題を解消するのにDBSCはかなり良いアプローチになりそうです。
一方で、ブラウザを閉じた時については考慮が必要です。
ブラウザを閉じて、再度ブラウザを開きアプリにアクセスしてもリフレッシュエンドポイントは実行されません。（Chrome環境での検証結果）
挙動に違和感はないですが、今回のようにCookieの有効期限が切れたら、再度認証が必要になりそうです。
今回話題になった証券系サイトなどであれば、ブラウザを開いている間はずっと操作できるけれど、ブラウザを閉じたら再認証を求めるのは受け入れられるかなと思います。
一方で、一般的なアプリでその挙動だとユーザーにはログイン疲れを引き起こしそうで、ちょっと考えものです。
一応、DBSCが失敗した時用に、ログインした時の有効期限が長いCookieを持つことはできるので、回避策はあります。
実際に[Chromeのブログ](https://developer.chrome.com/docs/web-platform/device-bound-session-credentials?hl=ja#caveats_and_fallback_behavior)には上記Cookieを用いた挙動が言及されています。
ただ、ブラウザを開き直した時にそのCookieをただ使うようにすると、せっかくDBSCでセキュアにしている利点が大分薄れてしまいます。
なので、長期間のCookieはかなり改ざん検知やログインしたユーザーと一致するCookieしか受け入れないようにするセキュアな仕組みが必要となります。
今回の実装ではそこまでやっていないのですが、こういった再度アクセスした時の挙動については、慎重になる必要があるなと感じています。
後、長期間のCookieを使うとして、もう一回DBSCのフローを始める機能も搭載しておく必要がありそうですね。
このように、DBSCを設定できれば多くの利点を得られそうですが、DBSC開始前、終了後をどうするかを慎重に検討する必要がありそうです。
そういったことをしないと、せっかくのDBSCの利点が薄れてしまうので。
とはいえ、やはりDBSCはロマンを感じるので、これからどんどん標準化が進んでいって欲しいです。
## おわりに
DBSCはロマンの塊！