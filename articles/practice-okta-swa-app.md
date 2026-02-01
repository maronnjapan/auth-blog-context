---
title: "OktaのSWAを体験してみた"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["okta","swa","OpenTofu"]
published: true
---
## はじめに
OktaにはSAML、Web Services Federation（WS-Fed）、OpenID Connect（OIDC）などのフェデレーションプロトコルをサポートしない外部のWebアプリケーションにシングルサインオン（SSO）の機能を提供するための仕組みがあります。
それが、[Secure Web Authentication（SWA）](https://support.okta.com/help/s/article/What-is-Secure-Web-Authentication-SWA?language=ja)です。
今回はこのSWAをOktaで構築してみて、Webアプリケーションと統合してみました。
昨今の事情を考えるとあまり導入する機会は多くないですが、誰かの参考になりましたら幸いです。
なお、Oktaのアカウントをすでに作成されていて、Okta organizationsはすでに存在する前提で記載しています。
他の記事頼みですが、作成方法の流れを[スクラップ](https://zenn.dev/link/comments/56ad0ef08a4a8f)に記載しています。（ちなみに私の認識では完全無料は無理です）

また、Okta Learningの閲覧が可能であることも前提としていますので、その点もご了承ください。
## 完成したログインの流れ
まずはアプリの流れを確認します。
動きは以下の通りです。
![](/images/practice-okta-swa-app/auth-login-by-swa-demo.gif)
Webアプリのログイン画面にアクセスすると、Oktaに保存されているユーザー情報が入力され、自動でログインボタンがクリックされます。
そして、ログインが成功したので、ログイン完了画面に遷移しています。
正直何をしているかイメージはつきにくいと思いますので、その場合はOktaラーニングの[Okta Browser Pluginをセットアップする](https://learning.okta.com/set-up-the-okta-browser-plugin-ja)の1:41からを参照してください。
以上動きを確認したので、実際に動かしていきます。
なお、これから記載していく内容は下記のコードを前提にしています。
https://github.com/maronnjapan/sample-id-app/tree/practice-okta-swa-app
一からであれば追加で説明する必要がある部分も本来はありますが、その説明は上記リポジトリを前提としているため省略させていただきます。
ご了承ください。
また、Webアプリケーション側のログイン処理は今回適当です。
ユーザー名とパスワードさえあればその時点でログインするという、おぞましい実装なのでちゃんとしたログイン処理をしていないことはご了承ください。
## Oktaの準備
ここではSWAによるログインを構築するために行うOktaの設定を行っていきます。
ただし、本来ダッシュボード操作だけでもできますが、ここではOpenTofuによるIaCで管理します。
OpenTofuで行っていますが、Terraformでも動くはずなのでTerraformを使用されている方も同様の手順で大丈夫なはずです。
### OpenTofu用のアプリの作成
まずは[Oktaのドキュメント](https://developer.okta.com/docs/guides/terraform-enable-org-access/main/)をもとに、OpenTofuを実行するためのアプリケーションを作成します。
管理画面のダッシュボードのサイドメニューから、アプリケーション→アプリケーションへ遷移します。
![](/images/practice-okta-swa-app/2025-08-10_12h32_21.png)
遷移後は「アプリ結合を作成」ボタンをクリックします。
クリックすると以下のモーダルが表示されるので、「API サービス」を選択し作成します。
![](/images/practice-okta-swa-app/2025-08-10_12h34_50.png)
名前は任意のものを設定したら、作成したアプリケーションが表示されます。
アプリケーションが表示された後は以下の作業を行います。
- クライアント認証の方式をクライアントシークレットから、公開鍵/暗号鍵方式に変更

クライアント認証については、[Prividerの設定にクライアントシークレットの設定がない](https://search.opentofu.org/provider/opentofu/okta/latest#argument-reference)ので、公開鍵/暗号鍵にする必要があります。
セキュリティ面を考慮してシークレットによるクライアント認証は不可にしているのだと推測しています。
以下の記事は公開鍵/暗号鍵方式の利点が完結にまとまっているので、良ければ参照ください。
https://www.criipto.com/blog/private-key-jwt
設定を変更したら鍵を作る必要があります。
作った秘密鍵はPEM形式でコピーし、任意の場所に保存します。
今回はお試しなので、PC端末にsecret.keyファイルを作り、そこにPEM形式の秘密鍵をコピーしました。
また、鍵を作成した後以下のようにどの鍵かを示すIDである`KID`が表示されます。
![](/images/practice-okta-swa-app/2025-08-10_18h02_03.png)
こちらは後ほど使うので、どこかに控えておいてください。
ここまでできたら以下のようなプロバイダーを定義したtfファイルを作成します。
```tf
terraform {
  required_providers {
    okta = {
      source  = "okta/okta"
      version = "~> 5.2.0"
    }
  }
}

provider "okta" {
  org_name       = var.okta_org_name
  base_url       = var.base_url
  client_id      = var.client_id
  scopes         = var.scopes
  private_key_id = var.private_key_id
  private_key    = file(var.private_key_file_path)
}
```
`provider "okta"`での各種値ですが、terraform.tfvars.exampleファイルを用意しているので、ファイル名末尾の`.example`を削除した上で一致する値を設定してください。
なお、org_nameとbase_urlですが、例えば管理画面のURLが`https://org-name-admin.okta.com`の場合、org_nameはサブドメインから`-admin`を除いた`org-name`になります。
base_urlはドメイン部分の`okta.com`になります。
これでプロバイダーの準備は完了です。
ちなみに、secret.keyファイルはプロバイダーを記載したtfファイルと同じ階層にいますので、その点はご注意ください。
###  Template Plugin Appの追加
SWAを使った認証を行う場合、統合を簡単にできるテンプレートがOktaに用意されています。
それがTemplate Plugin Appです。
OpenTofuには上記をコードで作成する機能があるので、それを使って定義します。
任意のtfファイルを作成し、以下のようなコードを記載します。
```tf
resource "okta_app_swa" "SampleSWAApp" {
  label          = "SampleSWAApp"
  button_field   = "#login-button"
  password_field = "#password"
  username_field = "#username"
  url            = "http://localhost:5173/login"
}

resource "okta_app_group_assignment" "everyoneInSWAApp" {
  app_id   = okta_app_swa.SampleSWAApp.id
  group_id = "ユーザーグループのID"
}
```
[okta_app_swaリソース](https://search.opentofu.org/provider/opentofu/okta/latest/docs/resources/app_swa)何の各属性の詳細はOktaラーニングの動画を確認いただけると幸いですが、簡単に説明を記載します。
- label：Template Plugin Appの名前。任意のもので良いです
- button_field：ログインボタンを識別できる値
- password_field：パスワードフォームを識別できる値
- username_field：ユーザー名を識別できる値
- url：ログインURL

`button_field`、`password_field`、`username_field`は対象要素に存在する[CSSセレクター]を指定する形になります。
今回は各要素のid属性を値に設定しています。
[okta_app_swaリソース](https://search.opentofu.org/provider/opentofu/okta/latest/docs/resources/app_swa)でアプリケーションは作成できるのですが、このアプリケーションに属するユーザーやグループを設定しないと、ログイン対象者とみなされずSWAの諸々の機能が動きません。
なので、アプリケーションに対して誰が対象かを紐づける必要があります。
それを担当するのが、[okta_app_group_assignmentリソース](https://search.opentofu.org/provider/opentofu/okta/latest/docs/resources/app_group_assignment)です。
このリソース内の`app_id`に作成したアプリケーションのidを設定し、`group_id`にはグループIDを設定します。(グループIDの確認方法は後述します)
なお、確認した範囲ではOpenTofuにてアプリケーションに対してユーザーを割り当てる機能はありませんでした。
おそらくですが、ユーザーでの紐づけを許してしまうと管理が複雑になるので、意図的に機能を提供していないかと思います。（あくまで推測です）
これで、コードは記載できたので、後はグループIDを取得しそれを設定します。
### グループIDの取得
アプリケーションに割り当てるためのOktaグループIDを取得します。
ここでは、基本的に存在するユーザー全てが所属するEveryoneグループのIDを取得し、それをアプリケーションに割り当てます。
本来はEveryoneグループではなく、目的に応じたグループを作成し、それを割り当てるのが適切です。
ですが、今回は動かすことを最優先にしたので、予め存在するユーザー全てが属するEveryoneグループにしています。
経緯を書いたので、実際にIDを取得します。
まず、Oktaの管理ダッシュボードのサイドメニューにあるディレクトリ→グループにアクセスします。
グループ一覧があるので、その中にあるEveryoneグループを選択します。
遷移したURLのパスの末尾がランダムな文字列かと思います。
それがグループIDなので、値を取得します。
IDを取得したら、先ほど作成したtfファイルの`group_id`にその値をセットします。
後は、`tofu apply`を実行して、Okta側に反映させます。
反映後、Oktaの管理画面ダッシュボードのアプリケーション→アプリケーションに遷移して、SampleSWAAppという名前のアプリケーションがあればOktaのダッシュボードでの準備は完了です。
### Okta Browser Pluginのインストール
Okta側の準備はできたのですが、WebアプリをOktaで作成したSWA用のアプリケーションと統合するにはOkta Browser Pluginという拡張機能が必要です。
[こちら](https://help.okta.com/eu/ja-jp/content/topics/end-user/plugin-download_install.htm)を参考にしつつブラウザへ拡張機能を導入し、自身のOktaアカウントでログインします。
ログインした後、拡張機能をクリックすると画像のような自身に割り当てられているOktaアプリケーションが表示されます。
![](/images/practice-okta-swa-app/2025-08-10_21h07_39.png)
ここに先ほどOktaに作成したSampleSWAAppが表示されていれば、Oktaに関連する準備はほぼ完了しました。
## アプリケーションの設定
最後にSWAを用いたログインを体験するためのWebアプリを実行します。
といっても[今回使用しているコード](https://github.com/maronnjapan/sample-id-app/tree/practice-okta-swa-app)のlogin-app配下に移動し、`npm i`実行後、`npm run dev`でアプリを起動するだけです。
その後、`http://localhost:5173/login`にアクセスしてもらえれば、最初に展開したデモの動きをすると思います。
なお、Oktaがログインフォームなどを識別するための実装はlogin-app/src/index.tsxをLoginFormコンポーネントを確認してください。
```tsx
  <div style="max-width: 400px; margin: 100px auto; padding: 20px; border: 1px solid #ddd; border-radius: 5px;">
    <h2>ログイン</h2>
    <form method="post" action="/login">
      <div style="margin-bottom: 15px;">
        <label htmlFor="username" style="display: block; margin-bottom: 5px;">ユーザー名:</label>
        <input
          type="text"
          id="username"
          name="username"
          required
          style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 3px;"
        />
      </div>
      <div style="margin-bottom: 15px;">
        <label htmlFor="password" style="display: block; margin-bottom: 5px;">パスワード:</label>
        <input
          type="password"
          id="password"
          name="password"
          required
          style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 3px;"
        />
      </div>
      <button
        type="submit"
        id="login-button"
        style="width: 100%; padding: 10px; background-color: #007bff; color: white; border: none; border-radius: 3px; cursor: pointer;"
      >
        ログイン
      </button>
    </form>
  </div>
```
二つのinput要素と一つのbutton要素にそれぞれ、先ほどtfファイルに記載したIDセレクターと同じ名前が存在していると思います。
これによって、Oktaはフォームを識別することができます。
## 流れを組んでみた感想
実際に構築してみた感想を記載します。
最初に思ったのは、SWAはなるべくユーザーの操作を減らす工夫がされているなということです。
SAMLやOpenID Connectなどのフローはユーザー管理を基本Okta側に任せることができます。
一方、上記仕様を使えないWebアプリは認証情報を自前で管理、取得する必要があります。
昨今の流れを考えると、あまりWebアプリ側で頑張るのは適切ではありません。
そこで、OktaのSWAはリクエストで受け取ったユーザー情報の扱いはWebアプリ側に任せていそうですが、ユーザー情報の入力はOkta側に委ねることができています。
なので、OpenID Connectなどに比べるとどうしても制約が多いですが、その中でも可能な限りSSOを達成するための工夫があると感じています。
ですが、SWAによるログインは積極的に使用すべきだとは思わないです。
Okta側も言っていますが、あくまでSAMLやOpenID Connectなどが使えないレガシーなシステムを扱う時の最終選択かなと思います。
結局ユーザー情報をWebアプリ側が扱わないといけないのは個人的にはかなり致命的ですし、拡張機能を入れないといけないのも場合によっては障壁になります。
このことから、仕組みとしては感心することは多々あれどやはりSWAによるログインは最終手段だなというのは実感としてあります。
## （余談）苦労したこと
余談として、苦労したことも書いていきます。
まず何より苦労した（今も苦労している）のは、OpenTofuで内容を反映させる時のアクセストークンについてです。
TerraformやOpenTofuのapplyコマンドを実行する前に` TF_LOG=DEBUG`を付与して実行してみてください。
ログを見ると、アクセストークンを取得するリクエストはclient_assertionが付与することでトークンを取得しようとしています。
するとかなりの確率で以下のエラーが確認されます。
```json
{
	 "error": "invalid_client",
	"error_description": "The client_assertion token has an expiration too far into the future."
}
```
これはclient_assertionに設定したJWTの時間が今より後に設定しすぎているため、エラーになっています。
どうやら1時間以上だとエラーになるみたいです。
じゃあclient_assertionを生成する時の有効期限を設定できないか確認したところ、以下のソースコードを見つけました。
https://github.com/okta/terraform-provider-okta/blob/master/sdk/v2_requestExecutor.go#L349
有効期限が1時間固定で設定されていますね。
なので、有効期限を変更することはできなさそうです。
じゃあ、先ほどのエラーは解消できないのでは？となっています。
ログに表示されたclient_assertionを[jwt.io](https://www.jwt.io/ja)などでデコードしてほしいのですが、expは`1754884336`のように設定されているので、秒単位な気がします。
秒単位なら、client_assertionを生成してすぐにトークンのリクエストをすると、1時間以上と判定されてエラーになりそうですし、実際になっています。
この問題が解消できず、applyコマンドは9割くらい失敗している状況です。
ごくたまにできるのですが、基本できないのでかなりストレスは溜まっています。
もし解消方法をご存知の方がいましたら、教えてください。お願いします。
苦労した大半は、client_assertionの有効期限がメインですが、SWAの自動ログインの挙動の理解も苦労しています。
Webアプリケーションにアクセスした時点で自動でログインする場合もあれば、以下のように画面右上に表示されたサインインボタンを押さないとフォームに値が入力されないケースもあります。
![](/images/practice-okta-swa-app/2025-08-11_12h04_57.png)
![](/images/practice-okta-swa-app/2025-08-11_12h05_44.png)
これらの動きになる条件がよくわからない状態です。
最悪私の場合は調べればいいのですが、実際にユーザーへ使ってもらうことを考えるとこの挙動の違いはちょっとストレスだなとは感じました。
以上が苦労したことです。
## おわりに
今回はOktaのSWAの仕組みを構築して、実際に動かしてみました。
レガシーなシステムでも、SSOができるのは魅力的だなと感じました。
一方で、やはり開発者側・ユーザー側それぞれやることが多くなるので、可能な限り導入は控えたい気持ちが強いです。
とはいえ、一度試してみないと気持ちがわからなかったと思うので、体験できたのは良かったです。
ここまで読んでいただきありがとうございました。
## 参考資料
https://developer.okta.com/docs/guides/terraform-enable-org-access/main/
https://help.okta.com/oie/ja-jp/content/topics/apps/apps_app_integration_wizard_swa.htm
https://help.okta.com/eu/ja-jp/content/topics/end-user/plugin-download_install.htm
https://learning.okta.com/path/create-app-integrations-jp/configure-swa-sso-ja/2165023/scorm/1i3lrrhckh7sw
https://learning.okta.com/path/create-app-integrations-jp/set-up-the-okta-browser-plugin-ja
https://info.nextmode.co.jp/blog/introduction-to-terraform-okta-provider