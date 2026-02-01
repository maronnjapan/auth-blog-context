---
title: "Gitのログ集めたり、git-secretsでキー流出を防いだりする"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Git", "AWS"]
published: true
---

# はじめに

今回は Git 操作周りの記事となっています。
それぞれ独立して書いてもよかったのですが、各内容のボリュームがそこまで多くないのでまとめて一記事にしました。
小ネタなので、気軽に読んでもらえたらなと思います。

# Git のコミットメッセージを取得するシェルスクリプトを作った

## はじめに

突然ですが、Git で行ったコミットメッセージを取得したいことはないでしょうか？
私はあります。
なので、今回は Git でコミットメッセージを取得するコマンドを紹介した後、より効率よくコミットメッセージを取得するスクリプトを紹介していきます。

## 早速結論

以下のシェルスクリプトを作成することで、任意の日付で任意のフォルダのコミットメッセージを取得することができます。

```bash
#!/bin/bash
CURRENT=$(pwd)
usage() {
    cat  1>&2 <<EOF
Usage: $(basename $0) [OPTIONS]
Options:
    -d YYYY-mm-dd       get logs after date
    -p DIR    Getting logs pass (default: .)
    -t Prompt message title
    -m Match Regex
EOF
    exit 1
}
pass='.'
date=$(date '+%Y/%m/%d')
promptTitle='以下のコミットメッセージを要約してください。'
match="^- Jwt"
while getopts 'p:d:t:m:h' opt; do
    case "${opt}" in
        d) date="${OPTARG}" ;;
        p) pass="${OPTARG}" ;;
        t) promptTitle="${OPTARG}" ;;
        m) match="${OPTARG}" ;;
        h) usage ;;
        ?) usage ;;
    esac
done
git -C $pass log --after "${date}" --pretty=format:"- %s" > $CURRENT/prompt.txt
sed -i  "1i${promptTitle}\n"  $CURRENT/prompt.txt
sed -i "/${match}/d" $CURRENT/prompt.txt
# このファイルはLinuxで使用することを想定している。
# そのため、予め「sudo apt-get install xsel」でxselをインストールしている必要がある。
cat $CURRENT/prompt.txt
cat $CURRENT/prompt.txt | xsel --clipboard --input
```

では何をしているのか上から見ていきます。
まず以下のコードですが、CURRENT 変数に現在シェルを実行しているディレクトリの値を格納しています。
その後、予期せぬオプションや h オプションを使用した時にメッセージを表示させるための関数を作成しています。

```bash
CURRENT=$(pwd)
usage() {
    cat  1>&2 <<EOF
Usage: $(basename $0) [OPTIONS]
Options:
    -d YYYY-mm-dd       get logs after date
    -p DIR    Getting logs pass (default: .)
    -t Prompt message title
    -m Match Regex
EOF
    exit 1
}
```

usage 関数についてもう少しだけ見ていきます。
cat は標準入力を標準出力に流し込むものです。
ざっくり言ってしまえば入力した内容を出力するコマンドとなります。
`<<`は後ろに指定した区切り文字が現れるまでは入力し続けることができるオプションとなります。
後ろの文字列は何でもいいのですが、慣例として EOF を使うことが多いらしいです。
<<オプションはなくて出力はできるのですが、記載せず cat コマンドによる出力をすると改行が上手くされません。
なので、自分で改行コードも付与する必要があり、コードの可読性が下がります。
そのため、今回は<<オプションを使いシェルファイル内の記載と実際の出力が同じになるようにしました。
<<オプションの区切り文字は今回「EOF」を使用しているので、Usage から始まり、Regex までが画面に出力されます。
`1>&2`は標準出力の内容を標準エラー出力させることを意味しています。
数字はファイルディスクリプタという特定の機能を示す番号となっています。
1 は標準出力を意味しており、2 は標準エラー出力を意味しています。
そして、`>&`で左側の内容を右側に出力します。
なお、ファイルディスクリプタの場合は&を使いますが、特定のファイルに書き込む場合は> test.txt といった感じで行うだけで十分です。
以上のことから、`1>&2` は表示出力させる内容をエラー出力として扱うと認識すれば良さそうです。
一応画面に表示させるだけなら、`1>&2`は無くてもいいのですが、この関数は本来シェルファイルで実行させたいものではないので、エラーとして記録が残る`1>&2`を付与しています。
`basename $0`は実行ファイルの名前を取得するコマンドで、`exit 1`で cat コマンドが完了したらその後の処理は行って欲しくないので、シェルファイルを終了するようにしています。
次に以下のコードについてです。

```bash
pass='.'
date=$(date '+%Y/%m/%d')
promptTitle='以下のコミットメッセージを要約してください。'
match="^- Jwt"
while getopts 'p:d:t:m:h' opt; do
    case "${opt}" in
        d) date="${OPTARG}" ;;
        p) pass="${OPTARG}" ;;
        t) promptTitle="${OPTARG}" ;;
        m) match="${OPTARG}" ;;
        h) usage ;;
        ?) usage ;;
    esac
done
```

これはシェルファイルを実行時に、オプションの値を取得して変数に格納しています。
変数については date のみ実行時の日付を取得するように`date '+%Y/%m/%d'`を指定しています。
while 文の処理は getopts でオプションの値を取得しつつ、一致するオプションがあるかを比較するケースを getopts のとなりに記載します。
なおオプション名の右隣に「:」を付けておくと、そのオプションは引数付きのオプションとして認識されます。
getopts で取得したオプションを opt に設定し、case 文の中で一致するオプション名があるかを比較します。
d,p,t,m に一致するオプションがあれば、引数の値が OPTARG に格納されているため対象の変数に格納しています。
h オプションや存在しないオプションを設定すると、先程定義した usage 関数を実行します。
これでどのプロジェクトのどの期間のコミットメッセージをどうやって取得するかを設定できるようになりました。
では最後に以下のコードを見ていきます。

```bash
git -C $pass log --after "${date}" --pretty=format:"- %s" > $CURRENT/prompt.txt
sed -i  "1i${promptTitle}\n"  $CURRENT/prompt.txt
sed -i "/${match}/d" $CURRENT/prompt.txt
# このファイルはLinuxで使用することを想定している。
# そのため、予め「sudo apt-get install xsel」でxselをインストールしている必要がある。
cat $CURRENT/prompt.txt
cat $CURRENT/prompt.txt | xsel --clipboard --input
```

まず git log コマンドで git のコミット内容を取得します。
その際、「log」の前に C オプションを指定することで、具体的に取得するプロジェクトのパスを指定できます。
さらに after オプションを使用することで、指定日付より後のログを取得することができます。
さらに`--pretty=format:"- %s"`を用いて、コミットメッセージのタイトルだけを取得しつつ、先頭に「- 」を付与するようにしています。
最後に取得した内容を prompt.txt ファイルに書き込んでいます。
sed コマンドはファイルの内容を追記したり、置換したりできます。
今回は先程作成した prompt.txt にタイトルを付与するための処理と、正規表現に一致する行を削除する処理を行っています。
なお、i オプションを付けないと上書きされず、prompt.txt の中身自体は変わらないので、i オプションを付与して実行しています。
最後に以下の cat コマンドで prompt.txt の中身をターミナル上に表示するのと、prompt.txt の中身をクリップボードに保存する処理を行っています。

```bash
cat $CURRENT/prompt.txt
cat $CURRENT/prompt.txt | xsel --clipboard --input
```

xcel コマンドでクリップボードに内容を保存することができます。
ただ、xcel は Ubuntu に標準搭載されていないので、別途インストールを行ってください。
これで、指定のプロジェクトのコミットメッセージを取得しつつ、クリップボードに保存することができます。
`./create-prompt.sh -d 2023/11/10 -p /var/www/nestjs`こんな感じのコマンドを実行すれば、Ctrl + v で以下のようなテキストを貼り付けることができます。

```
以下のコミットメッセージを要約してください。
- prisma確認用のリポジトリ作成
- devcontainerの変更
- README追記と自作モジュールのバージョン変更
- JwtAuthGuardのインポートし直し。
- passport-jwtを使用したガードの作成
- JWT発行処理実装
- ガードを別ファイルに切り出した
- LocalStrategyの実装とガード化完了
- passportユーザー認証用のクラスとメソッド実装
- passport検証用のプロジェクト作成
- 不要なものを削除
- passportモジュール検証用プロジェクト作成
```

## なぜコミットメッセージを集めたかったのか

最後にそもそも Git のコミットメッセージを集めたかった動機について説明します。
それは、コミットメッセージで作業の進捗をまとめることができないかと思ったためです。
そうすれば、出来上がった成果だけでなく現状の状況も言語化でき、過程も確認できそうだな妄想していました。
なので、今回コミットメッセージを集めるシェルスクリプトを作成して、まとめたものを生成 AI に投げてみました。
結果としては、ほとんどオウム返しでした。
正直期待していた結果ではなかったですが、当然としては当然です。
私自身がコミットメッセージから何を導きだしたいのか曖昧なのに、生成 AI がよしなにやってくれるわけではありません。
そのため、とりあえずはコミットメッセージを集める仕組みはできたので、今後は具体的にしたいこととそれをどのようにプロンプトへ落とし込むかを考えていきます。
もし、こんなのはどう？といったアイデアがあれば是非コメントに書いてもらえると嬉しいです！

## 参考資料

[https://scrapbox.io/nwtgck/git で cd せずに指定したディレクトリ内で git コマンド実行するには-C オプションを使える](https://scrapbox.io/nwtgck/git%E3%81%A7cd%E3%81%9B%E3%81%9A%E3%81%AB%E6%8C%87%E5%AE%9A%E3%81%97%E3%81%9F%E3%83%87%E3%82%A3%E3%83%AC%E3%82%AF%E3%83%88%E3%83%AA%E5%86%85%E3%81%A7git%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89%E5%AE%9F%E8%A1%8C%E3%81%99%E3%82%8B%E3%81%AB%E3%81%AF-C%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%92%E4%BD%BF%E3%81%88%E3%82%8B)
[https://scrapbox.io/nwtgck/git 標準だけで、log の情報を JSON にして取得する方法](https://scrapbox.io/nwtgck/git%E6%A8%99%E6%BA%96%E3%81%A0%E3%81%91%E3%81%A7%E3%80%81log%E3%81%AE%E6%83%85%E5%A0%B1%E3%82%92JSON%E3%81%AB%E3%81%97%E3%81%A6%E5%8F%96%E5%BE%97%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95)
[https://it-ojisan.tokyo/sed-d/](https://it-ojisan.tokyo/sed-d/)
[https://qiita.com/kagami_t/items/84f6ec3142a8b370d908](https://qiita.com/kagami_t/items/84f6ec3142a8b370d908)
[https://maku77.github.io/p/2fyizgw/](https://maku77.github.io/p/2fyizgw/)
[https://wa3.i-3-i.info/word14383.html](https://wa3.i-3-i.info/word14383.html)
[http://to-developer.com/blog/?p=1001](http://to-developer.com/blog/?p=1001)

# AWS の Terraform を Git で公開するときにアクセスキーの漏洩を防ぐ

## はじめに

以前 Terraform を使い AWS の設定を IaC 化しました。
せっかくコードに落とし込んだので、Git で管理をしたいという気持ちが芽生えました。
ただ、[AWS ルートアカウント流出と不正利用で 15 万円請求と請求免除まで](https://swyvern.hateblo.jp/entry/2018/05/10/230949)や[AWS が不正利用され 300 万円の請求が届いてから免除までの一部始終](https://qiita.com/AkiyoshiOkano/items/72002409e3be9215ae7e)といった記事をみると、どちらも免除はされていますが尻込みしてしまいます。
仮にミスをして、アップしてしまい同じ状況になったら冷や汗が止まらないと思います。
そこで、ここでは AWS のキーを誤って push してしまわないように[git-secrets](https://github.com/awslabs/git-secrets)を導入します。
これによって、アクセスキーをまとめた環境変数ファイルをプッシュしてしまう可能性がグッと低くなります。
なお、今回は WSL 上で git-secrets の設定を行っています。

## git-secrets のインストール

git-secrets を Linux 環境に導入するには、まず[git-secrets のリポジトリ](https://github.com/awslabs/git-secrets)を`git clone [https://github.com/awslabs/git-secrets.git](https://github.com/awslabs/git-secrets.git)`でクローンします。
クローンしたら、ディレクトリ内に入ります。
そこで、`sudo make install`を実行すると`git secretsコマンド`が使えるようになります。

## git-secrets をプロジェクトに適用する

次にインストールした git-secrets をプッシュしたいプロジェクトに適用します。
まずプッシュしたいプロジェクトに行き、`git initコマンド`で.git ディレクトリを生成します。
その後、`git secrets --install`で git-secrets を適用できるようにします。
上記が完了すれば、`git secrets --register-aws`を実行します。
では動作確認をしていきます。
.gitignore の対象ではない任意のファイルを作成します。

```bash
AWSAccessKeyId=AKIAXXXXXXXXXXXXXXXX
aws_secret_access_key=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

その後、上記ファイルをコミットしようとすると画像のエラーが発生し、コミットができなくなります。
![2023-11-26_11h29_02.png](/images/git-log-and-git-secret/2023-11-26_11h29_02.png)
これで git-secrets が有効になっていることが確認できました。
注意点としては[こちらの記事](https://qiita.com/michihito_t/items/ee1e7d73f11c6ede3f06#%E8%A9%A6%E3%81%97%E3%81%A6%E3%81%BF%E3%82%8B)にあるようにアクセスキーは 20 桁、シークレットキーは 40 桁固定なので、それ以外の値でテストとして試そうとしても上手く動きません。
なので、以下のようなファイルは特にエラーも出ずにコミットができます。

```bash
aws_secret_access_key=test
```

私はそれに気づかず 30 分くらい何で動かへんのやと四苦八苦しました。

## git-secrets がチェックしている内容を見る

ここまでで git-secrets によって、AWS のキーがコミットできないことを確認できました。
その際にアクセスキーは 20 桁、シークレットキーは 40 桁でないと、チェックされないことも確認しました。
何だか不思議ですね。
なので、この節は git-secrets のチェックしている中身を確認して、なぜアクセスキーは 20 桁、シークレットキーは 40 桁でないと上手く動かないのかを見ていきます。
プロジェクトで設定されている検査パターンを確認する際は`git secrets --list`を実行します。
すると以下のような結果が返ってきます。

```bash
secrets.providers git secrets --aws-provider
secrets.patterns (A3T[A-Z0-9]|AKIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16}
secrets.patterns ("|')?(AWS|aws|Aws)?_?(SECRET|secret|Secret)?_?(ACCESS|access|Access)?_?(KEY|key|Key)("|')?\s*(:|=>|=)\s*("|')?[A-Za-z0-9/\+=]{40}("|')?
secrets.patterns ("|')?(AWS|aws|Aws)?_?(ACCOUNT|account|Account)_?(ID|id|Id)?("|')?\s*(:|=>|=)\s*("|')?[0-9]{4}\-?[0-9]{4}\-?[0-9]{4}("|')?
secrets.allowed AKIAIOSFODNN7EXAMPLE
secrets.allowed wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

providers の部分は~/.aws/credential に保存しているアクセスキー、シークレットキー情報がコミットに現れるのを禁止するための設定となります。
こちらも重要ですが、今回は patterns の方を特に見ていきます。
patterns は一致するパターンを含むコミットを禁止するための設定となります。
設定としては正規表現を使用しており、この正規表現をみるとなぜ桁数が固定なのか分かります。
まずは下記のパターンを確認します。

```bash
(A3T[A-Z0-9]|AKIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16}
```

先頭の文字列が A3T「大文字のアルファベット一文字か 0 から 9 のいずれか一文字」、AKIA、AGPA、AIDA、AROA、AIPA、ANPA、ANVA、ASIA で始まり、後ろが大文字のアルファベットと 0 から 9 の数字の 16 桁で構成される文字列を検知します。
これは、アクセスキーの値と一致します。
よって、この正規表現はアクセスキーを禁止する正規表現となります。
次に以下の正規表現です。

```bash
("|')?(AWS|aws|Aws)?_?(SECRET|secret|Secret)?_?(ACCESS|access|Access)?_?(KEY|key|Key)("|')?\s*(:|=>|=)\s*("|')?[A-Za-z0-9/\+=]{40}("|')?
```

全部を説明するのが難しいので、大体以下のような感じのものがダメだと認識してください。

```
AWS_SECRET_ACCESS=大文字のアルファベットと数字で構成される40文字
secret_access:大文字のアルファベットと数字で構成される40文字
"Access:大文字のアルファベットと数字で構成される40文字
```

git-secrets が設定した表現を紐解くと、桁数固定だったのが見えてきました。
patterns の正規表現的にアクセスキーは 20 桁、シークレットキーは 40 桁でないとダメだったんですね。
残り一つの正規表現は今回使用していないので、省略します。
最後に secrets.allowed について軽く触れていきます。
secrets.allowed は patterns で検知させない例外を設定します。
今回 allowed に存在する値は AWS のテストキーとなります。
なので、仮にプッシュされても問題がないのでエラーを発生させないために設定されています。
以上が`git secrets --register-aws`で登録したチェックの内容でした。
今回は git-secrets がテンプレートとして用意しているものを使用したので、チェック内容が自動的に反映されていましたが、自前でチェック項目を使用したい場合は[—add オプション](https://github.com/awslabs/git-secrets#options-for---add)を使うと可能となります。
また、AWS 周りのチェックは全てのプロジェクトで行うようにしたい場合は` git secrets --register-aws``--global `と global オプションを付けて実行すれば可能となります。
その他にも多くのオプションがあるので、[GitHub](https://github.com/awslabs/git-secrets)を是非参照してみてください。

## 参考資料

[Ubuntu20.04 に git-secrets をインストールする](https://qiita.com/kannkyo/items/465be766b5af0bc89749)
[git-secrets を git init や git clone した時に自動で有効にして、AWS アクセスキーの漏洩を防ぐ](https://qiita.com/michihito_t/items/ee1e7d73f11c6ede3f06)
[awslabs/git-secrets](https://github.com/awslabs/git-secrets)

# おわりに

今回は Git のログを一発で取得するスクリプトを作成したり、AWS のキーを流出させないための git-secrets について記載しました。
どちらも普段の作業を少し楽にしてくれるのと、奥が深そうでした。
快適な Git ライフの一助となれば幸いです。
ここまで読んでいただきありがとうございました
