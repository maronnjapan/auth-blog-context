---
title: "なぜID Tokenにはsub Claimが必須なのか"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OIDC", "JWT", "IDToken", "sub"]
published: true
---

## はじめに

先日ふと ID Token について調べていました。
その ID Token で必須 Claim を見ていたのですが、sub クレームの存在理由があまり腑に落ちませんでした。
ID Token の使用用途を考えた時、sub 属性を活用する場面がないとその時は思ったためです。
その後、調べてみると活用方法があり、sub クレームの存在意義がある程度腑に落ちたため、今回記事を書きました。
まだまだ勉強中の身なので、間違い不足あればご指摘いただけますと幸いです。
なお、今回の記事は Authorization Code Flow の使用を前提として、記載しています。
その点もご承知おきください。

## ID Token について

### ID Token とは何か

sub クレームに行く前にそもそも ID Token とは何かについて説明していきます。※
※ここから先 ID Token について書いていきますが、正直[川崎さんの Qiita 記事](https://qiita.com/TakahikoKawasaki/items/8f0e422c7edd2d220e06)や[OpenID ファウンデーション提供の仕様書](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html)などを読んだ方が整理されており、かつ正確です。
なので、ここから先の記載は私の勉強ためという色合いが強いです。
ID Token とは[OpenID Connect Core 1.0](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#IDToken)を確認すると以下のように記載してあります。

> ID Token は, ある Client を利用するというコンテキスト中での, Authorization Server による End-User 認証に関する Claim を含んだセキュリティトークンである.

以下のように認証・認可の流れを完了した際、認可サーバーから ② を行ったユーザー情報を含めたトークンをもらうのですが、そのトークンのことを ID Token と言います。
![[https://qiita.com/TakahikoKawasaki/items/4ee9b55db9f7ef352b47](https://qiita.com/TakahikoKawasaki/items/4ee9b55db9f7ef352b47) より引用](/images/need-sub-in-id-token/2024-06-24_18h17_19.png)
[https://qiita.com/TakahikoKawasaki/items/4ee9b55db9f7ef352b47](https://qiita.com/TakahikoKawasaki/items/4ee9b55db9f7ef352b47) より引用
Openid Connect はこの ID トークンを定義することで、ユーザー認証の仕様を定めています。

### ID Token の構造について

OAuth 2.0 を使った認可フローでは JWT を使用したアクセストークンの発行をよく？見かけます。
ですが、OAuth 2.0 はトークンの形式について JWT の使用を必須とする記載は RFC などを見てもありません。
一方、ID Token は以下のように[JWT であることが明記](https://www.notion.so/ID-Token-sub-Claim-6d4a700580384f7390358bb12b99766b?pvs=21)されています。

> ID Token は  [**JSON Web Token (JWT)**](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#JWT) [JWT] である.

そして、以下のような[記載](https://www.notion.so/Github-Copilot-Github-e0b2a5cdd2504f0bb927ec6e337ae846?pvs=21)もあります。

> ID Token は  [**JWS**](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#JWS) [JWS] を使って署名されなければならない (MUST).

このことから、ID Token は JWS によって、署名された JWT だと分かります。
JWS の構造については[RFC-7515](https://datatracker.ietf.org/doc/html/rfc7515#section-3)を確認すると以下のような構造を取ると記載があります。

> o JOSE Header
> o JWS Payload
> o JWS Signature

以上から、ID Token の構造が見えてきました。
ユーザー情報を含めたトークンを返す時に、改ざんとかあったら大丈夫かと思っていましたが、署名によってその辺は守られることが必須となっているのがわかりますね。

### Payload 部分の決まりごと

先程 ID Token の構造については形式が定められていることを確認しました。
次に payload の部分の決まりごとをみていきます。
と言いたいのですが、この部分に関してはそれこそ[川崎さんの Qiita 記事](https://qiita.com/TakahikoKawasaki/items/8f0e422c7edd2d220e06)が詳細に記載いただいているので、ここでは記載しません。
ただし、以下のように sub 属性は必須だと[明記](https://www.notion.so/ID-Token-sub-Claim-6d4a700580384f7390358bb12b99766b?pvs=21)されていることは把握しておいてください。

> sub
> REQUIRED. Subject Identifier. Client に利用される前提で, Issuer のローカルでユニークであり再利用されない End-User の識別子. (例: 24400320 や AItOawmwtWwcT0k51BayewNvutrJUqsvl6qs7A4 等) この値は ASCII で 255 文字を超えてはならない (MUST NOT). sub 値は大文字小文字を区別する.

End-User の識別子は必ず含めることが求められています。

### ID Token の検証

ID Token を取得した場合、その ID Token が適切かを検証する必要があります。
そして、その検証する内容についても仕様書には記載があります。
以下は Authorization Code Flow で ID Token を取得した時の[検証要件](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#IDTokenValidation)になります。

> 1.  ID Token が 暗号化されているならば, Client が Registration にて指定し OP が ID Token の暗号化に利用した鍵とアルゴリズムを用いて復号する. Registration 時に OP と暗号化が取り決められても ID Token が暗号化されていなかったときは, RP はそれを拒絶するべき (SHOULD).
> 2.  (一般的に Discovery を通して取得される) OpenID Provider の Issuer Identifier は iss (issuer) Claim の値と正確に一致しなければならない (MUST).
> 3.  Client は aud (audience) Claim が iss (issuer) Claim で示される Issuer にて登録された, 自身の client_id をオーディエンスとして含むことを確認しなければならない (MUST). aud (audience) Claim は複数要素の配列を含んでも良い (MAY). ID Token が Client を有効なオーディエンスとして記載しない, もしくは Client から信用されていない追加のオーディエンスを含むならば, その ID Token は拒絶されなければならない.
> 4.  ID Token が複数のオーディエンスを含むならば, Client は azp Claim があることを確認すべき (SHOULD).
> 5.  azp (authorized party) Claim があるならば, Client は Claim の値が自身の client_id であることを確認すべき (SHOULD).
> 6.  (このフローの中で) ID Token を Client と Token Endpoint の間の直接通信により受け取ったならば, トークンの署名確認の代わりに TLS Server の確認を issuer の確認のために利用してもよい (MAY). Client は JWS [JWS] に従い, JWT alg Header Parameter を用いて全ての ID Token の署名を確認しなければならない (MUST). Client は Issuer から提供された鍵を利用しなければならない (MUST).
> 7.  alg の値はデフォルトの RS256 もしくは Registration にて Client により id_token_signed_response_alg パラメータとして送られたアルゴリズムであるべき (SHOULD).
> 8.  JWT alg Header Parameter が HS256, HS384 および HS512 のような MAC ベースのアルゴリズムを利用するならば, aud (audience) Claim に含まれる client_id に対応する client_secret の UTF-8 表現バイト列が署名の確認に用いられる. MAC ベースのアルゴリズムについて, aud が複数の値を持つとき, もしくは aud の値と異なる azp の値があるときの振る舞いは規定されない.
> 9.  現在時刻は exp Claim の時刻表現より前でなければならない (MUST).
> 10. iat Claim は現在時刻からはるか昔に発行されたトークンを拒絶するために利用でき, 攻撃を防ぐために nonce が保存される必要がある期間を制限する. 許容できる範囲は Client の仕様である.
> 11. nonce の値が Authentication Request にて送られたならば, nonce Claim が存在し, その値が Authentication Request にて送られたものと一致することを確認するためにチェックされなければならない (MUST). Client は nonce の値を リプレイアタックのためにチェックすべき (SHOULD). リプレイアタックを検知する正確な方法は Client の仕様である.
> 12. acr Claim が 要求されたならば, Client は主張された Claim の値が適切かどうかをチェックすべきである (SHOULD). acr Claim の値と意味はこの仕様の対象外である.
> 13. auth_time Claim が要求されたならば, この Claim のための特定のリクエストもしくは max_age パラメータを用いて Client は auth_time Claim の値をチェックし, もし最新のユーザー認証からあまりに長い時間が経過したと判定されたときは再認証を要求すべきである (SHOULD).

検証要件をそのままつらつらと引用していますが、確認して欲しいのは sub 属性の検証についての記載がどこにもないことです。
ID Token には sub Claim を含めることが必須となっているのに、検証の段階では一切ふれておりません。
これは中々不思議ですね。
まあ、仕様ってのは必要最小限に書かれていることが多いので、実際に使用する場合はもう少し検証しているかもしれません。
そこで、IDaaS の一つである Auth0 が提供している[auth0-spa-js](https://github.com/auth0/auth0-spa-js)というライブラリを見てみましょう。
このライブラリは SPA のフロントを Openid Connect のクライアントとした場合、クライアントについての振る舞いを実装しているものとなります。
このライブラリで ID Token を検証している、[jwt.ts](https://github.com/auth0/auth0-spa-js/blob/main/src/jwt.ts)にある sub を検証している部分を見てみます。

```tsx
if (!decoded.user.sub) {
  throw new Error(
    "Subject (sub) claim must be a string present in the ID token"
  );
}
```

分岐としては存在していますが、あくまで存在チェックだけとなっています。
sub がユーザーの識別子かどうかなどは見ておらず、好きな値を入れても検証エラーにはなりません。
このように、sub 属性は ID Token において必須としている割には、特に着目されていません。
この sub Claim はいつ使われているのでしょうか？
その答えが次に紹介する UserInfo Endpoint になります。

## UserInfo Endpoint とは

OpenID Connect Core 1.0 を確認すると以下のように定義されています。

> UserInfo Endpoint は, 認証された End-User に関する Claim を返す OAuth 2.0 Protected Resource である. 要求された End-User の Claim を取得するため, Client は OpenID Connect Authentication を通して得られた Access Token を用いて UserInfo Endpoint に要求する.

まず UserInfo Endpoint は End-User の情報を得るためのリソースです。
この UserInfo Endopoint で ID Token には含まれていないユーザーの追加情報をとることができます。
そして、UserInfo Endopoint にユーザー情報をリクエストする際に使用するのは ID Token ではなく、アクセストークンと明記してあります。
このことから、リソースサーバーからユーザー情報を取得するためのエンドポイントだとわかります。※
※この部分については仕様を読んで理解したというわけではありません。以下のツイートを参考にして書かせていただきました。自分で見つけたように書いており、恐縮ですが参考元はあるということを明記させていただきます。
https://twitter.com/ritou/status/1804186193372025233
なお、この UserInfo Endpoint からしか欲しいユーザー情報を取得できないわけではありません。
以下のような[記載](https://www.notion.so/Open-ID-Connect-SSO-a399fb2c85414a1184de6e7c7506280d?pvs=21)があるように、ID Token に含めて取得しても問題はなさそうです。

> これらは  [**Section 5.3.2**](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#UserInfoResponse)  に示す UserInfo Response, あるいは  [**Section 2**](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#IDToken)  に示す ID Token のどちらかに含めて返却するように要求できる.

UserInfo Endpoint と ID Token は取得するタイミングが違ったり、発行するサーバーが異なる場合があるので、ユースケースに合わせて使い分ける必要がありそうです。

## sub Claim の必要性をみていく

ここまで ID Token や UserInfo Endopoint の存在を確認していきました。
では、ようやく本題の sub Claim についてみていきます。
ID Token に sub Claim が必須な理由は Openid Connect Core 1.0 の[Sucessful UserInfo Response](https://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#UserInfoResponse)を見るとわかります。

> UserInfo Response には, 必ず sub (subject) Claim を含むこと (MUST).
>
> 注) トークン置換攻撃 (Section 16.11 参照) を考慮すると, UserInfo Response が必ずしも ID Token の sub Claim が示す End-User のものであるとは保証できない. そのため UserInfo Response に含まれる sub Claim が ID Token のそれと一致することを検証しなければならない (MUST). もし 2 つが一致しない場合, UserInfo Response を利用してはならない (MUST NOT).

ユーザー情報を取得するためのエンドポイントである UserInfo Endpoint は必ずユーザーの識別子を返す必要があります。
そして、そのユーザー識別子は End-User のものだと保証する必要があります。
その時に、使用するのが ID Token の sub Claim です。
なぜ、ID Token の sub Claim が使用できるかを思い出してみると、ID Token は認証した End-User の情報を持っているトークンでした。
であれば、OpenID Connect において ID Token のみが認証した End-User であることを担保していると言えそうです。
一方、UserInfo Response はアクセストークン経由でユーザー情報を取得します。
アクセストークンは認証した End-User の情報を持っていることは担保していないので、UserInfo Response も必ず認証した End-User の情報をとってきていることを保証はできないです。
そこで、取得したユーザー情報と ID Token の sub Claim を比較することで、認証した End-User であることを担保するのです。
以上のことから、ID Token に sub 属性が必要な理由が分かります。
UserInfo Endpoint という在を考ると、確かに ID Token に sub Claim を必須としておく必要があるなと理解できます。
私はこの UserInfo Endpoint の把握が漏れていたが故に ID Token の sub Claim の必要性がわからなかったので、今回知ることができてかなりスッキリしました。

## 少しだけ疑問：トークン置換攻撃は Authorization Code Flow でも起きるのか？

ID Token に sub Claim が必要な理由を確認してきました。
書いてあることは確かになるほどと思いました。
ただ、少し疑問も湧いています。
それは、Authorization Code Flow でトークンを取得した際、トークン置換攻撃にあうのかということです。
現状 state パラメータや[PKCE](https://tex2e.github.io/rfc-translater/html/rfc7636.html)を設定することで、リソースオーナーにちゃんとアクセストークンを渡すことを保証しています。
となると、トークン置換攻撃は Authorization Code Flow では対策されているのかなと思っています。
となると、Authorization Code Flow の時は ID Token の検証をしなくても、UserInfo Endpoint で取得したユーザー情報は End-User のものだと担保できそうだと感じています。
まあ、OpenID Connect は Authorization Code Flow だけではないですし、検証をすることによる弊害もないので、検証は不要だとは思いませんが。
ただ、Authorization Code Flow を適切に実装している場合、トークン置換攻撃がおきるのかが少し疑問に感じたので、余談として書かせていただきました。

## おわりに

今回は ID Token の sub Claim についてみていきました。
当初は sub Claim は必要か？という疑問を解消するために調べていましたが、その過程で UserInfo Endpoint について理解を深められたのは良かったです。
まだまだ理解できていない部分はたくさんあるので、引き続き調べていこうと思います。
ここまで読んでいただきありがとうございました。
