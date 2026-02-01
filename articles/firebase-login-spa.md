---
title: "Firebaseで始めるソーシャル認証システム"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Firebase", "NestJS"]
published: true
---

# 目次

- はじめに
- Firebase とは
- Firebase Authenticate とは
- なぜ IDaaS を使うのか
- Firebase Auth を使用したアプリケーション
  - 環境構築
  - ソーシャルログイン
  - ログアウト・退会
  - (補)API を画面で確認する
- Firebase は Auth0 の代わりになり得るか？
- おわりに

# はじめに

認証機能を持っている SPA を作りたい！
そんな思いと Firebase が気になっている思いがありました。
そこで今回は Firebase の認証機能を使ってソーシャルログイン機能がある SPA を作成しました。
そして、最後に私がよく使っている Auth0 の代わりに Firebase を使うことができるかも記載しています。
Firebase 部分はそこまで難しく無かったので、気になる方はぜひチェックしてください。

# Firebase とは

Firebase とは 2011 年に Firebase 社がリリースした BaaS(Backend as a Service)です。
その後、Firebase 社は 2014 年に Google に買収されており、現在は Google Cloud の一機能として組み込まれています。
BaaS とは私たちが行っているアプリケーション開発において、データベースの構築やデプロイなどバックエンド側で考えるべき内容を肩代わりしてくれるサービスです。
画像のようにバックエンド側の準備は、アプリケーション開発でかなりの作業を要します。
![[BaaSとは？|サービスとしてのバックエンド対サーバーレス](https://www.cloudflare.com/ja-jp/learning/serverless/glossary/backend-as-a-service-baas/)より引用](/images/firebase-login-spa/2023-10-02_08h44_28.png)
[BaaS とは？|サービスとしてのバックエンド対サーバーレス](https://www.cloudflare.com/ja-jp/learning/serverless/glossary/backend-as-a-service-baas/)より引用
そのバックエンド側を BaaS に任せてしまうことで、かなりの工数を削減することができます。
Firebase も BaaS なので、上記で紹介したような機能は揃えております。
その中で今回は Firebase Authentication を中心に使用していきます。

# Firebase Authentication とは

[Firebase Authentication](https://firebase.google.com/docs/auth?hl=ja)とは Fireabse に搭載されている認証を中心とした IDaaS(Identity as a Service)です。
バックエンド サービスに加えて、SDK、アプリでのユーザー認証に使用できる UI ライブラリが用意されています。
認証方法としては、パスワード、電話番号、Google や Github などのソーシャル認証機能を備えております。
基本的な認証機能は上記のように備えていますが、[Identity Platform](https://cloud.google.com/identity-platform?hl=ja)という Google 提供のサービスと連携すると認証周りの詳細なログ収集や、多要素認証や、ユーザー認証時の関数実行など多様な機能を搭載することができます。
なお、[Identity Platform](https://cloud.google.com/identity-platform?hl=ja)は今回は解説しないので、ご了承ください。
料金は基本的には無料で、一日に送信できる SMS が 10 件のみという制限はあります。
ただ、[Identity Platform](https://cloud.google.com/identity-platform?hl=ja)と接続した場合、無料プランの時一日のアクティブユーザーは 3000 人までとなります。
その他にも料金体系はありますが、この記事では[Identity Platform](https://cloud.google.com/identity-platform?hl=ja)とは連携しない Firebase しか使わないので、料金が発生する事態はないと思います。
そのため、詳しい料金については[ドキュメント ①](https://firebase.google.com/docs/projects/billing/firebase-pricing-plans?hl=ja)、[ドキュメント ②](https://firebase.google.com/pricing?hl=ja)に丸投げします。

# なぜ IDaaS を使うのか

Firebase を使った実装へ入る前に、IDaaS を使うメリットについて解説します。
認証機能を搭載するだけなら、自前でメールアドレスとパスワードの入力フォームを作りデータベースと一致するかを見るだけで良さそうです。
そんな単純なことなのに、なぜわざわざ IDaaS を使用するのかと考えてしまいます。
ですが、IDaaS を使うことは明確なメリットがあります。
この章ではそれらを見ていこうと思います。

## パスワード認証の課題

IDaaS のメリットを説明するまえに、パスワード認証の課題について見ていきます。
これによって、IDaaS の利点を強調することができます。
認証方法として当然とされているパスワード認証ですが、以下の 4 つの課題を持っています。
① パスワードの失念
② パスワードの使い回し
③ 推測されやすいパスワードの使用
④ フィッシングアプリへのパスワード入力
それぞれ見ていきましょう。

### ① パスワードの失念

ログインが必要なアプリにはそれぞれ異なるパスワードを使用するというのは皆さんご存知だとは思います。
とはいえ、使用するアプリが数個ならまだしても数十個が当たり前の昨今では異なるパスワードを忘れてしまうことが多々あります。
なので、パスワード認証を搭載するには別途パスワードを忘れた時の再設定機能を搭載する必要があります。
とはいえ、パスワード再設定を行うにはメールサーバーを用意して、メール内のリンクの有効期限を設定するなどの多くの作業が必要となります。
この手間は結構馬鹿になりません。

### ②**パスワードの使い回し**

先程話したように、各サイトで異なるパスワードを使用するとその数は膨大になり、覚えていられません。
そこで、アプリケーションごとではなく共通のパスワードを使用してログインを行うことが多々あります。
ですが、パスワードの使い回しは「パスワードリスト攻撃」という攻撃の対象となります。
この攻撃は流出したユーザー識別子とパスワードを用いて、Web サイトで不正なログインを試みる方法です。
パスワードを使いまわすことで、どこかのサイトのログインができてしまい、情報が抜き取られる可能性が高まります。

### ③**推測されやすいパスワードの使用**

こちらも適切なパスワードではないと知っていても、ランダムなパスワードを全て覚えていられないので、覚えるために使用する場合があります。
ただ、推測されやすいパスワードを使用すると今度は「[パスワードスプレー攻撃](https://cybersecurity-jp.com/column/30629)」で情報を抜き取られる可能性が上がってしまいます。
パスワードスプレー攻撃とは、同じパスワードを使用して複数のユーザー情報を入力してサイトにログインしようとする攻撃です。
推測されやすいパスワードを使用していると、攻撃用に使用するパスワードと一致する可能性が高くなりパスワードスプレー攻撃の被害を受けやすくなります。

### ④**フィッシングアプリへのパスワード入力**

[フィッシングサイト](https://www.nttpc.co.jp/column/security/phishing-site.html)とは、正規のサイトと見た目が全く一緒の偽物サイトです。
主に個人情報を入力させ、入力した情報を抜き取るために使用されます。
見た目が全く一緒なので、ユーザーはついパスワードなどを入力してログインを行ってしまい、情報を抜き取られてしまいます。
実際 IPA が発表する[情報セキュリティ 10 大脅威 2023](https://www.ipa.go.jp/security/10threats/10threats2023.html)では、昨年から引き続きフィッシングサイトによる個人情報の詐取が 1 位となっています。
![2023-11-03_12h27_30.png](/images/firebase-login-spa/2023-11-03_12h27_30.png)
これだけ主要な攻撃のため、ユーザー側のリテラシーを求め、引き続きパスワード認証を使用するというのは限界があります。
以上パスワード認証による課題となります。

### IDaaS はパスワード認証をどう解消するのか

では、IDaaS を使うことで、これらの課題はどのように解消されるのでしょうか。
まずは「① パスワードの失念」についてです。
IDaaS を用いることで、一つのアカウントで様々なアプリにログインすることを可能にします。
そのため、パスワードを作成しすぎてパスワードを忘れてしまうということが少なくなります。
「② パスワードの使い回し」はそもそもアカウントが一つになるので、使いまわすことが無くなります。
ただ、アカウントが一つだけになると一回情報が抜き取られてしまうと他のアプリでもログインできるようになってしまう問題は残ってしまいます。
なので、パスワードを推測されにくいものとすることで不正なログインを防ぎます。
アカウントを一個に集中させ、そのアカウントのセキュリティを強固にすることで、「② パスワードの使い回し」と「③ 推測されやすいパスワードの使用」を防ぎます。
これによって、安全性を保つことができます。
最後の「④ フィッシングアプリへのパスワード入力」については IDaaS を使えば解消できるわけではないですが、「④ フィッシングアプリへのパスワード入力」の対応をするには FIDO 認証などを導入する必要があります。
自前でも導入することは不可能ではないですが、かなりの手間を要します。
一方で、IDaaS はその機能を持っている可能性が高く、さらいに自前で導入するよりも簡単にできます。
実際 Auth0 には PassKey という FIDO 認証を用いたログイン機能があり、導入も非常に簡単となっています。
よって、IDaaS で管理することによってフィッシングサイトへの対応も簡単になる可能性が高いです。
以上のことから、IDaaS を用いたアプリを提供することでユーザーは安全にかつ快適にアプリを使用できるようになります。

# Firebase Auth を使用したアプリケーション

ここでは Firsuebase が提供する Firebase Auth SDK と Firebase Admin SDK を使用してログインページを作成します。
具体的には以下の機能を実装します。

- ソーシャルログイン
- ログアウト
- メールリンクログイン
- 退会

そして、これら機能を実装するためのアプリの構成は以下の通りです。
![monorepo-firebase.drawio.png](/images/firebase-login-spa/monorepo-firebase.drawio.png)
フロント側は React を使用し、バックエンド側は NestJs を使用します。
フロント側とバックエンド側のやりとりは ts-rest による REST API で行います。
そして、フロント・バックエンド側で認証に関わる部分の機能はそれぞれ Firebase が提供している API を使用して、Firebase とやり取りします。
以上が今回作成するアプリケーションとなります。
それではまず、環境構築から始めます。

## 環境構築

### Firebase の準備

Firebase の SDK を使用するには Firebase で必要な情報を取得する必要があります。
そのために、まず[Firebase のコンソール](https://console.firebase.google.com/u/0/?hl=ja)にアクセスします。
プロジェクトを作成する要素があるので、そちらをクリックします。
するとプロジェクト名を設定する画面と、Google アナリティクスの設定を行う画面が表示されます。
今回は Google アナリティクスは使わないので、設定せず作成します。
プロジェクトを作成したら、画像赤枠内の「</>」ボタンをクリックしてウェブアプリケーションを登録します。
![2023-10-04_11h58_59.png](/images/firebase-login-spa/2023-10-04_11h58_59.png)
まず名前を決めると以下のように Firebase を導入するための画面が表示されます。
![2023-10-04_12h03_14.png](/images/firebase-login-spa/2023-10-04_12h03_14.png)
この値はフロント側で使用するので、控えておきます。
**Firebase Hosting はいるか確認すること**
アプリケーションの設定を完了したら、コンソール左上にある「プロジェクト概要」の隣の歯車をクリックします。
そして、表示される「プロジェクトの設定」をクリックします。
![2023-10-04_12h11_05.png](/images/firebase-login-spa/2023-10-04_12h11_05.png)
プロジェクトの設定画面の「サービス アカウント」タブを選択すると下記のような画面が出てきます。
「Admin SDK 構成スニペット」を Node.js にして、「新しい秘密鍵を生成」ボタンをクリックします。
![2023-10-04_12h15_29.png](/images/firebase-login-spa/2023-10-04_12h15_29.png)
すると、秘密鍵の情報が格納されている JSON ファイルがダウンロードされます。
この JSON ファイルはバックエンド側で使用するので、同様に控えておきます。
これで Firebase 側の準備は完了です。
とても簡単ですね。
Firebase を使うための設定は以上で概ね完了なのですが、この後実装するソーシャルログインとメールリンクログインを行うための設定を最後に行います。
サイドメニューから「Authentication」を選択し、表示される画面の「Sign-in method」タブを選択します。
すると、以下のような画面が表示されます。
![2023-11-14_12h22_59.png](/images/firebase-login-spa/2023-11-14_12h22_59.png)
今回は Google によるログインとメールリンクログインを行いたので、「メール/パスワード」と「Google」を有効にします。
Google の方は画面に従って入力し、「メール/パスワード」はメールリンクの方を有効にしてください。

### プロジェクト全体の準備 Part1

まず devcontainer で Node.js が使用できる環境を構築します。
devcontainer の構築方法は[こちら](https://note.com/shift_tech/n/nf9c647e5264c)など複数記事があるので、ここでは記載しません。
devcontainer ができたら、任意のフォルダ(今回は monorepo-firebase)を作成して、内部で`touch package.json`
を実行します。
作成した package.json へ以下の記載を行います。

```json
{
  "name": "@monorepo-firebase",
  "description": "モノレポのFirebaseプロジェクトです",
  "private": true
}
```

そしたら、`npm init -w packages/common -w packages/backend -w packages/frontend`を実行して、バックエンド・フロントエンド・ルータープロジェクトをそれぞれ作成します。
また、`npm install typescript`を実行して、各プロジェクトでいちいち typescript をインストールしなくても使えるようにしておきます。
一旦プロジェクトの準備は完了です。

### API 定義書の準備

ここでは、API の仕様を決めるための設定を行っていきます。
packages/ts-router フォルダ配下の package.json を削除します。
packages/ts-router フォルダ配下に行き、`npm install @ts-rest/core zod`を実行して、定義書を作るために使う ts-rest のモジュールと型バリエーション用の Zod をインストールします。
インストールが完了したら、index.ts と contract.ts ファイルを作成します。
さらに、types フォルダを作成してその中に respose-type.ts と body-type.ts を作成します。
各ファイルは以下の通りに記載します。

```tsx
//contract.ts
import { initContract } from "@ts-rest/core";
import { z } from "zod";
import { Login } from "./types/body-type";
import { UserInfo } from "./types/response-type";
const c = initContract();
export const tsRestRoute = c.router({
  hello: {
    method: "GET",
    path: "/api",
    responses: {
      200: z.object({
        name: z.string(),
      }),
    },
  },
  login: {
    method: "POST",
    path: "/api/login",
    responses: {
      201: z.undefined(),
    },
    body: Login,
    contentType: "application/json",
  },
  logout: {
    method: "POST",
    path: "/api/logout",
    responses: {
      201: z.null(),
    },
    body: null,
  },
  getUserInfo: {
    method: "GET",
    path: "/api/users/me",
    responses: {
      200: UserInfo,
    },
  },
  deleteUser: {
    method: "DELETE",
    path: "/api/users/me",
    responses: {
      204: null,
    },
    body: null,
  },
});
```

この contract.ts が仕様の要となります。
定義のプロパティ名を決め、値にメソッドの種類や許容するリクエストの値や返す値を定義します。
これをバックエンド、フロントエンドに読み込ませるといい感じに補完が効きつつ API の定義や呼び出しをしてくれます

```tsx
//body-type.ts
import { z } from "zod";
export const Login = z.object({
  idToken: z.string(),
});
```

```tsx
//response-type.ts
import { z } from "zod";
export const UserInfo = z.object({
  email: z.string(),
  name: z.string(),
});
```

上二つはリクエストやレスポンスの値を限定しています。
Zod を使うことで型定義ではないが、まるで型定義のような変数を設定できます。

```tsx
//index.ts
export * from "./contract";
```

各ファイルの実装が完了したら、tsconfig.json ファイルの compilerOptions にある outDir プロパティの値を`./dist` にします。
また、`"declaration": true`も設定します。
これで API 定義書の準備は完了です。

### バックエンド側の準備 Part1

まず、設定の重複エラーを避けるために`rm packages/backend/package.json` を実行して、package.json を削除します。
`npm i -g @nestjs/cli` を実行し Nest.Js をインストールします。
インストール後`nest new packages/backend` で NestJS の初期設定を行います。
NestJS の作成が完了したら、package.json の name プロパティを「@monorepo-firebase/backend」に変更します。
一旦フロントエンドの用意をします。

### フロントエンド側の準備 part1

フロントエンドでは React を使用します。
まず packages/frontend の package.json を削除します。
その後、ルートディレクトリ配下`npm init vite@latest`で vite を使い、React のプロジェクトを作成します。
Vite をインストールしていない場合はここで、インストールも一緒にするか聞かれるので、インストールします。
プロジェクト名を「packages/frontend」にして後は Typescript と React を選択します。
React プロジェクトが完了したら、package.json の name プロパティを「@monorepo-firebase/frontend」に変更します。

### プロジェクト全体の準備 Part2

ルートディレクトリに戻り、ルートディレクトリにある package.json に以下のスクリプトを追加します。

```tsx
"prepare": "npm run compile",
"compile": "tsc -b tsconfig.json ",
"start:dev:api": "npm run start:dev -w @monorepo-firebase/backend",
"start:app": "npm run dev -w @monorepo-firebase/frontend"
```

そして、tsconfig.build.json を作成し、以下のコードを記載します。

```tsx
{
    "files": [],
    "references": [
        { "path": "packages/common" },
        { "path": "packages/backend" },
        { "path": "packages/frontend" },
    ]
}
```

なお、上記の記載は別に tsconfig.json でも構いません。
references ははっきりと理解はできませんでしたが、複数のプロジェクトがあってもよしなにビルドしてくれるそうです。
これらの設定が完了したら、`npm run compile`でビルドを実行します。
そして、`npm install`を実行します。
以上でルートプロジェクトの設定は概ね完了ですが、フロントエンドとバックエンドに ts-rest で作った API 定義ファイルを読み込ませておきます。
ルートプロジェクトで`npm install @monorepo-firebase/ts-router -w @monorepo-firebase/frontend`と`npm install @monorepo-firebase/ts-router -w @monorepo-firebase/backend` を実行します。

### バックエンド側の準備 part2

ルートディレクトリで`npm install @ts-rest/nest firebase-admin -w @monorepo-firebase/backend` を実行し、バックエンドに NestJS に対応した ts-rest のモジュールとバックエンドで使用する Firebase Admin SDK をインストールします。
そして、packages/backend 配下に.env ファイルを記載します。

```
CLIENTE_MAIL="秘密鍵内のclientEmail"
PROJECT_ID="秘密鍵内のclientId"
PRIVATE_KEY="秘密鍵内のprivateKey"
```

ルートディレクトリで`npm i --save @nestjs/config -w @monorepo-firebase/backend`を実行して、環境変数を読み込むモジュールをインストールします。
モジュールを使うために、app.module.ts を以下のように変更します。

```tsx
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { ConfigModule } from "@nestjs/config";
@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

isGlobal を true にしておくことで、他のモジュールでインポートを行う必要がなくなるので、treu にしています。
次に、packages/backend 内に行き、nest コマンドで firebase.module.ts と firebase.service.ts を作成します。
firebase.service.ts は以下のように記載します。

```tsx
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { credential } from "firebase-admin";
import { initializeApp } from "firebase-admin/app";
import { Auth, getAuth } from "firebase-admin/auth";
@Injectable()
export class FirebaseService {
  private readonly auth: Auth;
  constructor(private readonly configService: ConfigService) {
    initializeApp({
      credential: credential.cert({
        clientEmail: this.configService.getOrThrow<string>("CLIENTE_MAIL"),
        privateKey: this.configService.getOrThrow<string>("PRIVATE_KEY"),
        projectId: this.configService.getOrThrow<string>("PROJECT_ID"),
      }),
    });
    this.auth = getAuth();
  }
}
```

コンストラクタで Firebase の秘密鍵情報を読み込ませて、Firebase Admin の機能がまとまっている getAuth 関数を auth プロパティに代入しています。
注意点として、initializeApp 関数はインポートするディレクトリに`'firebase-admin/app'`と`'firebase-admin'`があります。
今回の書き方では`'firebase-admin/app'`からインポートしないとエラーが発生するので、必ず'firebase-admin/app'の initializeApp 関数をインポートしてください。
以上で Fireabse Admin の設定も完了しました。
バックエンド側の準備は一通り完了しましたが、最後に一旦動くことを確認するため app.controller.ts に以下のメソッドを追加します。

```tsx
import { TsRestHandler, tsRestHandler } from '@ts-rest/nest';
import { c } from './contract';
//...略
  @TsRestHandler(c.hello)
  async getHello() {
    return tsRestHandler(c.hello, async () => {
      return {
        status: 200,
        body: { name: 'Hello' }
      }
    })
  }
//...略
```

ts-rest を使用するときは`@TsRestHandler`に作成した API の定義を指定すれば、NestJS パスやリクエストボディの型などを設定しなくても勝手作成してくれます。
また、内部で`tsRestHandler`を使用すれば必要な戻り値の型補完を行ってくれます。
なので、ts-rest を用いて NestJS を実装する際はメソッドの上に`@TeRestHandler`を、メソッド内では`tsRestHandler`を使用するようにしましょう。

### フロントエンド側の準備 part2

ルートディレクトリで`npm install @ts-rest/react-query @tanstack/react-query w @monorepo-firebase/frontend`を実行し、フロントで ts-rest で設定した API の仕様を読み込めるようにします。
実際に読み込む前に React Query を使用するために、main.tsx を以下のように QueryClientProvider で囲むようにします。

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
const queryClient = new QueryClient();
ReactDOM.createRoot(document.getElementById("root")!).render(
  <QueryClientProvider client={queryClient}>
    <React.StrictMode>
      <App />
    </React.StrictMode>
  </QueryClientProvider>
);
```

次にフロントエンドプロジェクトの src 配下に任意のファイル(今回は ts-rest-client.ts)を作成し、以下のコードを記載します。

```tsx
import { tsRestRoute } from "@monorepo-firebase/ts-router";
import { initQueryClient } from "@ts-rest/react-query";
export const tsRestClient = initQueryClient(tsRestRoute, {
  baseUrl: window.location.origin,
  baseHeaders: {},
});
```

これで ts-rest による API 仕様を読み込むことができました。
早速使用してみたいところですが、このままでは CORS エラーが発生するので、vite.config.ts ファイルを以下のように変更します。

```tsx
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:3000",
      },
    },
  },
});
```

defineConfig の引数にプロキシーを設定することで、CORS エラーを回避できます。
これで懸念事項はなくなったので、App.tsx を以下のように記載します。

```tsx
import { useEffect, useState } from "react";
import reactLogo from "./assets/react.svg";
import viteLogo from "/vite.svg";
import "./App.css";
import { tsRestClient } from "./ts-rest-client";
function App() {
  const { data } = tsRestClient.hello.useQuery([]);
  const [hello, setHello] = useState("");
  useEffect(() => {
    if (data?.body) {
      setHello(data.body.name);
    }
  }, [data]);
  return (
    <>
      <div>
        <a href="https://vitejs.dev" target="_blank">
          <img src={viteLogo} className="logo" alt="Vite logo" />
        </a>
        <a href="https://react.dev" target="_blank">
          <img src={reactLogo} className="logo react" alt="React logo" />
        </a>
      </div>
      <h1>{hello}</h1>
    </>
  );
}
export default App;
```

これら記載が完了したら、ルートディレクトリで`npm run compile`を実行した後、`npm run start:dev:api`でバックエンドを、`npm run start:app` でフロントエンドを起動します。
http://localhost:5173 へアクセスすると、以下のような画面が表示されていると思います。
![2023-11-04_21h00_51.png](/images/firebase-login-spa/2023-11-04_21h00_51.png)
画面の確認ができたら、`npm install firebase -w @monorepo-firebase/frontend`でフロントエンドに Firebase を導入します。
そして、まだ実装はしませんが Firebase のプロジェクトを作成した時に表示された Firebase SDK を使うための初期値を以下のように.env.local に設定しておきます。

```tsx
VITE_APIKEY = "Firebaseで表示された値";
VITE_AUTHDOMAIN = "Firebaseで表示された値";
VITE_PROJECTID = "Firebaseで表示された値";
VITE_STORAGEBUCKET = "Firebaseで表示された値";
VITE_MESSAGINGSENDERID = "Firebaseで表示された値";
VITE_APPID = "Firebaseで表示された値";
```

ちなみに、[Vite のドキュメント](https://ja.vitejs.dev/guide/env-and-mode.html)を確認すると、クライアント側で環境変数の値を使用するには先頭に「VITE*」をつける必要があります。
今回はクライアント側で使用するためすべての変数に「VITE*」を付与しています。
以上で認証機能を実装する準備が整いました。
この時点でもはや疲労感で一杯ですが、ようやく本題の認証機能を搭載していきます。

## Firebase を使った認証機能を実装

ここからは認証機能を作成していきます。
Firebase を使うと非常に簡単にできるのを実感してもらえたらと思います。
今回はソーシャルログインしか行いませんが、通常のメールアドレスとパスワードを使ったログインもできますので、この記事を見て気になった方がいればぜひ試してみてください。

### ソーシャルログイン機能

**バックエンド側**
まずはログインをした時にバックエンドでセッションを持つための API を実装します。
auth.controller.ts に以下のメソッドを追加します。

```tsx
@TsRestHandler(c.login)
    async login(@Res({ passthrough: true }) response: Response) {
        return tsRestHandler(c.login, async ({ body }) => {
            const expiresIn = 5 * 60 * 1000;
            try {
                const sessionToken = await this.firebaseService.auth.createSessionCookie(body.idToken, { expiresIn });
                const options = {
                    maxAge: expiresIn,
                    httpOnly: true,
                    secure: false
                }
                response.cookie('sessionToken', sessionToken, options);
                return {
                    status: 201,
                    body: undefined,
                }
            } catch (e) {
                throw new BadRequestException('invalid_id_token')
            }
        })
    }
```

user.controller.ts に以下のメソッドを追加します。
リクエストに含まれるている ID トークンを Firebase に渡し、ID トークンが Fireabse 発行のものでかつ適切なものであれば、セッション用のトークンを作成します。
トークン作成時には有効期限も設定できるので、今回は 5 分を設定しています。
そして、Cookie に保存する際のオプションも options 変数に定義します。
最後に Cookie に「sessionToken」というキーで作成したトークンを設定します。
すべてが問題なければ、201 の成功レスポンスを返すようにします。
ちなみに、@Res を設定している場合、中のプロパティで`passthrough: true`を設定しないと自前で NestJS の戻り値として適切な設定を開発者側で行う必要があります。
`passthrough: true`を設定すればよしなにやってくれるので、基本的に Response を操作するために引数へ追加したら`passthrough: true`は付与するようにしましょう。
ここまでで、ログインセッションを Cookie に保存する処理を見てきました。
しかし、実は現状では作成したトークンを Cookie に保存することはできませ。
これは、NestJS で Cookie を設定する処理が完了していないからです。
なので、次の API を実装する前に Cookie を使えるようにします。
ルートディレクトリで`npm i cookie-parser -w @monorepo-firebase/backend` と`npm i -D @types/cookie-parser -w @monorepo-firebase/backend` を実行して Cookie の操作に必要なモジュールをインストールします。
そして、main.ts に以下のコードを追加します。

```tsx
import * as cookieParser from "cookie-parser";
//...省略
app.use(cookieParser());
//...省略
```

`app.use(cookieParser());`は bootstrap 関数内に記載します。
これで Cookie をセットしたり、取得したりすることが可能となりました。
最後に Cookie の値をとるためのデコレーターを作成します。
これを作成すれば、一々引数に Express の引数を設定して、メソッド内でキーを指定して Cookie の値を取得する必要が無くなります。
任意の場所に Cookie.ts を作成して、以下のコードを記載します。

```tsx
import { createParamDecorator, ExecutionContext } from "@nestjs/common";
export const Cookies = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return data ? request.cookies?.[data] : request.cookies;
  }
);
```

NestJS が提供している createParamDecorator 関数はリクエストやレスポンスの値を取得することができます。
そのため今回は、キーを指定すればリクエスト内にある指定 Cookie の値を取得し、無ければ Cookie 全体の値を返す処理を実装しています。
これでログインセッションの作成と Cookie の操作できるようになりました。
なので、次はログインしている時にユーザー情報を取得できる API を作成します。
user.controller.ts に以下のメソッドを追加します。

```tsx
@TsRestHandler(c.getUserInfo)
    async getLoginUserInfo(@Cookies('sessionToken') session: string) {
        return tsRestHandler(c.getUserInfo, async () => {
            if (!session) {
                throw new UnauthorizedException('invalid_session');
            }
            try {
                const decodeClaims = await this.firebaseService.auth.verifySessionCookie(session, true);
                return {
                    status: 200,
                    body: { name: decodeClaims.name, email: decodeClaims.email },
                }
            } catch (e) {
                console.log(e)
                throw new UnauthorizedException('invalid_session')
            }
        })
    }
```

セッションにあるトークンを Firebase に渡し、トークンが Firebase 発行のものであれば紐づくユーザー情報を取得し、欲しい情報を返すようにしています。
これでソーシャルログイン周りの API は実装完了です。
**フロント側**
packages/frontend のディレクトリにある、App.tsx を以下のように実装します。

```tsx
import { useEffect, useState } from 'react'
import './App.css'
import { tsRestClient } from './ts-rest-client'
import { Auth, AuthProvider, GoogleAuthProvider, getAuth, getIdToken, getRedirectResult,  signInWithRedirect } from 'firebase/auth'
function App() {
  const [name, setName] = useState('')
  const [email, setEmail] = useState('')
  const [isLoggedIn, setLoggedIn] = useState(false);
  const auth = getAuth()
  const { data, isLoading } = tsRestClient.getUserInfo.useQuery([])
  const { mutate: loginMutate } = tsRestClient.login.useMutation()
  useEffect(() => {
    (async () => {
      try {
        const redirectSignInResult = await getRedirectResult(auth);
        if (redirectSignInResult) {
          const idToken = await getIdToken(redirectSignInResult.user, true);
          loginMutate({ body: { idToken } })
        }
        if (!data?.body) {
          return;
        }
        setName(data.body.email ?? '');
        setEmail(data.body.email ?? '');
        setLoggedIn(true);
      }
      catch (e) {
        setLoggedIn(false);
      }
    })()
  }, [isLoading])
  const login = async (auth: Auth, provider: AuthProvider) => {
    await signInWithRedirect(auth, provider)
  }
  return (
    <>
      {isLoading && <div>ローディング中</div>}
      {!isLoading &&
        <div>
          <h3>名前:{name}</h3>
          <h3>メールアドレス:{email}</h3>
          <div>
            {isLoggedIn ? <button onClick={() => logout()}>ログアウト</button> : <button onClick={() => login(auth, new GoogleAuthProvider())}>ログイン</button>}
          </div>
      }
    </>
  )
}
export default App
```

React はド素人なので、useEffect や useState の解説はしません。
一方で、Firebase に関わる部分については少しみていきます。
まずは[signInWithRedirect 関数](https://firebase.google.com/docs/reference/js/auth.md?hl=ja#signinwithredirect)です。
この関数はサインイン画面にリダイレクトさせるための関数です。
第一引数にソーシャルログインをさせるサービスと連携する Firebase の情報を設定し、第二引数にログイン画面に遷移させるサービスを設定します。
これを行うことで、ソーシャルログインまですることができます。
とても便利ですね。
なお、[signInWithRedirect 関数](https://firebase.google.com/docs/reference/js/auth.md?hl=ja#signinwithredirect)は、第二引数に設定するサービスが OAuth を対応している必要があります。
OAuth に対応していないものにするとエラーが発生しますので、ご注意ください。
ソーシャルログインは[signInWithRedirect 関数](https://firebase.google.com/docs/reference/js/auth.md?hl=ja#signinwithredirect)でできますが、ログインした後アプリケーションにリダイレクトされた後の処理は対応できておりません。
そこで、リダイレクト後の処理に使うのが[getRedirectResult 関数](https://firebase.google.com/docs/reference/js/v8/firebase.auth.Auth#getredirectresult)です。
[getRedirectResult 関数](https://firebase.google.com/docs/reference/js/v8/firebase.auth.Auth#getredirectresult)はログインによるリダイレクトの場合のみ、ログインを行った情報を取得できます。
なので、[getRedirectResult 関数](https://firebase.google.com/docs/reference/js/v8/firebase.auth.Auth#getredirectresult)に値が存在する場合はログイン完了したので、先の処理を行うようにしています。
その際に使用するのが getIdToken 関数です。
getIdToken 関数にログインしたユーザーの情報を渡せば、そのユーザーに対しての ID トークンを取得できます。
ID トークンは認証したことを示すトークンで、改ざんを防御する仕組みが備わっています。
すなわち、ログインしたユーザーの情報をそのまま渡すより安全なものとなっています。
クライアント側のやることが終わったので、最後に`loginMutate({ body: { idToken } })`でログインセッションを作成する API を呼び出します。
これによって、ログインセッションが作られて画面にログインしたユーザーの情報が表示されます。
以上でソーシャルログイン周りは完了です。

### ログアウト・退会

**バックエンド側**
ログアウトと退会処理は大体同じなので、まとめて実装します。
auth.controller.ts に以下のメソッドを追加します。

```tsx
@TsRestHandler(c.logout)
    async logout(@Res({ passthrough: true }) response: Response, @Req() req: Request) {
        return tsRestHandler(c.logout, async () => {
            const tokenCookie: string = req.cookies?.sessionToken ?? '';
            const decodeClaims = await this.firebaseService.auth.verifySessionCookie(tokenCookie, true)
            await this.firebaseService.auth.revokeRefreshTokens(decodeClaims.uid);
            response.clearCookie('sessionToken')
            return {
                body: null,
                status: 201
            }
        })
    }
```

このメソッドはログアウトのメソッドです。
Cookie に保存したトークンが適切であれば、Firebase のログインセッションを消去し、API のログインセッションもクリアにしています。
次に、user.controller.ts に以下のメソッドを追加します。

```tsx
@TsRestHandler(c.deleteUser)
    async deleteUser(@Req() req: Request, @Res({ passthrough: true }) res: Response) {
        return tsRestHandler(c.deleteUser, async () => {
            const tokenCookie: string = req.cookies?.sessionToken ?? '';
            const decodeClaims = await this.firebaseService.auth.verifySessionCookie(tokenCookie, true)
            await this.firebaseService.auth.revokeRefreshTokens(decodeClaims.uid);
            res.clearCookie('sessionToken')
            return {
                body: null,
                status: 201
            }
        })
    }
```

こちらは退会処理の API です。
やっていることはログアウトの時とほぼ同じですが、Firebase の deleteUser メソッドを使用してログインセッションではなく、ユーザー情報そのものを削除しています。
**フロント側**
packages/frontend の App.tsx 内に以下のメソッドを追加します。

```tsx
//...略
const { mutate: logoutMutate } = tsRestClient.logout.useMutation();
const { mutate: deleteUser } = tsRestClient.deleteUser.useMutation();
const logout = async () => {
  try {
    logoutMutate({});
    location.href = window.location.origin;
  } catch (e) {
    alert("エラーが発生しログアウトが適切にできませんでした。");
  }
};
const withdraw = () => {
  try {
    if (confirm("退会します。よろしいでしょうか？")) {
      deleteUser({});
      location.href = window.location.origin;
    }
  } catch (e) {
    alert("退会に失敗しました。");
  }
};
//...略
```

API を実行するための値を取得し、それを呼び出すかんすうをそれぞれ作成しています。
そして、ログアウト・退会が正常に完了したら TOP ページに戻るようにしています。
そして、戻り値に以下のコードを記載します。

```tsx
<div>
  {isLoggedIn ? <button onClick={() => logout()}>ログアウト</button> : <button onClick={() => login(auth, new GoogleAuthProvider())}>ログイン</button>}
</div>
<div>
  {isLoggedIn && <button onClick={withdraw}>退会する</button>}
</div>
```

以上でログアウトと退会処理の実装が完了しました。

### メールリンクログイン

ここではメールリンクログインを実装します。
メールリンクログインとは画面のフォームでメールアドレスを入力して、メールアドレスに届くリンクをクリックすることでログインする方法です。
実際に持っているメールアドレスならばパスワードを入力せずともログインすることが可能となります。
メールのリンクを取得して、リンクからアクセスする必要があり手間を要するので、ログインの際の第一候補にはなりませんが、ソーシャルログインができないときなどの対応方法として実装するようにしています。
では実際に実装を行います。
**フロント側**
今回はバックエンドはないので、フロント側のみ行います。
まずはメールリンクを送るために App.tsx 内に下記のように作成します。

```tsx
//...略
const [formMail, setFormMail] = useState("");
//...略
const sendMailLink = () => {
  const actionCodeSettings = {
    url: location.origin,
    handleCodeInApp: true,
  };
  sendSignInLinkToEmail(auth, formMail, actionCodeSettings)
    .then(() => {
      alert("メールを送信しました。");
    })
    .catch(() => {
      alert("メールの送信ができません。");
    });
};
//...略
```

変数 actionCodeSettings は Firebase がメールを送る時の設定値です。
url は Firebase がメール認証をした後、リダイレクトさせる URL を指定します。
handleCodeInApp はリンクを押したとき、モバイルアプリで開くかウェブリンクかを決める設定です。
true だとモバイルアプリで開くことになるので、本来であれば設定する必要がないのですが、何故か true にしないと動きませんでした。
そのため、true に設定しています。
変数 actionCodeSettings を設定したら、sendSignInLinkToEmail 関数で使用する Firebase とメールを送るアドレスを指定して、最後に先程設定したオプションを指定します。
これによって、メールリンクログインの準備ができました。
では、実際に上記関数を実行するために、戻り値にもメールリンクログイン用のフォームを用意します。

```tsx
{
  !isLoggedIn && (
    <div>
      <p>メールリンクログイン</p>
      <input type="email" onChange={(e) => setFormMail(e.target.value)} />
      <button onClick={() => sendMailLink()}>送信</button>
    </div>
  );
}
```

最後に useEffect の中に以下の内容を記載します。

```tsx
if (isSignInWithEmailLink(auth, window.location.href)) {
  try {
    const email = window.prompt("確認のためメールアドレスを入力してください");
    const result = await signInWithEmailLink(
      auth,
      email ?? "",
      window.location.href
    );
    const idToken = await getIdToken(result.user, true);
    loginMutate({ body: { idToken } });
  } catch (e) {
    alert("ログインに失敗しました");
  } finally {
    location.href = window.location.origin;
  }
}
```

まずは isSignInWithEmailLink 関数で URL がメールリンクかを判定します。
もしメールリンクであれば、`window.prompt`を用いて画面上で再度メールアドレスを入力させるようにして、仮にリンクが盗まれてもメールアドレスを知っていないとログインできないようにします。
入力したメールアドレスとメールリンクを signInWithEmailLink 関数に渡し、Firebase 上でログインするようにします。
ログインが完了するとユーザー情報を返すので、その情報を基に getIdToken 関数で ID トークンを取得します。
最後に ID トークンを渡し、API 側でもログインセッションを作成するようにします。
以上でメールリンクログインは完了です。

### 補 API の中身を確認する

API を作成するための準備を行うために、[ts-rest](https://ts-rest.com/docs/intro)というライブラリを使いました。
このライブラリによって、API の定義を一つにまとめることが可能となりました。
しかし、ts-rest で実装したコードで API の定義を追って必要なパラメータや戻り値を確認するのは少し手間です。
そこで、ここでは Swagger を導入し現在定義している API をブラウザ上で確認できるようにします。
では早速実装していきます。
`npm install @ts-rest/open-api -w @monorepo-firebase/backend` を実行して、API の定義書を Swagger に読み込ませるためのモジュールをインストールします。
インストールが完了したら、main.ts に以下のコードを追記します。

```tsx
const document = generateOpenApi(
  contract,
  {
    info: {
      title: "Posts API",
      version: "1.0.0",
    },
  },
  {
    setOperationId: true,
    jsonQuery: true,
  }
);
SwaggerModule.setup("define-api", app, document);
```

これで、定義書を Swagger に読み込ませる処理は完了です。
http://localhost:3000/define-api にアクセスすると、以下のような API 定義一覧画面が表示されます。
![2023-11-11_09h07_27.png](/images/firebase-login-spa/2023-11-11_09h07_27.png)

# Firebase は Auth0 の代わりになり得るか？

最後に Firebase は Auth0 の代わりになるかを見ていきます。
早速結論ですが、Firebase は Auth0 の代わりにならないと考えた方が無難です。
その理由は以下の通りです。

- Firebase は認証機能の提供はしているが認証・認可サーバーは準備していない
- Firebase は OIDC をサポートするユーザーが無料枠で月 50 人未満と少ない

それぞれ見ていきましょう。
まず一つ目についてです。
Firebase を Auth0 の代わりにするには以下のような構成を満たす必要があります。
![**[Flutter × FirebaseAuth 自作APIの認証について](https://zenn.dev/ymktmk/articles/cc56296b096f39)から引用**](/images/firebase-login-spa/Untitled.png)
**[Flutter × FirebaseAuth 自作 API の認証について](https://zenn.dev/ymktmk/articles/cc56296b096f39)から引用**
そして、認証のフローは Auth0 から置き換えると考えるなら、OpenId Connect であることが求められます。
Firebase を使ってこれら構成を作成することは不可能ではありません。
認証の機能は揃っていますし、トークンを生成することもできます。
しかし、Firebase 単体で Authorization Code Flow における認可サーバーになるには、別途開発者側での実装が必要となります。
それを示すために、以下の図を見てください。
![[Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow)より引用](/images/firebase-login-spa/Untitled1.png)
[Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow)より引用
Authorization Code Flow を行うには、アプリから認可サーバーへ来た/authorize へのリクエストに対して認証・認可を行い、認可コードのやり取りをした後にトークンを渡す必要があります。Firebase は上記 Authorization Code Flow で以下画像の赤枠内のみをサポートしています。※
![2023-10-02_21h46_13.png](/images/firebase-login-spa/2023-10-02_21h46_13.png)
※③ と ④ しかサポートしていないわけではなく、③ と ④ の全てに対応しているわけではないです。大枠で判断した結果大体 ③ と ④ の機能を担うくらいで捉えていただけると幸いです。
その他の機能については我々が用意するものとなっています。
これは Firebase Authenticate は**「認証」**機能をサポートしているが認証の結果何がしたいかは実装者に委ねているためです。
一方で、Auth0 は画像で示したフローを行うのに必要な機能が搭載されています。
Auth0 は認証に加えて認可についての機能を搭載するためです。
そのため、Auth0 は自由度がそこまで高くないですが、OpenId Connect を使用した認証・認可機能を使いたい場合は簡単に構築ができるようになっています。
Firebase でも OpenId Connect を構築するのは不可能ではないですが、考えることが多く Auth0 の代わりで使おうとすると苦しいものがあります。
ただ、Firebase は Identity Platform と連携を行うことで OpenId Connect を使用した認証はできます。
なら、Firebase でも良さそうに思えますが二つ目の理由がそれを阻みます。
それは OpenId Connect を使用する場合、無料で使用できるのが月 50 人未満までとなっています。
この 50 人とはアクティブユーザーのことを指しており、Identity Platform が定義するアクティブユーザーは月の中で一回でもログインしたユーザーとなっています。([参考資料](https://cloud.google.com/identity-platform/pricing?hl=ja))
あまりに少ないとまでは言いませんが、少し心もとないです。
一方で、Auth0 は合計 7000 人までのアクティブユーザーであれば無料で使用することができます。
なお、Auth0 におけるアクティブユーザーは[こちら](https://community.auth0.com/t/what-is-meant-my-monthly-active-users-for-auth0/104447/5)や[こちら](https://portal.saatisfy.jp/hc/ja/articles/900006396623-Auth0%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8BMAU%E3%81%AE%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E6%96%B9%E6%B3%95)の資料を確認する限り Firebase と同様、概ね一か月の間にログインしたユーザーと認識して良さそうです。
このように料金体系が異なることから、Auth0 の代わりに Firebase へ乗り換えることをした場合予期せぬ料金が生じるかもしれません。
お金の面でも、簡単に乗り換えるといったことは考えることが多く難しそうです。
以上 Firebase は Auth0 の代わりにならない理由を記載しました。
注意点として、私の Firebase の理解はまだまだ全然浅い初心者です。
そのため、今回は Auth0 の代わりにならないとしましたが、もしかすると他の機能で対応できる可能性はあるので、話半分で読んでいただけると幸いです。

# おわりに

今回は Firebase を用いてソーシャルログイン機能を実装しました。
ts-rest の導入や React の機能がよくわからないといった、Firebase 以外の部分で苦しみましたが Firebase 自体は簡単に実装できました。
これだけ簡単なら、今後は認可サーバー的なものを作る際内部の認証管理は Firebase だけに任せてがわを自分で作るといったことが出来そうです。
夢が広がりますね。
ここまで読んでいただきありがとうございました。

# 参考資料

- TS-REST を使って open api を導入する方法
  [https://ts-rest.com/docs/open-api](https://ts-rest.com/docs/open-api)
- NestJs に Swagger 導入
  [https://docs.nestjs.com/openapi/introduction](https://docs.nestjs.com/openapi/introduction)
- モノレポの環境構築
  [https://note.com/shift_tech/n/nbecb007ac2ee](https://note.com/shift_tech/n/nbecb007ac2ee)
- モノレポの操作
  [https://blog.covelline.com/entry/2021/11/02/154101](https://blog.covelline.com/entry/2021/11/02/154101)
- NestJS に Cookie 導入する
  [https://docs.nestjs.com/techniques/cookies](https://docs.nestjs.com/techniques/cookies)
- Firebase のメールリンクログインのパラメータたち
  [https://firebase.google.com/docs/auth/web/passing-state-in-email-actions?hl=ja#passing_statecontinue_url_in_email_actions](https://firebase.google.com/docs/auth/web/passing-state-in-email-actions?hl=ja#passing_statecontinue_url_in_email_actions)
