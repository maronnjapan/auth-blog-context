---
title: "OAuthにおけるPublic ClientとConfidential Clientについて"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OAuth"]
published: true
---

## はじめに

まずは以下の Authorization Code Flow における、トークンを取得するためのリクエストをご覧ください。

```
POST https://{yourDomain}/oauth/token
Content-Type: application/x-www-form-urlencoded
grant_type=authorization_code
&client_id={yourClientId}
&client_secret=YOUR_CLIENT_SECRET
&code=AUTHORIZATION_CODE
&redirect_uri={https://yourApp/callback}
```

次にもう一つのトークンを取得するためのリクエストをご覧ください。

```
POST https://{yourDomain}/oauth/token
Content-Type: application/x-www-form-urlencoded
grant_type=authorization_code
&client_id={yourClientId}
&code_verifier=CODE_VERIFIER
&code=AUTHORIZATION_CODE
&redirect_uri={https://yourApp/callback}
```

違いは client_secret クエリがあるか、code_verifier クエリがあるかの違いです。
この違いを紐解くには、Public Client と Confidential Client を把握するとスッキリします。
ここまでの話で、クライアントがシークレットを保持できるか否かを言いたいのねと思ったかたはあまり新しい発見はないかもしれません。
しかし、どういうこと？と思った方は是非とも続きを読んで見てほしいです。

## この記事のここだけ!!!

Public Client と Confidential Client の違いはシークレットキー(認可サーバーへアクセスするためのパスポート)を保持できるクライアントかどうかです。
シークレットキーを保持できるなら、Confidential Client として実装を行い、保持できないなら Public Client として実装を行います。

## Public Client とは

Public Client は[RFC](https://openid-foundation-japan.github.io/rfc6749.ja.html#client-types)を見ると、以下のように定義されています。

> パブリック (public)
>
> (例えば, インストールされたネイティブアプリケーションやブラウザベースの Web アプリケーションなど, リソースオーナーのデバイス上で実行されるクライアントのように) クライアントクレデンシャルの機密性を維持することができず, かつ他の手段を使用したセキュアなクライアント認証もできないクライアント.

わかるようなわからないような感じですが、着目するのは以下の部分です。

```
クライアントクレデンシャルの機密性を維持することができず
```

[クレデンシャル](https://www.sompocybersecurity.com/column/glossary/credential)は認証情報と置き換えてください。
そして、クライアントは認可サーバーへリクエストを投げるアプリケーションと置き換えます。
すると先程着目した部分は以下のように変更できます。

```
認可サーバーにリクエストを投げるアプリの認証情報を隠しておくことができず
```

もっと言い換えてみます。

```
ソースコードが閲覧できてしまうため、認証情報が取得可能なアプリ
```

ここまでくると見えてきます。
すなわち、フロント側で認可サーバーとやり取りせざるを得ないアプリケーションのことを Public Client といえそうです。※
※この説明は正確ではないとは思っています。上記の状態は確かに Public Client に属すると思いますが、フロントエンド=Public Client とするのは乱暴です。
ただ、他の良い例が思い浮かばず、かつ Web 畑の私がイメージしやすいのがフロントエンドでの説明だったので、このように記載しています。

## 余談 OAuth なのに認証とは如何に？

Public Client の説明の時に、クライアントクレデンシャル、すなわちクライアントの認証情報の機密性について言及していました。
これに違和感を持つ人もいると思います。というか私がそうでした。
OAuth は認可(委譲?)プロトコルなので、認証については範囲外だと感じます。
それなのに、突然クライアントと認可サーバーで認証情報のやり取りが出ているのはよくわかりません。
ただ、紐解いていくと認可について扱うのはリソースオーナーの部分で、クライアントと認可サーバー間はまた別の話だとわかります。
まずは OAuth における登場人物を再度確認します。
以下の 4 つの登場人物がいます
①Resource Owner：権限を付与される対象。通常操作する人を指します。
②Resource server：アクセストークンを用いたリクエストを受理してレスポンスを返すサーバ
③Client：Resource Owner の代わりに Resource server にアクセスするソフトウェア
④Authorization Server：アクセストークンをクライアントに発行するサーバー
この登場人物の内、OAuth が委譲する対象と見ているのは ① の Resource Owner に対してです。
Resource Owner に対しては、誰であろうと適切な手順さえ踏んでいれば移譲を行うと OAuth はしています。
なので、Resource Owner が誰であるかを決定する認証を OAuth で行おうとすると、OAuth の目的からは外れます。
Resource Owner に対しては認証の強制を行わないと言いましたが、Client と Authorization Server は別の話です。
Client が Authorization Server にリダイレクトではなくリクエストを投げる際は、Client を一意に特定可能とする必要があります。
ここを一意に特定できないと、仮に Authorization Code Flow において認可コードを取得した際に、認可コード横取りによって、アクセストークンを盗みとることが簡単にできてしまうためです。
なので、Client から Authorization Server へのリクエストに対しては認証を行うことが求められます。

## Confidential Client とは

Confidential Client は[RFC](https://openid-foundation-japan.github.io/rfc6749.ja.html#client-types)を確認すると以下のように記載されています。

> コンフィデンシャル (confidential)
>
> クレデンシャルの機密性を維持することができるクライアント (例えば, クライアントクレデンシャルへのアクセスが制限されたセキュアサーバー上に実装されたクライアント), または他の手段を使用したセキュアなクライアント認証ができるクライアント.

Public Client の逆ですね。
Public Client がソースコードを閲覧でき、認証情報を探し出せてしまうのに対して、Confidential Client はバックエンド側のアプリなどで第三者が認証情報を確認するのが困難なアプリとなっています。
Client なのにバックエンドとはどういうことだと思われるかもしれませんが、OAuth においては Resource server にアクセスするアプリを Client とします。
例えば、以下のような構成があるとします。
![oauth.png](/images/oauth-public-client-and-confidential-client/oauth.png)
BFF は Backend For Frontend の略称なので、バックエンド側のアプリケーションとなります。
しかし、認可サーバーとのやり取り自体は BFF が行っているため、OAuth の範囲では BFF が Client という扱いになります。
Client = Frontend ではないので、認証情報を機密保持できるクライアントが存在可能であり、Confidential Client という概念が存在します。
Confidential Client は認証情報を持っているので、Authorization Server とのやり取りは認証情報を渡しつつ行うことができます。

## Public Client は辞められないの？

Confidential Client と Public Client の違いを見てきましたが、Confidential Client の方が安全に通信できるように感じます。
なら、Public Client は使わない方が良いのではと思いますが、世の中には Public Client にしか分類できないアプリが存在します。
なので、ここではそういったアプリについて確認していきます。

### Public Client になるアプリ ①：外部 API と通信する SPA

まずは Web 畑の自分にとって一番しっくりくるのが、外部 API を使用している SPA です。
外部 API はその名の通り自身が管理していない API です。
そのため、認可サーバーと接続するための認証情報を保存することはできませんし、認可サーバーとのやり取りも設定することが困難です。
となると、認可サーバーとのやり取りはフロント側すなわち SPA 側で行わざるを得ません。

### Public Client になるアプリ ②：ネイティブアプリケーション&モバイルアプリケーション

ネイティブアプリケーションやモバイルアプリケーションは自身の端末にアプリケーションをインストールします。
通常の Web アプリであれば、Authorization Code (認可コード)といった情報 はブラウザを介して直接 Client に対して到達しますが、ネイティブアプリの場合、ブラウザから離れ、OS を介してアプリに到達します。
ブラウザからアプリに遷移するときの方法としては、Custom URL Scheme と呼ばれる方法が多く用いられています。
しかし、この Custom URL Scheme は対象のものと同じ Custom URL Scheme のアプリをインストールさせることで、本来リダイレクトさせたいアプリとは別のアプリにリダイレクトさせることができます。
となると、ここでクライアントシークレットの値を使用していると傍受される可能性があります。
こういった懸念点が多くあるので、ネイティブアプリケーションやモバイルアプリケーションも Public Client として分類する必要があります。
このように Public Client は様々な要件を考えた時に、どうしても存在します。
特にスマホが普及している現在において、スマホアプリを使用しないという選択肢はありません。
そういったアプリに OAuth を使う場合は、Public Client にする必要があります。
よって、Public Client を無くすというて選択肢は有り得ないのが回答となります。

## おわりに

今回は Public Client と Confidential Client の違いについて軽く見ていきました。
内容としてはとても簡単なものです。
認可サーバーとのやり取りが第三者でも行えるか、開発者が管理している場所からしかやり取りができないかの違いでしかないです。
ただ、今回の内容は私にとって一つの壁を超えることができた気がしています。
私はフロントエンドを Public Client とした SPA から OAuth を学び始めました。
そのため、Authorization Code Flow でクライアントシークレットを使う意味がわかりませんでした。
クライアントシークレットを使う=Client Confidentials だと思っていました。
しかし、今回改めてクライアントの種類について学んだことで、Authorization Code Flow でもクライアントシークレットを使う場合についてや、なぜそもそも種類分けが行われているのかが理解できました。
それに伴い動的クライアントの意義、なぜ OAuth2.0 では PKCE の実装が必須でないのかも理解できました。
分かった瞬間は知識が一気に繋がりました。
この感動を文字として残したく、今回記事を投稿しました。
ここまで読んでいただきありがとうございました。

### 参考資料

@[card](https://openid-foundation-japan.github.io/rfc6749.ja.html)
@[card](https://qiita.com/TakahikoKawasaki/items/63ed4a9d8d6e5109e401)
https://www.shoeisha.co.jp/book/detail/9784798159294
