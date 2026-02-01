---
title: "OpneId Connectのオプショナルなパラメータ達① ~promptパラメータ~"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OAuth", "OpenIdConnect", "Auth0"]
published: true
---

## はじめに

ユーザー認証を行うための手段の一つとして、OpenId Connect(以降 OIDC)があります。
OIDC は OAuth の機能を拡張して、認証でも使用できるようにしたものとなります。
この OIDC によって、一つのサーバーで複数アプリ内のユーザーを管理することができ、シングルサインオンなども簡単に構築することができます。
その他にも OIDC は多くの機能を有しており、認証を考える際に有効な選択肢となります。
今回は OIDC が持つ多くの機能の一つである、prompt パラメータについて説明していきます。
OIDC のコア機能ではないかもしれませんが、挙動して興味深いものだったので、是非とも読んでいただけると幸いです。
では始めます。

# 前提

これからお話する prompt パラメータについてですが、OpenId Connect の中の Authrization Code Flow を用いた認証の時に使用されます。
Hybrid Flow や Implicit Flow での使用を想定して記載していないことはご認識お願いします。

# Authrization Code Flow について

prompt パラメータについて理解してには、以下画像の Authrization Code Flow について理解しておく必要があります。
そこで、まず Authrization Code Flow について説明をした後に prompt パラメータについて見ていきます。
なお、画像は Auth0 から引用しているので Auth0 Tenant となっていますが、説明の際は認可サーバーとさせていただきます。
さらに、OpenId Connect の話をしようとしているのに認可サーバーを使うのは適切でないとは思いますが、Identify Provider を使用して Authorization Code Flow の説明がしにくので、認可サーバーを使っています。
ご容赦ください。
![[Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow)から引用](/images/c9f76dd96e5d65/2023-09-25_08h34_12.png)
[Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow)から引用

## Authorization Code Flow の目的

各手順を説明する前に Authrizaton Code Flow の目的を記載します。
OpenId Connect において、Authrizaton Code Flow は以下の二つを取得するために存在しています。

1. アクセストークン
2. ID トークン

今回は各トークンの詳細は説明しませんが、Authrizaton Code Flow はこのトークンを取得するためのフローであることはご認識お願いします。

## ①Click login link

これは操作したいアプリケーション画面にあるログインボタンをクリックすることです。
例として下記画像をみてください。
![2023-09-25_08h41_07.png](/images/c9f76dd96e5d65/2023-09-25_08h41_07.png)
これは Chat GPT のトップ画面です。
赤枠にある「Log In」ボタンをクリックすることが、Click login link の操作となります。

## ②Authrization Code Request to /authorize

① でログインボタンをクリックした時、ログインしているかのチェックを行う URL に遷移する動作です。
この遷移するときに様々なパラメータを付与しながら遷移します。
今回解説する prompt パラメータも上記パスへの URL 遷移の際に付与されています。

## ③Redirect to login/authorization prompt

ここでは ② で遷移した時に、認証の有無などの設定でログイン画面や同意画面を出すのかを決定します。
Chat GPT で言うと、ログインボタンを押すと以下のようなログイン画面が表示されます。
これは/authrize に遷移したときに認証されていないので、認可サーバーがログインを求める画面へリダイレクトさせたためです。
![2023-09-25_11h55_53.png](/images/c9f76dd96e5d65/2023-09-25_11h55_53.png)
ログインではなく同意画面を求める場合もあります。
以下は Yahoo の例です。
![Untitled](/images/c9f76dd96e5d65/Untitled.png)
[【更新あり】Yahoo! ID 連携での同意画面のデザインが新しくなります](https://developer.yahoo.co.jp/changelog/2015-06-01-yconnect.html)から引用
同意画面を表示させる場合は外部連携の時に用いることが多いです。
これは、連携先にユーザー情報を渡す必要があるので、渡しても問題ないかを確認するためです。
なお、認証と同意は両方要求しても良いです。
実際に、ユーザーを入力してログインした後にユーザー情報を提供してよいかの同意画面が出ることはソーシャルログインの場合多々あります。
また、ログイン画面や同意画面も必ず表示させる必要はありません。
認証済みの場合はログイン画面を表示させず、③ と ④ の手順を省略しても問題ありません。
ただ、/authorize へ遷移後以下の 3 つの経路を辿ることはこの後解説する prompt パラメータと大いに関わってきますので、把握していただきたいです。

## ④Authenticate and Consent

ユーザー側が「③Redirect to login/authorization prompt」によって表示された画面に沿って、連携の許可を行ったり認証を行ったりします。
「③Redirect to login/authorization prompt」でも記載しましたが、一般的には認可サーバー側でセッションなどに情報が残っていれば ③ から ④ の動作は省略されます。
実際にログインしている状態で Chat GPT のトップにアクセスすると、認証画面に行かずそのままプロンプト入力画面にアクセスできると思います。

## ⑤Authorization Code

これは認証や認可が完了した後、Chat GPT などのユーザーが操作するアプリケーションにリダイレクトさせる処理です。
リダイレクトの際に、トークンを取得するための引換券も付与します。
引換券でなく、認証・認可が完了したのだからそのままトークンを渡して欲しいですが、セキュリティの関係上できません。
この記事では詳細に触れませんが、この段階を挟むことは Authrizaton Code Flow で安全性が保たれる根幹の一つなので、省略はできません。

## ⑥Authorization Code + Client ID + Client Secret to /oauth/token

先程受け取った引換券を Chat GPT などのアプリケーションが使用してトークンを取得するためのリクエストを出します。
Authorizaton Code 以外も色々と記載していますが、トークンの引換券がちゃんと認可サーバーが発行したものかを証明するために設定しています。

## ⑦Validate Authorization Code + Client ID + Client Secret

これは Chat GPT などのアプリケーションから認可サーバーにきたトークンが欲しいという要望に対して、ちゃんと正規の手続きを踏んでいるかを確認しています。
このチェックが通れば、ようやく Authorization Code Flow の目的であるトークンの取得ができます。

## ⑧ID Token And Access Token

⑦ でチェックが通ったら、トークンが欲しいと要望していたアプリケーションに ID トークンとアクセストークンを渡します。
なお、Authrization Code Flow の画像には記載ありませんが、ID トークンを受け取った後アプリケーション側で ID トークンがちゃんと正規のものかをチェックします。
これによって、アプリケーション側でちゃんと認証ができたことを確認できるようになります。

## ⑨Request user data with Access Token

アクセストークンを使って、API からユーザー情報を取得しています。

## ⑩Response

アクセストークンに問題がなければ ⑨ のリクエストに対して、情報を返します。
なお、展開したパスは Authorization Code Flow の一般的なパスとなっていますが、必ずこのパスで Authorization Code Flow を実現する必要はありません。
パスは異なっていても、流れが同じようであれば Authorization Code Flow を実装できていると言えます。
ただ、他のシステムとの互換性を考えたときに、特別な事情がなければ一般的なパスに沿うことが推奨されます。
以上で Authorization Code Flow の簡単な概要を説明しました。
それでは本題の prompt パラメータの説明を行っていきます。

# prompt パラメータの各値について

ここからは prompt パラメータの説明を行っていきますが、その前に prompt パラメータは何をするための値かを説明します。
prompt パラメータは Authorization Code Flow における「③Redirect to login/authorization prompt」と「④Authenticate and Consent」の挙動をどうするかを定めるためのパラメータとなります。
すなわち、認証や外部連携の同意画面の扱いをどうするのかを決めるのが prompt パラメータとなります。

## prompt=none

この値の定義は[ドキュメント](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#AuthRequest)では以下のようになっています。

> Authorization Server はいかなる認証および同意 UI をも表示してはならない (MUST NOT). End-User が認証済でない場合, Client が要求する Claim 取得に十分な事前同意を取得済でない場合, またはリクエストを処理するために必要な何らかの条件を満たさない場合には, エラーが返される. 典型的なエラーコードは  login_required, interaction_required  であり, その他のコードは  **[Section 3.1.2.6](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#AuthError)**  で定義されている. これは既存の認証と同意の両方, またはいずれかを確認する方法として使用できる.

ざっくり説明すると、認証や外部連携の同意をしてようがしていますが、Authorization Code Flow の「③Redirect to login/authorization prompt」と「④Authenticate and Consent」を省略するためのパラメータとなります。
なので、このパラメータの値を付与して/authorize に遷移すると必ず設定したアプリケーションにリダイレクトします。
とはいえ、認証などができていない場合はトークンの引換券を渡すことはできないので、引換券の代わりにエラークエリを渡すようにはなっています。

## prompt=login

ドキュメントは以下の通りです。

> Authorization Server は End-User を再認証するべきである (SHOULD). 再認証が不可能な場合はエラーを返す (MUST). 典型的なエラーコードは  login_required  である.

こちらは先程の none とは違って、ユーザーが認証していようがいまいが、必ず/authorize へのリクエストが来た時はログイン画面へ移動し、認証しないと次にいけないようになっています。
ちなみに、[Chat GPT の/auth/login](https://chat.openai.com/auth/login)の画面に存在するログインボタンをクリックすると、このパラメータが付与されます。
そのため、仮にログインしていたとしても必ずログイン画面に遷移するようになっています。

## prompt=consent

ドキュメントは以下の通りです。

> Authorization Server は Client にレスポンスを返す前に End-User に同意を要求するべきである (SHOULD). 同意要求が不可能な場合はエラーを返す (MUST). 典型的なエラーコードは  consent_required  である.

こちらは prompt=login と似ており、ユーザーがどのような状態であっても必ず同意画面を表示させ、同意しないと次にいけなようになっています。

## prompt=select_account

ドキュメントは以下の通りです。

> Authorization Server は End-User にアカウント選択を促すべきである (SHOULD). この prompt 値は, End-User が Authorization Server 上に複数アカウントを持っているとき, 現在のセッションに紐づくアカウントの中から一つを選択することを可能にする. End-User によるアカウント選択が不可能な場合はエラーを返す (MUST). 典型的なエラーコードは  account_selection_required  である.

これは認証する際にどのアカウントで認証するかを選択したいときに付与します。
これは少し具体的に見ていきます。

### prompt=select_account の挙動を確認する

下準備のに Auth0 で Google 認証ができるようにします。
まず Google 側の準備ですが、[こちらの記事](https://dev.classmethod.jp/articles/auth0-google/)が詳しいので、その通りに行います。
上記リンクで、Google 認証用の ID とシークレットが取得できたので、次に Auth0 の設定を行います。
Auth0 の Authenticate→Social へ遷移し、右上にある「Create Connection」をクリックします。
ソーシャル認証できるアプリの一覧が表示されるので、そこから Google の項目を選択します。
すると設定画面が表示されるので、そこのとに先ほど作成した Google のクライアント ID とシークレットキーを入力します。
![2023-09-28_21h27_50.png](/images/c9f76dd96e5d65/2023-09-28_21h27_50.png)
下にいくと、Permissions という項目があります。
これは Google と連携する際にどのような Google 側の情報を Auth0 が取得するのかを選択する項目です。
この項目を設定すると、画面のように Auth0 へ提供する情報が Google の認証の際に表示されます。
![2023-09-28_21h31_19.png](/images/c9f76dd96e5d65/2023-09-28_21h31_19.png)
設定が完了したら、Applications のタブを選択して Google 認証を行えるようにしたい Auth0 のアプリケーションと連携させます。
これで設定は完了です。
ログイン画面に Google でログインできるボタンが存在するかを確認し、実際にログインできることを確かめます。
![2023-09-28_21h30_13.png](/images/c9f76dd96e5d65/2023-09-28_21h30_13.png)
これで準備が完了したので、本題の prompt=select_account の説明をします。
なお、今回のサンプルアプリは React を使用しています。
Vue などとほぼ同じですが、若干違う可能性があるのでその点はご留意ください。
なお、React での Auth0 の構築は[クイックスタート](https://auth0.com/docs/quickstart/spa/react/01-login)を参照してください。
まず何も prompt を指定せずに Google で認証します。
すると、馴染みの Google のログイン画面が出てきます。
そこで、認証を行った後いったんログアウトして再度 Google 認証を行うと Google の画面は表示されずそのまま認証が完了します。
次に prompt=select_account を付与して、Google 認証を行うようにします。
すると、今度はどのアカウントでログインするかを選択する画面が必ず表示されるようになります。
これが prompt=select_account の効果です。
なお、注意点としてブラウザに複数の Google アカウント情報が存在していると prompt=select_account を設定していようがいまいが、ログインするアカウントを選択する画面を経由します。
これは、ブラウザ側がどのユーザー情報を使用すればいいのかがわからないことが原因と考えられるます。
以上が prompt パラメータの概要となります。
アプリケーションによって、ログイン画面などの出し分けができるのはいいですね。

## 余談 Google 認証を Terraform を使って、Auth0 で設定する

prompt=select_account の動作を確認するために先程 Auth0 で Google 認証の設定を行いましたが、これを Terraform で行うと以下の通りです。
なお、初期設定や変数定義の部分は省いています。

```terraform
resource "auth0_client" "my_client" {
  name                = "Application - Acceptance Test"
  description         = "Test Applications Long Description"
  app_type            = "spa"
  callbacks           = ["http://localhost:3000"]
  allowed_origins     = ["http://localhost:3000"]
  allowed_logout_urls = ["http://localhost:3000"]
  web_origins         = ["http://localhost:3000"]
  grant_types = [
    "authorization_code",
  ]
}
resource "auth0_connection" "google_oauth2" {
  name     = "Google-OAuth2-Connection"
  strategy = "google-oauth2"
  options {
    client_id     = var.google_oauth_client_id
    client_secret = var.google_oauth_client_secret
  }
}
resource "auth0_connection_client" "google_oauth2_connection" {
  client_id     = auth0_client.my_client.client_id
  connection_id = auth0_connection.google_oauth2.id
}
```

まず最初に以下のコードで、Google 認証と紐づける Application を作成しています。

```terraform
resource "auth0_client" "my_client" {
  name                = "Application - Acceptance Test"
  description         = "Test Applications Long Description"
  app_type            = "spa"
  callbacks           = ["http://localhost:3000"]
  allowed_origins     = ["http://localhost:3000"]
  allowed_logout_urls = ["http://localhost:3000"]
  web_origins         = ["http://localhost:3000"]
  grant_types = [
    "authorization_code",
  ]
}
```

次に以下コードで Google 認証との連携を行っています。
client_id と client_secret は先程 Google Cloud で取得した値となります。

```terraform
resource "auth0_connection" "google_oauth2" {
  name     = "Google-OAuth2-Connection"
  strategy = "google-oauth2"
  options {
    client_id     = var.google_oauth_client_id
    client_secret = var.google_oauth_client_secret
  }
}
```

そして、最後に作成した Application と Google 認証の設定を紐づけます。

```terraform
resource "auth0_connection_client" "google_oauth2_connection" {
  client_id     = auth0_client.my_client.client_id
  connection_id = auth0_connection.google_oauth2.id
}
```

以上で完了となります。
案外少ないコードで表現できるので、簡単に理解できました。良かったです。

# prompt パラメータを活用する

ここまで prompt パラメータの説明をしましたが、これをどう活用するかを最後に軽く説明します。
今回活用するのは prompt=none です。
おさらいすると、prompt=none は認証・認可してようがいまいがアプリにリダイレクトする処理を強制する値でした。
このアプリへ強制的にリダイレクトすることで、認証の有無で画面の出し分けをおこなうことができます。
具体的には以下の通りです。

1. 認証するかを判定するアプリ側の URL にアクセスする。
2. 認証・認可サーバーへ prompt=none を付与しつつ、認証しているかを問い合わせる。
3. 認証していれば、認証済みの人が見ることができる画面に遷移する。
4. 認証していなければ、認証していなくても画面に遷移する。

これを図に示すと以下の通りです。
![AWS.drawio (5).png](</images/c9f76dd96e5d65/AWS.drawio_(5).png>)
これによって、認証していればそのまま認証済みの画面を閲覧でき、認証できなくても何かしらページを表示させたいというユーザーの要望に応えることができます。

# おわりに

今回は OpenId Connect における prompt パラメータについて説明しました。
正直 Authorization Code Flow の根幹を支えるものではないので、今までスルーしていました。
しかし、今回調べてみると中々使えるパラメータだと分かりました。
となると他のパラメータもちゃんと調べないといけないので、また分かり次第展開します。
ただ、id_token_hint の活用方法がいまいち分かっていないので、展開できるのは大分先になりそうですが…。
ここまで読んでいただきありがとうございました。
