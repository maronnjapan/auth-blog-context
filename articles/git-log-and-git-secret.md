---
title: "git-secretsでAWSキー流出を防ぐ方法"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Git", "AWS"]
published: true
targetCategories: ["security"]
---

## はじめに
ちょっとした検証用で作ったファイルはgitignoreに書き忘れ、そのままgit管理されてしまうケースがあります。
通常は問題ないのですが、シークレットが記載されているファイルだとそうは行きません。
そこで今回はgit-secretを用いて、AWSキーを間違えてプッシュするのを防ぐ方法を見ていきます。

## AWS の Terraform を Git で公開するときにアクセスキーの漏洩を防ぐ

以前 Terraform を使い AWS の設定を IaC 化しました。
せっかくコードに落とし込んだので、Git で管理をしたいという気持ちが芽生えました。
ただ、[AWS ルートアカウント流出と不正利用で 15 万円請求と請求免除まで](https://swyvern.hateblo.jp/entry/2018/05/10/230949)や[AWS が不正利用され 300 万円の請求が届いてから免除までの一部始終](https://qiita.com/AkiyoshiOkano/items/72002409e3be9215ae7e)といった記事をみると、どちらも免除はされていますが尻込みしてしまいます。
仮にミスをして、アップしてしまい同じ状況になったら冷や汗が止まらないと思います。
そこで、ここでは AWS のキーを誤って push してしまわないように[git-secrets](https://github.com/awslabs/git-secrets)を導入します。
これによって、アクセスキーをまとめた環境変数ファイルをプッシュしてしまう可能性がグッと低くなります。
なお、今回は WSL 上で git-secrets の設定を行っています。

### git-secrets のインストール

git-secrets を Linux 環境に導入するには、まず[git-secrets のリポジトリ](https://github.com/awslabs/git-secrets)を`git clone [https://github.com/awslabs/git-secrets.git](https://github.com/awslabs/git-secrets.git)`でクローンします。
クローンしたら、ディレクトリ内に入ります。
そこで、`sudo make install`を実行すると`git secretsコマンド`が使えるようになります。

### git-secrets をプロジェクトに適用する

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

### git-secrets がチェックしている内容を見る

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

### 参考資料

[Ubuntu20.04 に git-secrets をインストールする](https://qiita.com/kannkyo/items/465be766b5af0bc89749)
[git-secrets を git init や git clone した時に自動で有効にして、AWS アクセスキーの漏洩を防ぐ](https://qiita.com/michihito_t/items/ee1e7d73f11c6ede3f06)
[awslabs/git-secrets](https://github.com/awslabs/git-secrets)

## おわりに

今回は Git のログを一発で取得するスクリプトを作成したり、AWS のキーを流出させないための git-secrets について記載しました。
どちらも普段の作業を少し楽にしてくれるのと、奥が深そうでした。
快適な Git ライフの一助となれば幸いです。
ここまで読んでいただきありがとうございました
