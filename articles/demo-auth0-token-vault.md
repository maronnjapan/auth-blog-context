---
title: "Auth0のToken Vaultを体験できるデモアプリの流れ解説"
emoji: "📑"
type: "tech"
topics: ["Auth0", "Token Vault"] 
published: true
targetCategories: ["security"]
---

## はじめに
以前の記事で、Auth0の[Token Vault](https://auth0.com/docs/secure/call-apis-on-users-behalf/token-vault)について言及しました。
https://web.maronn-room.com/maronn2/articles/token-vault-by-auth0

そこでは、Token Vaultを試せるアプリのサンプルコードを記載しました。
もちろん可能な限りセットアップしやすいことは意識しましたが、実際に動かすのは環境違いなどで面倒かと思います。
そこで、この記事ではデモアプリが何をしているのかを解説していこうと思います。
Auth0のToken Vaultは気になるけど、アプリを動かすのはちょっと手間だと思う方は是非参考にしてください。

## Token Vaultの前提を確認
アプリを解説する前にToken Vaultの技術概要を軽く見ていきます。
Token Vaultはざっくり言ってしまうと、Auth0のトークンを元に外部プロバイダーのトークンをシームレスに取得することを可能にするストアのようなものです。
これによって、一回連携をすればAuth0がいい感じにトークンの交換をしてくれます。
そのため、ユーザーが何度も外部プロバイダーにアクセスして、実行許可を与える必要がなくなるという利点があります。
このToken Vaultですが、大きく分けて以下二つのフローがあります。
- Auth0と外部プロバイダーの連携
- Auth0のトークンエンドポイントへリクエスト

それぞれ見ていきます。
まずは、外部プロバイダーとの連携です。
トークンを交換するためには、Auth0と外部プロバイダーが連携できるということを伝える必要があります。
そのフロー全体が以下の通りです。
![](/images/demo-auth0-token-vault/connect-flow.png)
Auth0のToken Vaultの設定を行うには、[My Account API](https://auth0.com/docs/manage-users/my-account-api)というAuth0提供のAPIの実行が必要です。
なので、まずアプリ（クライアント）はMy Account APIに対してToken Vaultの設定をはじめますと伝えます。
Auth0側が連携の開始をOKとしたら、受け付けたことを示す値やリダイレクト先URLなどをクライアントに渡します。
クライアントはそれらをもとにクエリやURLを構築し、Auth0にリダイレクトします。
Auth0にリダイレクト後、もともと連携する予定の外部プロバイダーとの連携画面に遷移します。
連携が完了したら、Auth0->クライアントという形でリダイレクトされます。
クライアントはリダイレクトした時に、Token Vaultを有効にするための値を受け取るので、それをAuth0に渡します。
Auth0はその値を受け取り、適切であれば対象のプロバイダーにおけるToken Vaultの利用を可能にします。
以上が外部プロバイダーとの連携です。
次に、Auth0のトークンエンドポイントへリクエストです。
[Token Exchange](https://www.rfc-editor.org/rfc/rfc8693.html)のお作法に従い、Token Vaultを用いたトークン交換に必要な値をトークンエンドポイントにリクエストするだけです。
値については色々と要件はありますが、流れは上記を抑えておけば十分です。
以上がざっくりとしたToken Vaultの技術概要です。
以降はデモアプリのキャプチャを掲載することで、今言及した流れがどういった値で行われているかがわかるかと思います。

## デモアプリの流れ
ここからは実際にアプリの動きを記載します。
アプリを動かすのに手間がかかる方や、設定はしたけどどう動きを確認すればよいか悩んでいる方は参考にしてください。
なお、今回はToken Vaultを用いたトークン交換は基本的にリフレッシュトークンを使用します。
アクセストークンを使用する場合の流れなどは以下のドキュメントを参照してください。
https://auth0.com/docs/secure/call-apis-on-users-behalf/token-vault/access-token-exchange-with-token-vault
### Auth0へのログインからトークンを取得

まずアプリを動かし、TOP画面にアクセスします。
そこではログインボタンが存在するので、そのボタンをクリックします。
Auth0へのログイン画面に遷移するので、そこでログインを行うと以下の画面が表示されます。
![](/images/demo-auth0-token-vault/after-login-not-connected.png)
ここまで、遷移したらToken Vaultを利用するために必要なAuth0が発行したトークンを取得できます。
### Token Vaultの有効化
Auth0発行のトークンを取得できたら、先程の画面の「連携する」ボタンをクリックします。
クリックをすると内部ではまずアプリがAuth0のMy Account APIに連携することを伝えます。
Auth0はリクエストが適切であれば、連携用のリダイレクト用URLとリクエストを識別するためのチケットを返します。
上記のリダイレクトURLやチケットをもとにAuth0へリダイレクトし、Auth0は更に外部プロバイダーへ連携のためリダイレクトします。（今回はGoogle）
ユーザーは連携したい外部プロバイダー（Google）のアカウントを選択し、権限を与えたらAuth0にリダイレクトします。
Auth0はアプリが事前に渡していたリダイレクトURLにリダイレクトします。
アプリはリダイレクトされた際に付与されたコードなどをAuth0に渡すことで、Token Vaultの利用設定が完了します。
完了すると、以下の画面が表示されます。
![](/images/demo-auth0-token-vault/complete-connected.png)
また、アカウント連携実行ログには今説明した内容について、フローのどこに位置するかとリクエスト・レスポンス内容が以下のように表示されます
![](/images/demo-auth0-token-vault/initial-connect-by-my-account-api.png)
![](/images/demo-auth0-token-vault/initial-connect-info.png)
![](/images/demo-auth0-token-vault/integrate-auth0-and-provider.png)
![](/images/demo-auth0-token-vault/finish-connected.png)
![](/images/demo-auth0-token-vault/finish-connected-info.png)
これでToken Vaultの有効化と有効化の際に行ったことが確認できます。
### Token Vaultによるトークン交換
ここまでトークンを交換する準備はできました。
ですが、まだ実際に交換はしていないので、その操作をするための画面について説明します。
Token Vaultの有効化を行う画面を下にスクロールすると、以下のようなトークン交換の画面があります。
![](/images/demo-auth0-token-vault/exec-token-exchange-view.png)
トークンを交換するボタンをクリックすると、Token Vaultの有効化の時と同様にリクエスト・レスポンス内容が以下のように表示されます。
![](/images/demo-auth0-token-vault/exec-token-exchange-info.png)
フロー図については、トークン交換の時はアクセストークンなどを発行するエンドポイント（`/oauth/token`）にリクエストするだけなので記載していません。
以上がアプリの流れです。

## おわりに
今回はToken Vaultの簡単なフローの説明と、Token Vaultの動きを確認できるアプリの動作について記事にしました。
この記事でToken Vaultのフローと、[以前の記事](https://web.maronn-room.com/maronn2/articles/token-vault-by-auth0)で触れたようにToken Vaultは確かにAIエージェントに限定した話ではないことが伝われば幸いです。
また、この記事で[以前の記事](https://web.maronn-room.com/maronn2/articles/token-vault-by-auth0)の理解を深める一助になれば幸いです。
ここまで読んでいただきありがとうございました。