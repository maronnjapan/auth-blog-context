---
title: "DBSCを理解する個人的最短経路"
emoji: "😺"
type: "tech"
topics: ["DBSC", "Cookie"]
published: true
---
(今回は短めです)
現在Cookieの盗難対策として、[Device Bound Session Credentials](https://w3c.github.io/webappsec-dbsc/)(以降はDBSC)という仕様が提言されています。
これは、デバイスに保存した秘密鍵を使い、セッションの発行・更新をデバイスに紐づけることで盗難リスクを低減させる仕様となります。
:::message
上記の説明はかなり端折った説明なので、雰囲気だけ感じてください
:::
このDBSCですが、まだ正式な採用はされていませんが、以下のように情報自体はぽつぽつと存在します。(こそっと自分の記事も入れさせていただきました...)
https://w3c.github.io/webappsec-dbsc/
https://developer.chrome.com/blog/dbsc-origin-trial
https://blog.jxck.io/entries/2025-01-16/device-bound-session-credentials.html
https://developer.chrome.com/docs/web-platform/device-bound-session-credentials
https://blog.chromium.org/2024/04/fighting-cookie-theft-using-device.html
https://krishnag.ceo/blog/what-are-device-bound-session-credentials-dbsc/
https://github.com/w3c/webappsec-dbsc
https://zenn.dev/maronn/articles/program-dbsc-app
https://gigazine.net/news/20240403-chrome-device-bound-session-credential/
このように、多くはないですがすでにDBSCへの対応の準備はできるくらいの情報はあります。
なので、ここでは個人的DBSCに馴染むための最短経路を見ていこうと思います。
DBSC触れたいけど、何からすればいいの？となっている方の役に立てば幸いです。
## DBSCに慣れるための最短経路
最短経路は以下の通りです。
①以下の記事で概要をざっくり掴む
https://developer.chrome.com/docs/web-platform/device-bound-session-credentials
上記記事はDBSCのことがとてもコンパクトにまとまっていて、何をしているかのざっくりとした把握に向いています。
ただし、この記事は読むくらいに留め、この記事をもとに実装はまだしません。
というか、この記事のみで実装できる方はすでにDBSCをある程度理解している人なので、そもそも今回私が書いている記事を読む必要がないです。
②実際に動かしてみる
私が書いた以下の記事をもとに、実際にアプリを動かしてみてください。
https://zenn.dev/maronn/articles/user-login-and-dbsc
（`chrome://flags/`の設定は、[こちら](https://zenn.dev/maronn/articles/dbsc-when-browser-restart#%E5%85%88%E3%81%AB%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%81%AE%E5%86%8D%E8%B5%B7%E5%8B%95%E6%99%82%E3%81%AB%E3%82%82dbsc%E3%82%92%E5%8B%95%E3%81%8B%E3%81%99%E8%A8%AD%E5%AE%9A%E3%82%92%E5%85%B1%E6%9C%89)が正しいので、ご注意ください。）
なお、もっとコードがシンプルなものが良い場合は以下のリポジトリを参照すると良いです。
https://github.com/drubery/dbsc-test-server
こちらは、[DBSCリポジトリ](https://github.com/w3c/webappsec-dbsc)でのコラボレーターとなっている方が実装されています。
以上のいずれかで、とりあえずDBSCの登録やCookieの有効期限が切れた時に、更新処理が走ることを体感いただければと思います。
③DBSCリポジトリのREADMEを読む
なんとなくDBSCの輪郭が掴めてきたら、今度は、[DBSCリポジトリ](https://github.com/w3c/webappsec-dbsc)のREADMEを読んでみます。
このREADMEを読むことで、DBSCの輪郭はかなり掴めます。
特にREADMEの[High Level Overview](https://github.com/w3c/webappsec-dbsc?tab=readme-ov-file#high-level-overview)をしっかり読むことを勧めます。
ここが理解できれば、DBSCの解像度がかなり上がると個人的には思います。
以上が個人的なDBSCを理解するための最短経路です。
良ければ、この手順で読んでみてDBSCの素晴らしさを皆様も是非感じてください。
## 仕様書は読まなくていいの？
最短経路について書きましたが、その中に[DBSCの仕様書](https://w3c.github.io/webappsec-dbsc/)の言及がありませんでした。
これは忘れているのではなく、意図的に書いていません。
理由は、仕様がDBSCを使う側よりDBSCのAPIを実装する側に向けた記載が多いためです。
DBSCは大きく分けて、DBSCを利用するアプリのサーバー側と、DBSCを提供するブラウザ側があります。（やや不正確な記載ですがご容赦ください）
仕様書はその中でも、DBSCを提供する際の記載が多く含まれています。
DBSCは利用する側に負担を負わせないことが前提としてあります。（[参照](https://w3c.github.io/webappsec-dbsc/#:~:text=The%20API%20takes%20special%20care%20to%20integrate%20easily%20with%20existing%20server%2Dside%20auth%20stacks%2C%20providing%20an%20incremental%20path%20to%20such%20protections%20that%20does%20not%20require%20rewriting%20large%20parts%20of%20the%20web%20software%20stack.)）
そのため、DBSCを提供する側で色々と考慮することが多く、その注意点などを仕様書に記載している形となります。
試しに、仕様書を読むと以下のような記載があります。
> After computing cookies in step 8.21, run [§ 8.5 Identify session needing refresh](https://w3c.github.io/webappsec-dbsc/#algo-identify-session-needing-refresh). If the resulting session is non-null:

Cookieを計算した後、DBSCにおけるリフレッシュが必要かを判断しろとあります。
これを真面目に適用させるなら、ブラウザの設定を触る必要がありますね。
すなわり、DBSCを提供する側しかできない作業となります。
紹介したのは一部でしたが、アルゴリズムを説明している8節なども、DBSCの提供側で行うロジックが多いです。
このように、仕様書はDBSCを使う我々というよりは、DBSCの提供側が関係する記載が多くあります。
もちろん仕様書を読むことが時間の無駄だとは全く思いませんが、利用する側としてはREADMEを理解すれば事足りるかなと思っています。
READMEも結構詳しく書いてくれていますしね。
以上から、仕様書は一旦は読まなくても良いかなと思います。
それよりもDBSCの良さを体験して欲しいので、一旦READMEを理解することに注力いただければ幸いです。
