---
title: "CIでOkta LDAP InterfaceにTOTPによるMFAを用いてリクエストを投げる"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Okta","MFA","TOTP","LDAP"]
published: true
published_at: 2025-08-16 22:00
---
## はじめに
認証するというと、人間の操作によるものだけではありません。
例えば、CI上で認証して操作をしたいというケースは往々にしてあります。
実際に[GitHub WorkflowでのOIDCによる認証についての記事](https://zenn.dev/kou_pg_0131/articles/gh-actions-oidc-aws)などは色々とあります。
ただ、OpenID Connectが主要であるがゆえに、OpenID Connect以外で認証を通す記事はそこまで多くないと思います。（主観です）
なので、この記事ではOktaを使用して、CI上でパスワードとOTPを使用した認証を行い、ユーザー情報を取得する方法を見ていきます。
なお、この記事はOktaのプロジェクトはすでに用意している前提で記載しています。
これから用意するつもりの方は[こちらのスクラップ](https://zenn.dev/link/comments/56ad0ef08a4a8f)から参考記事をもとにプロジェクトの作成を行ってください。（ドメイン料金はかかります）
## 流れの概要
今回試す機能の流れは以下の通りです。
![](/images/totp-in-ci-with-okta/okta-totp.png)
GitHub Action経由で、Oktaとの接続するためのコマンドを実行します。
Oktaとの接続はOkta LDAP Interfaceで行われ、ldapsearchコマンドを実行する時に渡す認証情報が正しければ、ユーザー情報を返すようにしています。
今回特に注目してほしいのが、認証情報にOne-Time Password（OTP）を渡している点です。
画像のフローを達成するには、ユーザーのパスワードさえあれば可能です。
しかし、パスワードだけでは漏洩した時のリスクが高くなります。
そこでパスワードに加えて、OTPを必須とすることできちんとしたフローからでないとアクセスできないようにしたのが今回の肝です。
これで単純なパスワード認証を避け、セキュリティの強度を上げることができます。
なお、実装については環境変数の読み込みからldapsearchコマンドの間に、色々と設定を実行しています。
その辺も踏まえて、この後実装部分で確認していきます。
なお、今回の記事は以下のコードを動かすことを前提としています。
https://github.com/maronnjapan/sample-id-app/tree/auth-otp-in-ci-by-okta
## GitHub Actionコードの全体像
まずは今回の主要部分である、GitHub Action用のyamlファイルを見ていきます。
```yaml
name: Okta TOTP Setup and LDAP Authentication
on:
  workflow_dispatch:
  push:
    # 自身のプロジェクトで試す時はブランチ名を動かしたいブランチ名に変更してください
    branches:
      - auth-otp-in-ci-by-okta

jobs:
  delete-existing-totp:
    runs-on: ubuntu-latest
    steps:
      - name: Delete existing TOTP factor
        run: |
          # 対象ユーザーの認証要素を取得する
          FACTORS=$(curl -s -X GET \
            "https://${{ secrets.OKTA_DOMAIN }}.okta.com/api/v1/users/${{ secrets.USER_ID }}/factors" \
            -H "Authorization: SSWS ${{ secrets.OKTA_API_TOKEN }}" \
            -H "Accept: application/json")
          # 取得した認証要素から、TOTPに該当する要素のIDを取得する
          TOTP_FACTOR_ID=$(echo $FACTORS | jq -r '.[] | select(.factorType=="token:software:totp") | .id') 
          # もしTOTP要素が存在する場合は削除する
          if [ ! -z "$TOTP_FACTOR_ID" ]; then
            curl -X DELETE \
              "https://${{ secrets.OKTA_DOMAIN }}.okta.com/api/v1/users/${{ secrets.USER_ID }}/factors/${TOTP_FACTOR_ID}" \
              -H "Authorization: SSWS ${{ secrets.OKTA_API_TOKEN }}";
          fi

  ldap-authentication:
    runs-on: ubuntu-latest
    needs: delete-existing-totp
    steps:
      - name: Install tools
        run: sudo apt-get update && sudo apt-get install -y oathtool ldap-utils

      - name: Perform LDAP authentication
        run: |
          # 対象ユーザーの認証要素としてTOTPを登録する(有効化作業が別途必要)
          # API仕様書（https://developer.okta.com/docs/api/openapi/okta-management/management/tag/UserFactor/#tag/UserFactor/operation/enrollFactor）
          ENROLL_RESPONSE=$(curl -s -X POST "https://${{ secrets.OKTA_DOMAIN }}.okta.com/api/v1/users/${{ secrets.USER_ID }}/factors" \
            -H "Authorization: SSWS ${{ secrets.OKTA_API_TOKEN }}" \
            -H "Accept: application/json" \
            -H "Content-Type: application/json" \
            -d '{ "factorType": "token:software:totp", "provider": "OKTA" }')

          # 登録したTOTPのIDとTOTP生成のためのシークレットを取得する
          FACTOR_ID=$(echo $ENROLL_RESPONSE | jq -r '.id') 
          OKTA_TOTP_SECRET=$(echo $ENROLL_RESPONSE | jq -r '._embedded.activation.sharedSecret')

          # OTPを生成する
          OTP=$(oathtool --totp -b "$OKTA_TOTP_SECRET") 
          # 登録したTOTPによる認証を有効化するために、生成したOTPを使用して有効化リクエストを送信する
          # API仕様書（https://developer.okta.com/docs/api/openapi/okta-management/management/tag/UserFactor/#tag/UserFactor/operation/activateFactor）
          curl -s -X POST "https://${{ secrets.OKTA_DOMAIN }}.okta.com/api/v1/users/${{ secrets.USER_ID }}/factors/${FACTOR_ID}/lifecycle/activate" \
            -H "Authorization: SSWS ${{ secrets.OKTA_API_TOKEN }}" \
            -H "Accept: application/json" \
            -H "Content-Type: application/json" \
            -d "{ \"passCode\": \"${OTP}\" }"

          # TOTP有効化後、新しいOTPウィンドウまで待機
          echo "Waiting for new OTP window..."
          sleep 35

          # リトライ処理をするためにループ処理を記載
          for attempt in 1 2 3; do
            echo "LDAP authentication attempt $attempt"
            
            # 認証用のOTPを生成する
            OTP_FOR_LDAP=$(oathtool --totp -b "$OKTA_TOTP_SECRET")

            # Okta LDAP Interfaceにリクエストを送信して認証を試みる
            # ldapsearchが成功した場合はループを抜ける
            ldapsearch -H "ldaps://${{ secrets.OKTA_DOMAIN }}.ldap.okta.com:636" \
              -D "uid=${{ secrets.LDAP_UID }},dc=${{ secrets.OKTA_DOMAIN }},dc=okta,dc=com" \
              -w "${{ secrets.USER_PASSWORD }},${OTP_FOR_LDAP}" \
              -b "dc=${{ secrets.OKTA_DOMAIN }},dc=okta,dc=com" \
              "(objectClass=person)" && {
                echo "LDAP authentication successful"
                break
              }
            
            echo "LDAP authentication failed (attempt $attempt)"
            [ $attempt -lt 3 ] && {
              echo "Waiting 2 seconds before retry..."
              sleep 2
              continue
            }

            # 全てのLDAP認証試行が失敗した場合はエラーを出力
            echo "All LDAP authentication attempts failed"
            exit 1
          done
```
大きく分けると、以下の二つを行っています。
- 対象ユーザーにTOTPの認証ができるように設定
- 上記の設定で作成したOTPを使用しつつ、Okta LDAP Interface経由でユーザー情報を取得する

その他にも、最初に既存のTOTP認証設定を削除することや、TOTPの兼ね合い上リクエストが失敗する可能性があるのでリトライ処理を行うことを行っています。
ですが、特に注目してほしいのは対象ユーザーにTOTPでの認証ができるようにするための
```sh
ENROLL_RESPONSE=$(curl -s -X POST "https://${{ secrets.OKTA_DOMAIN }}.okta.com/api/v1/users/${{ secrets.USER_ID }}/factors" \
            -H "Authorization: SSWS ${{ secrets.OKTA_API_TOKEN }}" \
            -H "Accept: application/json" \
            -H "Content-Type: application/json" \
            -d '{ "factorType": "token:software:totp", "provider": "OKTA" }')

# 登録したTOTPのIDとTOTP生成のためのシークレットを取得する
FACTOR_ID=$(echo $ENROLL_RESPONSE | jq -r '.id') 
export OKTA_TOTP_SECRET=$(echo $ENROLL_RESPONSE | jq -r '._embedded.activation.sharedSecret')

# OTPを生成する
OTP=$(oathtool --totp -b "$OKTA_TOTP_SECRET") 
# 登録したTOTPによる認証を有効化するために、生成したOTPを使用して有効化リクエストを送信する
curl -s -X POST "https://${{ secrets.OKTA_DOMAIN }}.okta.com/api/v1/users/${{ secrets.USER_ID }}/factors/${FACTOR_ID}/lifecycle/activate" \
            -H "Authorization: SSWS ${{ secrets.OKTA_API_TOKEN }}" \
            -H "Accept: application/json" \
            -H "Content-Type: application/json" \
            -d "{ \"passCode\": \"${OTP}\" }"
```
と実際にOktaへ通信する
```sh
OTP=$(oathtool --totp -b "$OKTA_TOTP_SECRET")
# ...省略
ldapsearch -H "ldaps://${{ secrets.OKTA_DOMAIN }}.ldap.okta.com:636" \
              -D "uid=${{ secrets.LDAP_UID }},dc=${{ secrets.OKTA_DOMAIN }},dc=okta,dc=com" \
              -w "${{ secrets.USER_PASSWORD }},${OTP}" \
              -b "dc=${{ secrets.OKTA_DOMAIN }},dc=okta,dc=com" \
              "(objectClass=person)"
```
を把握してもらえれば十分かなと思います。
Okta LDAP InterfaceでOTPを使った認証は、[ドキュメント](https://help.okta.com/en-us/content/topics/directory/ldap-interface-mfa.htm)にも記載がありますが、`ユーザーパスワード,OTP`とカンマでそれぞれの認証情報をつなげばよいです。
これで実行の準備ができました。
なお、GitHub Actionの実行条件として以下の設定をしています。
```yaml
branches:
	- auth-otp-in-ci-by-okta
```
これはサンプルコードで動かすための条件なので、自分のプロジェクトでサンプルコードを動かす場合は、適宜動かしたブランチ名を指定してください。
## Oktaの準備
GitHub Actionの主要部分はできたので、CIを適切に動かすためのOkta側の設定を行います。
### Okta LDAP Interfaceの有効化
今回Oktaへの通信はLDAPを想定しています。
先ほどGitHub Actionで記載したldapsearchコマンドはLDAPサービスに対しての操作なので、必然的に認証もLDAP通信に乗っ取る必要があります。
LDAPによる通信と聞くと、LDAPサーバーを用意し、それをOktaと統合する必要がありそうな気がします。
ですが、必ずしも上記作業は必要なく、[Okta LDAP Interface](https://help.okta.com/oie/ja-jp/content/topics/directory/ldap-interface-main.htm)を使用すればOktaだけでLDAP上での認証が可能となります。
そのため、まずはダッシュボードにてOkta LDAP Interfaceを準備します。
Oktaの管理画面から、ディレクトリ→ディレクトリ統合をクリックします。
ディレクトリ統合画面で「ディレクトリを追加」ボタンをクリックし、LDAPインターフェイスを追加を選択します。
![](/images/totp-in-ci-with-okta/2025-08-17_21h37_38.png)
すると、Okta LDAP Interfaceが設定されます。
とても簡単にOktaをLDAPサーバーぽく扱えるので、便利な機能ですね。
### 認証の設定とCI用のユーザーを追加
Okta LDAP Interfaceを設定したので、GitHub ActionからLDAPによる通信でユーザー情報を取得する準備はできました。
後は、取得するユーザーの設定やセキュリティのためにMFAを要求する設定をします。
ですが、Okta LDAP Interface以外の準備は以下リポジトリにてIaC化しています。
https://github.com/maronnjapan/sample-id-app/tree/auth-otp-in-ci-by-okta
そのため、上記コードをクローンしてもらいopentofuディレクトリ配下にて、[こちらの記事のOpenTofu用のアプリの作成](https://zenn.dev/maronn/articles/practice-okta-swa-app#opentofu%E7%94%A8%E3%81%AE%E3%82%A2%E3%83%97%E3%83%AA%E3%81%AE%E4%BD%9C%E6%88%90)部分の設定を行ってください。
なお、上記記事に加えて、ユーザーのパスワード(`ci_user_password`)も設定する必要があるので、任意のパスワードを追加してください。
設定が終わったら、`terraform apply`か`tofu apply`でOktaに適用してください。
これでOktaの準備は完了するはずですが、どこに何が設定されるか把握した方が良いと思いますので、実際に記載した内容の主要部分を記載します。

```tf
# CIシステム用のユーザーが所属するグループの作成
# ドキュメント：https://search.opentofu.org/provider/opentofu/okta/latest/docs/resources/group
resource "okta_group" "CI_group" {
  name        = "CI Group"
  description = "CIで動かすためのグループです"
}

# CIシステム用ユーザーの作成
# ドキュメント：https://search.opentofu.org/provider/opentofu/okta/latest/docs/resources/user
resource "okta_user" "ci_system" {
  first_name = "CI"
  last_name  = "System"
  login      = "ci.system@example.com"
  email      = "ci.system@example.com"
  password   = var.ci_user_password
}

# CIグループにユーザーを追加
# ドキュメント：https://search.opentofu.org/provider/opentofu/okta/latest/docs/resources/group_memberships
resource "okta_group_memberships" "CI_membership" {
  group_id = okta_group.CI_group.id
  users = [
    okta_user.ci_system.id,
  ]
}


# グローバルセッションポリシーの設定
# ドキュメント：https://search.opentofu.org/provider/opentofu/okta/latest/docs/resources/policy_signon
resource "okta_policy_signon" "LDAP_TOTP_policy" {
  name        = "LDAP TOTP Policy"
  status      = "ACTIVE"
  description = "Policy for LDAP users with TOTP"
  # CI用に作成したユーザーグループのみに適用させる
  groups_included = ["${okta_group.CI_group.id}"]
}

# グローバルセッションポリシー内のルールの設定
# ドキュメント：https://search.opentofu.org/provider/opentofu/okta/latest/docs/resources/policy_rule_signon
resource "okta_policy_rule_signon" "LDAP_TOTP_rule" {
  name      = "LDAP TOTP Rule"
  status    = "ACTIVE"
  policy_id = okta_policy_signon.LDAP_TOTP_policy.id

  # 認証対象の限定
  # セッションポリシーを作る時に、対象ユーザーは限定しているが予期せぬ影響を防ぐために
  # 認証タイプをLDAP_INTERFACEに限定する
  authtype = "LDAP_INTERFACE"

  # セッションの有効期限とアイドル時間の設定
  # CI上での認証しか想定していないため、短い時間に設定
  session_lifetime = 5
  session_idle     = 1

  # 二要素認証の設定
  # サインインごとに常にMFAを要求させる
  mfa_prompt   = "ALWAYS"
  mfa_required = true
}
```
やっていることは以下の4つです。
- CI用のユーザー作成
- 上記ユーザーが所属するグループを作成
- [グローバルセッションポリシー](https://help.okta.com/oie/ja-jp/content/topics/identity-engine/policies/about-okta-sign-on-policies.htm)の追加とポリシーの対象に上記グループを紐づけ
- 上記ポリシーに二要素認証を要求するルールを追加

これによって、CIでの認証を行うことができ、さらにLDAPによる通信の時にのみMFAを強制することができます。
さらにユーザーグループをグローバルセッションポリシーに紐づけることで、CI用に作成したユーザーのみ上記設定が適用されるようになっています。
### APIトークンの取得
最後にCI上で各種エンドポイントを実行するために、APIトークンを取得します。
Oktaの管理画面にアクセスし、セキュリティ→APIにアクセスします。
APIの画面で、トークンのタブに移動し「トークンの作成」ボタンをクリックします。
すると、トークン名の入力や、使用可能なIPの設定を行うとトークンが生成されます。
最終的に以下のようなトークンが生成されたら、その値を控えておきます。
![](/images/totp-in-ci-with-okta/2025-08-17_23h25_47.png)
以上でOkta側の設定も完了です。
最後にGitHubでの設定を行います。
## GitHubの準備
最後にCIで動かすために必要なシークレットの準備を行います。
最初展開した[コード](https://github.com/maronnjapan/sample-id-app)をプッシュしたリポジトリにて、Settingsタブをクリックします。
クリックしたら、サイドメニューからSecrets and variablesをクリックし、Actionsを選択します。
すると、画像のようなシークレットを追加する場面が出てきます。
![](/images/totp-in-ci-with-okta/2025-08-16_11h43_22.png)
後はシークレットを追加するボタンを押して、以下のシークレット名とその値を追加します。
- LDAP_UID：`ci.system@example.com`を設定
- OKTA_API_TOKEN：「APIトークンの取得」にて取得したトークンを設定
- OKTA_DOMAIN：以下画像の赤枠で囲んだdcの値を設定

![](/images/totp-in-ci-with-okta/2025-08-17_23h29_47.png)
- USER_ID：Oktaの管理画面でディレクトリ→ユーザーから、先ほど作成したユーザーの詳細画面にアクセスし、URL末尾の文字列を設定（以下画像の赤枠部分）

![](/images/totp-in-ci-with-okta/2025-08-17_23h33_38.png)
- USER_PASSWORD：terraform.tfvarsで入力したユーザーのパスワードを設定

ここまで、行けば準備は全て完了です。
後は、コードをCI実行用のブランチにコミットします。
すると、以下のようなjobが実行されます。
![](/images/totp-in-ci-with-okta/2025-08-17_23h37_32.png)
そして、ldap-authentication内の「Perform LDAP authentication」にて最終的に以下のようなログが出力されていれば、認証ができておりさらにユーザー情報も取得できています。
```
LDAP authentication attempt 1
# extended LDIF
#
# LDAPv3
# base <dc=***,dc=okta,dc=com> with scope subtree
# filter: (objectClass=person)
# requesting: ALL
#

# ***, users, ***.okta.com
dn: uid=***,ou=users,dc=***,dc=okta,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: ***
uniqueIdentifier: ***
organizationalStatus: ACTIVE
givenName: CI
sn: System
cn: CI System
mail: ***

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
LDAP authentication successful
```
今回はユーザー情報を取ってくるだけでしたが、この仕組みを活かして別途他の方法で活用していただければ幸いです。
## やってみた感想
一通り試してみたので、設定してみた感想を記載します。
一番に思ったのは、TOTPめんどくさいということです。
ダッシュボードでユーザーにTOTPの認証要素を追加する方法はついぞ分からなかったです。
APIを実行することでなんとか設定できましたが、そこまで到達するのは結構時間がかかりました。
上記のように、どうすればTOTPを設定できるかを見つけるのはしんどかったです。
また、TOTPを設定した後の挙動も難しかったです。
TOTPをアクティベートした後すぐに、TOTP認証を実行するとエラーになります。
なので、CI上ではスリープすることで待機し、TOTPが有効となった後実行するようにしています。
さらにタイミングによっては、TOTPがちゃんと動いてもOTPが切り替わったことでタイミング的にエラーとなる場合がありそうです。
なので、CI上ではリトライ処理を入れることで対応しました。
これらのことから、やり方はわかれど期待動作しなかったときに何が起きているかを把握することが難しかったです。
つまり、総じてTOTP面倒だったということです。
色々と挙動の理解を深めることができたのは良かったですけどね。
## (余談) 今回あきらめたこと
ここまででLDAPによる通信で、二要素認証を必須にすることでセキュリティの安全性を高めてきました。
そして、TOTPに使うためのシークレットもCIを動かす度に再設定するようにし、定期的な交換もできるようにしています。
なので、単純な一度作ってパスワードをずっと使い回すことはなくなり、セキュアにはなっています。
ですが、もっと安全に行うためにできることは残っています。
それは、TOTPの取得・削除・追加に使っているAPIトークン部分です。
APIトークンは良くも悪くも全ての操作を行うことができます。
実際にAPIトークンを作成したら、ロールに`Super Admin`が適用されています。
さらに、APIトークンには有効期限がないため、APIトークンが盗まれたら、やりたい放題されます。
なので、実行できるAPIを最小限にしたり、有効期限を設けるためにOAuth2によるアクセストークンを発行するとよりセキュアにできます。
というか、本来はAPIトークンではなく、秘密鍵/公開鍵を使いアクセストークンを取得するべきです。
ただ、今回はそこまでやると手間がかかりしんどかったので、さぼらせていただきました。
この辺はまた気力があるタイミングで設定してみて、記事にできればと思います。
## おわりに
今回はOkta LDAP InterfaceをOTPを用いて認証し、ユーザー情報取得するようにしました。
CI上で二要素認証を設定できるのは便利だなと思う反面、やはり画面操作がないと事象の把握は難しいなと感じます。
ただ、実際に触ってみてセキュアにするために必要なことが以前より言語化できたのは良かったです。
ここまで読んでいただきありがとうございました。