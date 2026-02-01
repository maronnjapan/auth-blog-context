---
title: "MCPのAuthorizationについてのプルリクエストを眺めてみて"
emoji: "💨"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
先日以下のプルリクエストがクローズされました。
https://github.com/modelcontextprotocol/modelcontextprotocol/pull/284
[MCPのAuthorization](https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization)についての改定で、様々な議論がされていました。
その中で今回はDPoPについての議論と個人的に感じたことを書きます。
## DPoPとその周辺の概念について
DPoPは受け取ったアクセストークンが正当な持ち主からのものかをチェックするための仕組みです。
詳しいことは以下の川崎さんの記事ありますので、そちらを参照ください。
https://qiita.com/TakahikoKawasaki/items/34c82fb5c0595b6fc289
DPoPの詳細はこの記事に委ねるとして、ここで注目したのがDPoPによって発行されるトークンは**Sender-Constrained**というものになることです。
Sender-Constrainedと対になるのが、Bearerかなと思います。
このあたりについても、以下の記事を参考にいただけますと幸いです。
https://sizu.me/ritou/posts/emomf80fe2w6
とにかく、有効なトークンなら誰が使ってもOKにするか、ちゃんと然るべき所有者しか使えないようにするかが肝かなと思っています。
## プルリクエストを眺めつつ感想を書いていく
ではプルリクエストを眺めていって感想を書いていきます。
先に結論としては以下のようになります。
「時代の流れとしてSender-Constrainedが求められるように感じているが、それを全ての人に求めるのは厳しいよね。でもSender-Constrained自体は推し進められるだろうからいつでも対応できる必要はある。」
まずプルリクエストの説明文を見ると以下DPoPの記載があります。
> **Need for DPoP** - DPoP (Demonstrating Proof-of-Possession) prevents token theft attacks by cryptographically binding access tokens to the client that originally requested them, ensuring that even if tokens are intercepted, they cannot be used by attackers from different locations or devices. Implementing DPoP in client applications creates a stronger security posture against various OAuth 2.0 threats like token replay attacks, as each API request includes a unique, signed proof that only the legitimate token holder can generate, significantly reducing the risk surface even if TLS is compromised.
> 

雑にまとめるとセキュリティ面の向上を求めるために、DPoPの実装をクライアントに求めてはどうかという感じですかね。
実際プルリクエスト内の以下部分に、DPoPの記載がありました。
https://github.com/modelcontextprotocol/modelcontextprotocol/pull/284/files#diff-961c36b26a515b33b6cda2907f55dc53504999d9775e090f789d310617f45efaR137
「MAY」になっているとはいえ、クライアント側の実装としてDPoPも選択肢として提示されています。
最近[Googleがオリジントライアルとしてテスト](https://developer.chrome.com/blog/dbsc-origin-trial?hl=ja)できるようにした[Device Bound Session Credentials(DBSC)](https://w3c.github.io/webappsec-dbsc/)も、Sender-Constrained的な形でCookieを扱うことが出てきていることから、昨今とてもホットなMCPにもそれを求めるのは自然だなと感じました。
MCPの仕様として、よりセキュアな選択肢を書くことに対して特に違和感はありませんでした。
ただ、[プルリクエスト内のコメント](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/284#issuecomment-2786460148)を見ていくと以下の記載がありました。
> 1. Remove DPoP: This is too much of a heavy lift for clients and I would need to be presented with very strong arguments to buy into adding this to the spec. Generally I am already unhappy how much is required by nature of the space (e.g. dynamic client registration, etc), so unless it really adds a lot, i would prefer if we keep it out. I know the intentions are right here, but i like to meet people where they are. People are free to use additional OAuth extensions internally.
> 

DPoPについては削除するとのことでした。
理由として、DPoPの実装を仕様に書き、それを求めることは開発者にとって相当な負担になるとのことです。
このコメントもすごく納得感がありました。
セキュリティ向上のために、DPoPは選択肢としてありですが、例えば鍵の管理とかトークンと紐づける実装とか考えることが多くて、現状自分としては実装できるイメージが湧いていないです。
川崎さんのブログ内の画像を見ても、クライアント含めやること多くて大変そうですね。
![2025-05-03_11h42_48.png](/images/read-about-mcp-authorization/2025-05-03_11h42_48.png)
これらから、DPoPの記述は無くすとのことで、実際今回見ていたプルリクエスト内に書かれた[別のプルリクエスト](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/338)についてはDPoPの記載がなかったです。
やっぱりSender-Constrainedってセキュアですけど、やること・やらないといけないことが増えるのでしんどいんですよね。
## でもSender-Constrainedへは進んでいく
今回DPoPを実装者に求めるのは負担が大きいということで、ドキュメントへの記載は見送られていました。
この決定自体は納得感があり、コメントにもあったDPoPを求めるならそれ相応の理由が必要というのも違和感はありません。
ただ、MCPという範囲を超えるとDPoPみたいに正当な所有者しか使用を禁止する機能を搭載する流れはこれからますます進んでいくと思います。
昨今推進が勧められて認証方式である[パスキー](https://fidoalliance.org/passkeys/?lang=ja)やセッション管理の[Device Bound Session Credentials(DBSC)](https://w3c.github.io/webappsec-dbsc/)も、デバイスやパスワードマネージャーなどに紐づく場合しか使えないようにしています。
上記は認証方式やセッション管理の話なので、DPoPとは少しことなるかもしれません。
ただ、これらは知っている・持っているだけでは使えないようにする点で共通です。
この使える人・モノを限定する流れは、今後主流になっていく可能性はかなり高いと思っていまし、実際パスキーは新しい認証方式として導入が推進されています。
今すぐにSender-Constrainedの波が来るわけではありませんが、今後波が来ることは間違いないと考えているので、流されないように備えはしておきたいです。
こんなことをMCPのプルリクエストを眺めてみて感じました。
以上、とりとめのない話で恐縮ですが、Sender-Constrainedの波を感じていただけますと幸いです。