---
title: "Device Bound Session Credentials (DBSC)の概要や分からないことをつらつらと書く"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["google","Cookie","DeviceBoundSessionCredentials","DBSC","Session"]
published: true
---

## はじめに

先日 Google のブログに以下の投稿がありました。
https://blog.chromium.org/2024/04/fighting-cookie-theft-using-device.html
詳細はこの後見ていきますが、かなり端折ると以下のツイートに凝縮されると思っています。
https://twitter.com/agektmr/status/1775315054873002206
正直この先の話はこのツイート以上の話はないのですが、一緒に新しい仕様を見ていけたらと思います。
なお、以降記述量削減のため Device Bound Session Credentials は DBSC と略称で記載いたします。
また、大変申し訳ないのですが、表題のように私自身 DBSC について理解しきれていない部分が多々あります。
なので、正確性に欠く記述やただの感想が散見されます。
その際は、コメントをいただけますと幸いです。
でははじめます。

## DBSC の目的

Cookie は現代の Web において、あなたがサイトに訪れたことを示す時によく使用さます。
さらにはログインしたことを示す情報を保持することで、ユーザーはアプリにアクセスするたびにログインせず使用できます。
このように Cookie は昨今 Web においてはなくてはならないものとなっています。
ですが、それ故に多くの攻撃にさらされる危険性をはらんでいます。
実際に以下のようにマルウェアによる Cookie の盗難を図っている事象もあります。
[https://gigazine.net/news/20211021-cookie-theft-malware-youtuber/](https://gigazine.net/news/20211021-cookie-theft-malware-youtuber/)
このマルウェアを起動してしまうと、ユーザーを識別する Cookie の値を取得でき、なりすましを可能とします。
上記のことは認証全てが終わった後発行された Cookie を盗難しているので、二要素認証にしたとしても回避できません。
他にも Cookie の有効期限を短くすることで、盗難されても悪用しにくくする方法があります。
しかし、Cookie をログインしたことを示すキーとして使用していると、有効期限が切れた際に都度ログインする必要があり、ユーザービリティが低下します。
このように Cookie 盗難によるマルウェアに対する対策は完璧でありません。
そこで現れたのが今回紹介する DBSC です。
DBSC はデバイスごと固有の鍵を使用し Cookie の発行サイクルを行います。
これによって、Cookie の盗難を行うマルウェアはローカルでの実行を余儀なくされます。※
ローカルで実行させれば検知がしやすくなり、盗難リスクを軽減できます。
以上が DBSC の目的となります。

※この点については私が分かっていない部分があります。
この後仕様についても見ていくのですが、セッションの開始と更新については鍵を使用しています。
一方で、セッションのそのものの形式については特に言及はない気がします。
そのため、現状の DBSC の仕様を実装してもセッションそのものと鍵の紐づきはないと思っています。
となると、リクエストの覗き見でもセッションの盗難はできるようなきがしており、マルウェアの実行をローカルに限定するという DBSC の目的といまいち繋がっておりません。

## DBSC のフロー

DBSC に関わる全体的なフローは以下のとおりになります。
![**[Device Bound Session Credentials explainer](https://github.com/WICG/dbsc/blob/main/README.md) より引用**](/images/about-dbsc-infomation/Untitled.png)
**[Device Bound Session Credentials explainer](https://github.com/WICG/dbsc/blob/main/README.md) より引用**
この中でセッションの登録と維持について見ていきます。
なお、この図の中にある TPM は Trusted Platform Module の略称で、データの暗号化・復号や鍵ペアの生成、ハッシュ値の計算、デジタル署名の生成・検証などの機能を有する半導体だと思ってください。
また各エンドポイントはおそらく仮のエンドポイントで、実際に使用する時は別のものになっていると思います。

### セッションの登録

セッションの登録は以下のようにサーバー 側が Sec-Session-Registration ヘッダーを付与してレスポンスを返してからスタートします。

```
HTTP/1.1 200 OK
Sec-Session-Registration: "path";challenge=:Y2hhbGxlbmdl:;es256;rs256;authorization=:YXV0aGNvZGU=:
```

path は[MDN](https://developer.mozilla.org/ja/docs/Web/HTTP/Cookies)を確認すると以下のような記載があります。

> `Path`  属性は、 `Cookie`  ヘッダーを送信するためにリクエストされた URL の中に含む必要がある URL のパスを示します。 `%x2F` ("/") の文字はディレクトリー区切り文字として解釈され、サブディレクトリーにも同様に一致します。

Cookie を使用できる URL のパスだそうです。
challenge については詳細の言及はありませんでした。
なので、推測にはなりますがこの後の流れを見る限り、ブラウザから再度サーバーへ送ることでなりすましを防止するために使用すると思っています。
es256 や rs256 は多分署名を行う際にサポートしているアルゴリズムを指定します。
authorization については、セッションを登録した時に他のフローのトークンと紐づけるためにありそうです。※
これがあることで、他のフローとの統合がしやすいみたいです。

※統合のイメージを OAuth で考えてみて、以下の感じかなとイメージしました。

1. クライアント（ブラウザ）は、OAuth の認可エンドポイントにリクエストを送信し、認可コードを取得します。
2. サーバーは、DBSC のセッション登録レスポンスの「authorization」属性に、この認可コードを含めます。
3. ブラウザはセッション登録のリクエストの際にこの認可コードを JWT の payload に含めます。
4. サーバーは、セッション開始リクエストの Authorization ヘッダーから認可コードを取り出し、これを使って OAuth のアクセストークンを取得します。
5. サーバーは、取得したアクセストークンを、新しく開始する DBSC セッションに関連付けます。

この方法で、認可コードが OAuth の認可フローと DBSC のセッション開始フローをつなぐ役割を果たします。
サーバーは、認可コードを介して取得したアクセストークンを、DBSC セッションに関連付けることができます。
あっているかはわかりませんが、上記のイメージがしっくり来たので記載しました。
サーバーからレスポンスが来たらブラウザ側は秘密鍵と公開鍵を生成します。
そして、秘密鍵を TPM に保存し、保存した秘密鍵を使用し以下のような JWT に対して署名を作成します。

```json
// Header
{
  "alg": "Signature Algorithm",
  "typ": "JWT",
}
// Payload
{
  "aud": "URL of this request",
  "jti": "nonce",
  "iat": "timestamp",
  "key": "public key",
  "authorization": "<authorization_value>", // optional, only if set in registration header
}
```

後は以下のようにエンコードした署名付き JWT を送信します。

```
POST /securesession/startsession HTTP/1.1
Host: auth.example.com
Accept: application/json
Content-Type: application/json
Content-Length: nn
Cookie: whatever_cookies_apply_to_this_request=value;
<base64-URL-encoded registration JWT>
```

その後、サーバー側で JWT の中に含まれた公開鍵とアルゴリズムを使用しつつ署名を検証します。
検証が完了すれば、サーバー側でセッションを登録することができます。
なお、セッションが登録した時のレスポンスには以下のような値が含まれます。

```json
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Set-Cookie: auth_cookie=abcdef0123; Domain=example.com; Max-Age=600; Secure; HttpOnly;
{
  "session_identifier": "<server issued identifier for the session>",
  "credentials": [{
    "name": "auth_cookie",
    "excluded_scope": [
      // the path is a prefix
      "path_a",
      "path_b"
    ]
  }],
}
```

### セッションの維持

まずセッションの有効期限が切れている状況になります。
その時、ブラウザは以下のようにセッションの再発行リクエストを送ります。

```
POST /securesession/refresh HTTP/1.1
Host: auth.example.com
Content-Type: application/json
Content-Length: nn
Cookie: whatever_cookies_apply_to_this_request=value;
Sec-Session-Id: [session ID]
```

すると、サーバー側から以下先ほど見た challenge を含む Sec-Session-Challenge ヘッダーが付与された 401 エラーを受け取ります。

```
HTTP/1.1 401
Sec-Session-Challenge: session_identifier=<session identifier>,challenge=<base64url encoded challenge>
```

エラーを受け取ったら、セッション登録の時に生成した秘密鍵を使用して署名した JWT を付与しつつリクエストします。

```
POST /securesession/refresh HTTP/1.1
Sec-Session-Response: <base64-URL-encoded JWT>
```

この時リクエストボディに JWT を含めるのではなく、Sec-Session-Response ヘッダーに JWT を付与します。
なお、JWT の payload には以下の値が含まれるのを想定していそうです。※

```json
{
  "jti": "challenge from Sec-Session-Challenge header",
  "aud": "the URL to which the Sec-Session-Response will be sent",
  "sub": "the session ID corresponding to the binding key"
}
```

検証が成功すれば、セッション登録の時と同様に cookie がセットされ、以下のようなレスポンスが返ってきます。

```json
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Set-Cookie: auth_cookie=abcdef0123; Domain=example.com; Max-Age=600; Secure; HttpOnly;
{ // optionally adjusting the session
  // id must be the same as previously
  "session_identifier": "<server issued identifier for the session>",
  "credentials": [{
    "name": "auth_cookie",
    "excluded_scope": [
      // the path is a prefix
      "path_a",
      "path_b"
    ]
  }],
}
```

これでセッションの維持が行えます。

※payload にこれら値が含めることが必須かどうかについての言及はなく、他の値も含めてよいかも特に言及はありませんでした。

### セッションの終了

先ほど図で示したフローにはセッションの明示的な終了について言及はありませんでしたが、終了方法についても言及はあったので紹介します。
セッションを終了する方法は以下の 3 つです。
① セッション維持をする時のサーバー側のレスポンスに以下の値を含める。

```
{
  "session_identifier": "<server issued identifier for the session>",
  "continue": false
 }
```

② サーバー側が Clear-Site-Data: "storage”ヘッダーを送信する
③ ユーザーがブラウザでサイトのセッションなどを削除する
①、② については具体的な動きについてはわかりませんが、少なくとも Google はこれら方法でセッションの終了を図る機能を提供するぽそうです。

## DBSC でよく分からないこと

ここでは DBSC の概要などを見てよく分からなかった部分をつらつらと書いていきます。

### Cookie の有効期限が短くするのが前提なのか？

DBSC のフローを見ると、セッションの登録や維持についてはデバイスに保存した鍵を使用しており、鍵との紐づきがないと先に進めないようになっていると考えています。
一方で、有効な Cookie を使ったリクエストについての値については特に言及がなさそうです。
発行と維持の時は保護されても Cookie そのものが奪われたら、DBSC でセッションを張った意義がなくなりそうです。
これらのことから、登録とか Cookie を発行する時は安全にできるようにしたから、Cookie の有効期限はなるべく短くしてねということになると推測しています。
ただ、上記のことが正しいかわかりませんし、何となく正しくない気もしています。

### 再発行の時にユーザー情報と紐づけはどうするのか？

セッションが無くなった時に、セッションを維持する流れについては先ほど見ました。
流れ自体に大きな違和感はないのですが、仮にセッションがユーザー情報と紐づいている場合に疑問が残ります。
再度ユーザー情報と紐づけるためのキーがないので、セッションを再度張り直すことが困難な気がします。
セッションが無くなったとしても、ユーザーを識別する必要はあると思うのですがそれについての言及はなく、どうしてよいかが不明です。
個人的には有効期限を決めつつ、有効期限がすぎてもセッション自体は残っている必要はあるのかと思っています。
セッションが残っているので、セッションの維持ができるようになります。
ただ、これだと開発者側で実装することがかなり多くなると思うので、適切かは定かじゃないです。
以上が DBSC 周りの仕様などを読んで腑に落ちていない点です。
ただ、これは DBSC の問題ではなく、そもそもセッションを私がよくわかっていないのが原因かもしれません。
なので、他力本願で恐縮ですが、ここは DBSC の話ここはセッションそれ自体の話といった指摘をコメントいただけると幸いです。

## おわりに

今回は DBSC について確認して、仕様や感じたことを見ていきました。
私が分かっていないことも多く、何を言いたいか分からない文章になってしまい申し訳ない気持ちです。
とはいえここが私の現在地だということも分かり、引き続き学んで行かないことを自覚できたのは良かったです。
最後に DBSC はセッションをどう扱うようにしたいのかを説明している記事のリンクを添付して終わりとします。
https://sizu.me/ritou/posts/emomf80fe2w6
この記事を読めば、内部の仕様は分からなくても DBSC が目指すセッションの取り扱いの方向性が見える気がします。
是非読んでみてください。
ありがとうございました。
