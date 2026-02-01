---
title: "衝撃！Auth0のダッシュボードをログインするのにMFAが必須化となりました"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Auth0", "MFA"]
published: true
---

## はじめに

先日 Auth0 のダッシュボードへログインしたら、下記のようなお知らせが右下に表示されました。
![2024-01-14_15h36_41.png](/images/auth0-dashboard-login-require-login/2024-01-14_15h36_41.png)
よくあるお知らせなのかなと思って眺めていたのですが、ちゃんと読んでみると「2024 年 2 月 1 日から Auth0 へログインするには MFA の設定が必須だよ。」というアナウンスでした。
MFA の設定する必要があるのは面倒くさいなと思いつつ、ちゃんとやるかと思い案内に従いました。
そこで出てきたのが以下の画面です。
![2024-01-09_12h02_38.png](/images/auth0-dashboard-login-require-login/2024-01-09_12h02_38.png)
衝撃を受けました。
なんと、Passkey 関連の記事で紹介したセキュリティキーか指定のアプリをスマホにインストールする方法しかありません。
これはかなり面倒くさいなと思い、他の方法がないかを探しました。
同様の疑問を抱えている人は[Auth0 のコミュニティ](https://community.auth0.com/t/please-tell-me-about-requiring-mfa-when-logging-into-the-dashboard/124432)にもいて、その回答を確認する限り上記 4 つの方法以外は存在しなさそうでした。
一応、[ドキュメント](https://auth0.com/docs/get-started/manage-dashboard-access/add-change-remove-mfa)を確認すると SMS 認証もできるのですが、有料プランを契約していないと使用できなさそうで、無料プランで契約している私は使えません。
となると、上記 4 つの中から選択する必要があります。
といっても、セキュリティキーについては二つのセキュリティキーが登録できるというだけなので、実質方法としては 3 つです。
そして、セキュリティキーについては[Yubkey](https://www.yubico.com/yubikey/?lang=ja)などで別途キーを購入したり、セキュリティキー用の装置を購入し別途[OpenSK](https://github.com/google/OpenSK)などで設定したりする必要があります。
どちらもかなり面倒で、お金もかかるので今回セキュリティキーの選択肢はなしです。
そこで今回は Auth0 Guardian と Google authenticator それぞれを使った方法で MFA を有効にする手段を見ていきます。
仰々しく書きましたが対象のアプリさえインストールしてしまえば、設定自体はとても簡単です。

## Auth0 Guardian による MFA の有効化

まずは Auth0 Guardian を使った方法です。
この認証方式は以下の通りです。

1. QR コードを Auth0 Guardian で読みとり、認証するユーザーを登録します。
2. 登録をした後 Auth0 にログインすると、Auth0 Guardian に認証して良いかの通知が届きます。
3. 通知を許可すればログインすることが可能になります。

では実際にやっていきます。
まず、Auth0 Guardian([Apple 版](https://apps.apple.com/jp/app/auth0-guardian/id1093447833) [Android 版](https://play.google.com/store/apps/details?id=com.auth0.guardian&hl=en_US))をスマホにインストールします。
アプリを開くと下記画像のような画面が表示されるので、赤枠内の追加ボタンをタップします。
![2024-01-14_16h23_12.png](/images/auth0-dashboard-login-require-login/2024-01-14_16h23_12.png)
するとカメラが起動して、QR コードを読み取る状態になります。
そしたら、Auth0 のダッシュボードに移動して右上のアイコンをクリックします。
すると、「Your profile」という項目が表示されるのでクリックします。
Multi-Factor Authentication という項目があるので、その中の「**Auth0 Guardian**」の列の add ボタンをクリックします。
![2024-01-09_12h02_38.png](/images/auth0-dashboard-login-require-login/2024-01-09_12h02_38%201.png)
以下のようなウィンドウが表示されるので、「続ける」を押します
![2024-01-14_17h25_59.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h25_59.png)
画面上に R コードが表示されるので、先程インストールした Auth0 Guardian で QR コードを読みとります。
![2024-01-14_17h26_49.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h26_49.png)
すると、アプリ側に以下画像の Auth0 部分のような項目が追加されます。
![2024-01-14_17h30_00.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h30_00.png)
後は Auth0 のダッシュボードにログインする際にアプリにログイン可否の通知が行くので、自分でログインした場合は通知を許可すればログインが完了します。
ちなみに、Auth0 Guardian の登録が完了した場合、QR コードを表示していたウィンドウは以下のように変更します。
![2024-01-14_17h33_32.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h33_32.png)
これは仮に Auth0 Guardian を登録したデバイスを無くした場合でもログインするためのコードとなっています。
なので、これをコピーしてどこかに保存しておく必要があります。
ここまで行うと、先程の設定画面は以下のようになります。
![2024-01-14_17h36_32.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h36_32.png)
設定が完了したので、試しにログアウトしてログインしてみます。
パスワードとメールアドレスを入力してログインすると下記画面が表示されます。
![2024-01-14_17h37_45.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h37_45.png)
後は Auth0 Guardian へ通知が来るので、それを許可すると勝手にログインされます。
なお、デバイスを無くし、リカバリーキーなどを入力する場合は「別の方法を試す」をクリックすればコード入力でのログインができます。
以上で Auth0 Guardian での MFA の設定が完了です。
では、次に Google Authenticator での MFA 設定を行います。

## Google Authenticator による MFA の有効化

Google Authenticator の場合も Auth0 Guardian とほぼ同じですが、一点明確な違いがあります。
それは Auth0 Guardian は通知を許可すればログインできましたが、Google Authenticator の場合はアプリで表示されているコードを手で入力する必要があります。
上記違いはありますが、全体的には大体一緒なので早速登録します。
まず、スマホに Google Authenticator([Apple 版](https://apps.apple.com/jp/app/google-authenticator/id388497605) [Android 版](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=ja&gl=US))をインストールします。
その後、Auth0 のダッシュボードで Google authenticator or similar の列の Add ボタンをクリックします。
![2024-01-14_17h46_34.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h46_34.png)
なお、similar とあるように Google authenticator のみしか登録ができないわけではありません。
似たようなアプリであれば任意のものを使用してください。
新しいウィンドウが開き、QR コードが表示されるので Google authenticator 読みとりアカウントを登録してください。
![2024-01-14_17h49_25.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h49_25.png)
ただし、今回は登録した後に Google authenticator で表示されている 6 桁の数字を QR コードの下のフォームに入力する必要があります。
入力が完了すれば、MFA の設定は以下のようになっています。
![2024-01-14_17h52_38.png](/images/auth0-dashboard-login-require-login/2024-01-14_17h52_38.png)
これで Google authenticator 側の設定も完了です。
ちなみに、Google authenticator の設定が最初だった場合はそちらに先程示したリカバリーキー保存用の画面が表示されます。
では先程と同様にログアウトして、パスワードとメールアドレスを入力してログインしてみてください。
すると同様の画面が表示されますが、今度は「コードをマニュアル入力」をクリックして、Google authenticator に表示されている 6 桁の数字を入力すればログインが完了します。
以上で Google authenticator での MFA 設定が完了です。

## 余談 MFA 設定後のログインの際に表示される画面について

Google authenticator もしくは、Auth0 Guardian どちらかを用いてログインした場合、以下の画面が表示されるかもしれません。
![2024-01-14_15h24_22.png](/images/auth0-dashboard-login-require-login/2024-01-14_15h24_22.png)
これですが、もし使用している端末が適切に管理されており、第三者に使用されてしまう心配が少ないなら、登録するのもありです。
上記画面ですが、MFA の設定に現在使用している端末のロックを解除できることを追加するか尋ねています。
もし有効にする場合は「続ける」を押します。
すると、画面に表示されるパスワードの入力、指紋認証など画面ロック解除のための情報入力が求められるので、適切な情報を渡します。
これらが完了すれば、以下のような項目が追加され、Auht0 へのログインに Auth0 Guardian や Google authenticator を使用する必要がなくなります。
![2024-01-14_15h27_09.png](/images/auth0-dashboard-login-require-login/2024-01-14_15h27_09.png)
端末が安全な状況下で使用されているなら、これを適用するとログインが楽になるので個人的にはおすすめです。

## おわりに

今回は Auth0 のダッシュボードにログインに MFA が必須化されることについて取り上げました。
Auth0 は[メール検証やパスワードリセットをテナントログに残さない](https://community.auth0.com/t/action-required-password-reset-and-email-verification-links-removed-from-tenant-logs/122576)など、よりセキュリティに対して厳しくなっています。
今後より引き締めることはあっても、緩くなることはないので引き続き状況を把握して、開発体験を損なわないようにしていきたいです。
