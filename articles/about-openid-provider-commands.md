---
title: "OpenID Provider Commandsの仕様をざっくり読んだので、ざっくり感想を書く"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenID"]
published: true
---
## はじめに
先日以下の投稿を見かけました。
https://x.com/phr_eidentity/status/1898168504488259676
どうやらOpenID Connect周りで、OpenID Provider Commandsという新しい仕様が考えられているという内容でした。
そして、上記ツイート内の記事には仕様のリンクがあったので、ざっとですが読んでみました。
今回はざっと読んだ感想を中心に書いていきます。
正直この記事からの学びはないので、概要を知りたい方は以下のリンクから仕様に遷移していただけますと幸いです。
https://dickhardt.github.io/openid-provider-commands/openid-provider-commands-1_0-00.html
## そもそもどんな感じの仕様？
OpenID Provider （以降OP）からRelying Party（以降RP）に個別のアカウントやテナント内のアカウントの状態を変更するリクエストの形式を決めた仕様みたいです。
個人的には、上記機能をOpenID Connectという概念の中に取り込んだことが大きな特徴かと感じました。
似たような仕組みとして、Twitterで共有されていた記事では、[SCIM](https://scim.cloud/)があると言及していました。
ただ、SCIMでやろうとするとOPの他に別途SCIM用のサーバーを立てる必要があります。
認証や認可の一連の流れをOPで行い、ユーザーへの操作はSCIMで行う流れとなります。
これは手間である一方で、今回の仕様はOP内で完結するので、管理するサーバーは少なくなりそうです。
ざっくりした概要や利点を見たので、リクエスト内容も一部確認します。
リクエストの値は仕様には以下のような例が載っていました。
```json
{
  "iss": "https://op.example.org",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "iat": 1734003000,
  "exp": 1734003060,
  "jti": "bWJq",
  "command": "activate",
  "given_name": "Jane",
  "family_name": "Smith",
  "email": "jane.smith@example.org",
  "email_verified": true,
  "tenant": "ff6e7c9",
  "groups": ["b0f4861d", "88799417"]
}
```
なお、上記はあくまでpayload部分のみの例であり、実際には署名などをした上でRPに渡す必要があります。
例で特に着目するところはcommand部分かと思います。
これが対象のアカウントをどのような状態にするかを定義しているので、個人的にはこの仕様の肝だと思っています。
Twitterで共有されている記事の中では、例えば`unauthorize`というコマンドを渡すことで、対象のアカウントのログイン状態を取り消すことができる旨の記載がありました。
## テナントレベルでの操作
OpenID Provider Commandsはアカウントの状態を設定できますが、アカウント単位だけでなくテナント単位でも指定できるのも特徴かなと感じます。
リクエストの例としては以下の例がありました。
```json
{
  "iss": "https://op.example.org",
  "aud": "s6BhdRkqt3",
  "iat": 1734006000,
  "exp": 1734006060,
  "jti": "bWJt",
  "command": "metadata",
  "tenant": "ff6e7c96",
  "metadata": {
    "groups": [
      {
        "id": "b0f4861d",
        "display": "Administrators"
      },
      {
        "id": "88799417",
        "display": "Finance"
      }
    ],
    "domains": ["example.com"],
    "claims_supported": [
      "sub",
      "email",
      "email_verified",
      "name",
      "given_name",
      "family_name",
      "groups"
    ]
  }
}
```
アカウント単位と異なるのは、subクレームの代わりにtenantクレームが必須になることと、metadataが必須なことです。
metadataの中には、対象のテナント内でコマンドを実行して欲しいアカウントをgroupsクレームで指定できるみたいです。
またテナントの場合の特徴として、リクエスト方式があります。
アカウント単位ではREST APIを使用したPOSTリクエストでのやり方が書いてありました。
一方、テナント単位では[Server-Side Events](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events) (SSE)を使用したやり取りを想定して書かれています。
そのため、リクエストBodyだけでなく、途中で通信が切れてしまった時の再開方法についても書かれています。([こちら](https://dickhardt.github.io/openid-provider-commands/openid-provider-commands-1_0-00.html#section-6.4-12)を参照)
なぜテナントはSSE前提なのかについての記載は仕様の中に見つけることができませんでした。
結構気になっているので、もし理由をご存知の方は教えていただければ幸いです。
## コマンドトークンにはnonceを含めない
[仕様](https://dickhardt.github.io/openid-provider-commands/openid-provider-commands-1_0-00.html#section-4-9)の中で個人的に目をひいたのは以下の部分です。
>
> The following Claim MUST NOT be used within the Command Token:[¶](https://dickhardt.github.io/openid-provider-commands/openid-provider-commands-1_0-00.html#section-4-9)
> 
> - **nonce**PROHIBITED.A `nonce` Claim MUST NOT be present. Its use is prohibited to prevent misuse of the Command Token. See [Cross-JWT Confusion](https://dickhardt.github.io/openid-provider-commands/openid-provider-commands-1_0-00.html#cross-jwt-confusion) for details.

クレームの中にnonceを含めてはいけないと書いてあります。
これはJWTを別の目的で使用するCross-JWT Confusionを防ぐためのようです。
確かに、今回のケースではnonceの概念は不要ですし、IDトークンとの混同を避けるために明記するのはなるほどと思いました。
## 全体的な感想
最後に全体的な感想を書いていきます。
ざっと読んでまず思ったのが、[Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html)に雰囲気が似ているなと思いました。
Back-Channel Logoutはログアウトに特化した機能ではありますが、同じようにログアウト用のトークンをRPにリクエストするので、とても似ているなと感じています。
ただ、OpenID Provider Commandsの方は他の様々な状態を定義できるので、よりできることが多いのかなという印象はあります。
今後、OpenID Provider Commandsの仕様が確定したら、Back-Channel Logoutの扱いがどうなっていくは気になっています。
実務でBack-Channel Logoutを触ったことはないので、実際は棲み分けが起きるかもしれないですが、それ含めて方向性を確認出来たらなと思います。
後は、RP側の実装大変そうだなという印象です。
ただ、OpenID Provider Commandsが要因で大変そうというよりは、そもそもOPがトークンを発行した後も管理していくのが大変ということだとは思います。
とはいえ、ログインはアプリケーションとしては始まりにすぎないので、ログイン後もユーザーの管理は必要となるからこそ、こういった仕様が生まれていくのだとも感じています。
どこかのタイミングで使ってみたり、実装してみたりしたいですね。
以上です。
ありがとうございました。
