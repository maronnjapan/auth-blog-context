# Blog Idea: npm Trusted Publisherでnpm tokenなしpublishを試す

## article-key

npm-trusted-publisher-tokenless-publish

---

## 仮タイトル

npm Trusted Publisherでnpm tokenなしpublishする  
初回公開からGitHub Actions連携まで

---

## 記事の主題

npmパッケージをGitHub Actionsからpublishするときに、
GitHub Secretsへ長期的なnpm access tokenを置かず、
Trusted Publisherを使って短命トークンでpublishできる状態を作る。

この記事では、単なる概念説明ではなく、
初回publishからTrusted Publisher設定、
GitHub Actionsによるpublishまでを実装ベースで整理する。

---

## 記事のゴール

読者がこの記事を読み終えたときに、以下の状態になれることを目指す。

- npmパッケージを初回publishできる
- npm側でTrusted Publisherを設定できる
- GitHub Actionsからnpm tokenなしでpublishできる
- npm tokenをGitHub Secretsに置かないpublishフローを理解できる
- 手動操作が必要な部分も、迷わず進められる

---

## 想定読者

- npmパッケージを公開している、またはこれから公開したい人
- GitHub Actionsでnpm publishを自動化したい人
- GitHub Secretsにnpm tokenを置き続けることに不安がある人
- Trusted Publisherという言葉は知っているが、具体的な設定方法がわからない人
- 公式ドキュメントだけでは導入の流れをイメージしづらい人
- 実際に動くサンプルを見ながら試したい人

---

## 記事で伝えたい中心メッセージ

npmパッケージのpublishは、
静的なnpm access tokenをCIに置き続けるのではなく、
Trusted Publisherを使って短命トークンで安全に行える。

Trusted Publisherを使えば、
GitHub Actionsにnpm tokenをSecretとして保存しなくてもpublishできる。

ただし、導入時には初回publishやnpm画面での設定が必要になるため、
この記事ではそこを迷わず進められるように整理する。

---

## 記事の切り口

Trusted Publisher自体は便利だが、
実際に導入しようとすると以下のポイントで少し迷いやすい。

- 初回publishはどうやるのか
- Trusted Publisherはnpmのどこで設定するのか
- GitHub Actions側では何を書けばいいのか
- npm tokenを本当にSecretに置かなくてよいのか
- 手動で設定する値は何なのか

この記事では、この導入時の迷いやすさを解消するために、
完全自動化ではなく「半自動ガイド」として整理する。

---

## サブアイディア

### idea-001: 静的npm tokenをCIに置くリスク

#### 目的

GitHub ActionsなどのCI/CDに長期的なnpm access tokenを置き続けるリスクを説明する。

#### 記事に入れる理由

Trusted Publisherを使う動機を明確にするため。

単に「新しい仕組みだから使う」のではなく、
長期tokenをCIに置く運用にはリスクがあるため、
短命トークンを使うpublishフローに移行したい、という流れを作る。

#### 扱う内容

- npm access tokenとは何か
- CI/CD Secretに長期tokenを置くリスク
- token漏洩時に起こり得ること
- 短命トークンにしたい理由
- Trusted Publisherへつなげる導入

#### 記事内での扱い

導入・問題提起パート。

---

### idea-002: Trusted Publisherとは何か

#### 目的

Trusted Publisherが何を解決する仕組みなのかを説明する。

#### 記事に入れる理由

読者が手順だけをなぞるのではなく、
なぜnpm tokenなしでpublishできるのかを理解できるようにするため。

#### 扱う内容

- Trusted Publisherの概要
- GitHub Actionsとnpmの信頼関係
- CI/CD実行時に短命トークンを使う考え方
- 静的npm token方式との違い
- npm tokenをGitHub Secretsに置かなくてよい理由

#### 記事内での扱い

基礎説明パート。

---

### idea-003: 初回publishを行う

#### 目的

Trusted Publisher設定前に必要な初回publishの流れを整理する。

#### 記事に入れる理由

Trusted Publisherを使う前提として、
npmパッケージが一度publishされている必要があるため。

ここで読者がつまずくと、
Trusted Publisherの設定までたどり着けない。

#### 扱う内容

- package.jsonの確認
- package nameの確認
- versionの確認
- npm login
- 初回npm publish
- npmパッケージページの確認

#### 記事内での扱い

実装手順パート。

---

### idea-004: 初回publishガイドスクリプト

#### 目的

初回publish時に必要な確認や操作を、
CLI上で順番に案内する。

#### 記事に入れる理由

読者に「何を確認してからpublishすればよいか」を明確に伝えるため。

単にコマンドだけを書くのではなく、
実際に手を動かしやすい形にする。

#### 扱う内容

- package nameの表示
- versionの表示
- npm login済みかの確認
- publish前チェック
- publishコマンドの案内
- publish後のnpmパッケージページ表示
- 次のTrusted Publisher設定への誘導

#### 記事内での扱い

実践パート。

---

### idea-005: Trusted Publisher設定を半自動ガイド化する

#### 目的

npmのWeb画面で行うTrusted Publisher設定を、
読者が迷わず進められるようにする。

#### 記事に入れる理由

Trusted Publisherの設定はnpm画面上で行う必要があり、
完全にCLIだけで完結しない。

そのため、手動操作は残しつつも、
開くURL、入力する値、確認すべき状態を明確にする。

#### 扱う内容

- npmパッケージ設定ページを開く
- Trusted Publisher設定画面への案内
- GitHub owner / repository の表示
- workflow file名の表示
- npm画面で入力する値の表示
- 設定完了後の確認ポイント

#### 記事内での扱い

記事の独自性になる実践パート。

---

### idea-006: AIにも渡しやすい半自動ガイドにする

#### 目的

半自動ガイドを、人間が読むだけでなく、
Claude in Chrome や ChatGPT Agent などのブラウザ操作支援AIにも渡しやすい形にする。

#### 記事に入れる理由

npm画面の操作をPlaywrightなどで完全自動化するのは重い。

一方で、最近はブラウザ操作を支援できるAIに、
手順や入力値を渡して画面操作を手伝ってもらう選択肢がある。

そのため、この記事のガイドは、
人間にもAIにも伝わりやすい構造にしておく。

#### 扱う内容

- 開くURLを明示する
- 入力する値を明示する
- クリックする場所を説明する
- 設定後に確認する状態を明示する
- publishや権限変更など重要操作前には人間が確認する
- AIに完全委任するのではなく、画面遷移や入力補助を任せる位置づけにする

#### 記事内での扱い

半自動ガイドの設計思想として入れる。

---

### idea-007: GitHub Actionsからpublishする

#### 目的

Trusted Publisher設定後に、
GitHub Actionsからnpm publishできることを確認する。

#### 記事に入れる理由

この記事の最終ゴールが、
GitHub Actionsにnpm tokenを置かずにpublishすることだから。

#### 扱う内容

- publish用workflowファイル
- `permissions` の設定
- `npm publish` の実行
- npm tokenをSecretに置かないこと
- Actions実行結果の確認
- npmパッケージの新バージョン確認

#### 記事内での扱い

実装の最終パート。

---

## 今回の記事で扱わないこと

この記事では、以下は扱わない。

- patch / minor / major の自動判定
- changelog生成
- GitHub Release連携
- 本格的なrelease branch運用
- Playwrightによるnpm画面操作の完全自動化
- npmパッケージ自体の高度な設計

これらは、Trusted Publisherによるpublishフローが整った後の発展として扱う。

---

## 記事の構成案

# npm Trusted Publisherでnpm tokenなしpublishする

## はじめに

- npmパッケージをGitHub Actionsからpublishしたい
- ただし、npm tokenをGitHub Secretsに置き続けるのは避けたい
- Trusted Publisherを使うとnpm tokenなしでpublishできる
- この記事では、初回publishからGitHub Actions publishまでを整理する

## 静的npm tokenをCIに置くリスク

- npm access tokenとは
- CI/CDに長期tokenを置くリスク
- token漏洩時の影響
- 短命トークンを使いたい理由

## Trusted Publisherとは

- Trusted Publisherの概要
- GitHub Actionsとの関係
- npm tokenなしでpublishできる理由
- 静的token方式との違い

## 今回作るもの

- サンプルnpmパッケージ
- 初回publishガイド
- Trusted Publisher設定ガイド
- GitHub Actions publish workflow

## 初回publishする

- package.jsonを確認する
- npmにログインする
- 初回publishを実行する
- npmパッケージページを確認する

## Trusted Publisherを設定する

- npmのパッケージ設定画面を開く
- Trusted Publisher設定画面へ移動する
- GitHub repositoryを設定する
- workflow fileを設定する
- 設定完了を確認する

## 完全自動化ではなく、AIにも渡しやすい半自動ガイドにする

- npm画面操作は完全自動化しない
- CLIで開くURLと入力値を表示する
- 人間が見ても進められる形にする
- Claude in Chrome や ChatGPT Agent にも渡しやすい形にする
- 重要操作前は人間が確認する

## GitHub Actionsからpublishする

- workflowファイルを確認する
- publishを実行する
- npm tokenなしでpublishできることを確認する
- npmパッケージの更新を確認する

## まとめ

- npm tokenをCIに置き続ける運用にはリスクがある
- Trusted Publisherを使えばnpm tokenなしでpublishできる
- 初回publishとnpm画面設定は少し手間がある
- そこを半自動ガイド化すると導入しやすくなる
- 人間にもAIにも渡しやすい形にしておくと、画面操作の負担を減らせる

---

## 記事の完成イメージ

この記事は、単なるTrusted Publisherの設定手順ではなく、
「npm tokenなしpublishを導入するときに迷いやすい部分」を実践ベースで整理する記事にする。

特に、初回publishとnpm画面でのTrusted Publisher設定は、
完全自動化しようとすると重くなる。

そのため、この記事では、
CLIで必要な情報を表示し、
npm画面で入力すべき値を明確にし、
人間にもAIブラウザにも渡しやすい半自動ガイドとして整える。

最終的に、読者がGitHub Actionsにnpm tokenを置かず、
Trusted Publisherで安全にnpm publishできる状態を目指す。
