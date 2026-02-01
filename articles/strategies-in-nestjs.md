---
title: "NestJSでのStrategyマスターに俺はなる!!!"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS", "passport"]
published: true
---

# はじめに

以前[NestJS のドキュメント](https://docs.nestjs.com/security/authentication)をもとに認証方法を[実装する記事](https://zenn.dev/maronn/articles/authenticate-document-in-nestjs)をあげました。
もちろん、上記やり方でもいいのですが世の中には passport という便利なモジュールがあります。
これを使えば多種多様な検証を行うことができ、とても有用です。
そこで今回は、[ドキュメント](https://docs.nestjs.com/recipes/passport)から passport モジュールを NestJS で使う方法を学び、passport モジュール周りの内部構造を見て、最後に passport モジュール周りをカスタマイズしていきます。
気づいたら中々長い記事になってしまいましたが、読んでいただければ幸いです。

# ドキュメントから passport を使った NestJS での認証を学ぶ

## 導入

passport モジュールは多くのストラテジーがあり、様々な認証に対応しています。
ですが、数がとてもおおくどれを使ってよいか分からなくなります。
そこで@nestjs/passport を用いて、NestJS に対応したストラテジーを自動的に選択できるようになっています。
[ドキュメント](https://docs.nestjs.com/recipes/passport)では@nestjs/passport を合わせて、passport モジュールの使い方を確認するものとなっています。

## ドキュメントの要件

[ドキュメント](https://docs.nestjs.com/recipes/passport)では以下の機能を実装していきます。
① ユーザー名/パスワードでの認証
② ユーザー名/パスワード認証をガード化
③ 認証後の JWT 発行
④Authorization Header に存在する JWT の検証
⑤JWT 検証をガード化
なお、ユーザー名/パスワード認証と JWT 発行と JWT の検証はそれぞれ使用するモジュールが若干異なります。
詳細はそれぞれ見ていきますが、ユーザー名/パスワード認証は`passport-local`モジュール周りの機能を、JWT 発行は`@nestjs/jwt`モジュール周りの機能を、JWT の検証は`passport-jwt`モジュール周りの機能を使用します。
それぞれの機能は密接な関係というわけではないので、JWT の検証部分だけを自身のプロジェクトに活かすといったことができます。
そのため、「[Implementing Passport local](https://docs.nestjs.com/recipes/passport#implementing-passport-local)」と「[JWT functionality](https://docs.nestjs.com/recipes/passport#jwt-functionality)」と「[Implementing Passport JWT](https://docs.nestjs.com/recipes/passport#implementing-passport-jwt)」の章は独立したものと考えた方がドキュメント全体の見通しが良くなります。
要件とドキュメントの構造について軽くみたので、まずユーザー名/パスワード認証から実装していきます。

## passport-local を使ったユーザー名/パスワード認証とガード化

passport-local を使用する前に疑似的なログイン処理が必要となるので、以下のコードをそれぞれ作成してください。
users/users.service.ts

```tsx
import { Injectable } from "@nestjs/common";
// This should be a real class/interface representing a user entity
export type User = any;
@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: "john",
      password: "changeme",
    },
    {
      userId: 2,
      username: "maria",
      password: "guess",
    },
  ];
  async findOne(username: string): Promise<User | undefined> {
    return this.users.find((user) => user.username === username);
  }
}
```

users/users.module.ts

```tsx
import { Module } from "@nestjs/common";
import { UsersService } from "./users.service";
@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

auth/auth.service.ts

```tsx
import { Injectable } from "@nestjs/common";
import { UsersService } from "../users/users.service";
@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
```

auth/auth.module.ts

```tsx
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { UsersModule } from "../users/users.module";
@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
```

下準備ができたので、passport-local を使っていきます。
まず、以下のコマンドで passport-local を使うためのモジュールをインポートします。

```
$ npm install --save @nestjs/passport passport passport-local
$ npm install --save-dev @types/passport-local
```

実際に活用する機能は passport-local の機能ですが、それを呼び出すために passport モジュールが必要で、passport モジュールを NestJS で使いやすい形にしたのが@nestjs/passport です。
そのため、計 3 つのモジュールをインポートしています。
ちなみに、先程「[Implementing Passport local](https://docs.nestjs.com/recipes/passport#implementing-passport-local)」、「[JWT functionality](https://docs.nestjs.com/recipes/passport#jwt-functionality)」、「[Implementing Passport JWT](https://docs.nestjs.com/recipes/passport#implementing-passport-jwt)」の章は独立して考えて良いと言いましたが、モジュールのインポートは前にインストールしたものが導入されている前提で話が進みます。
なので、各章だけを見る場合それぞれ以下のモジュールのインストールが必要なのでご注意ください。

- ユーザー名/パスワード認証とガード化
  - dependencies:`@nestjs/passport`、`passport` ,`passport-local`
  - devDependencies:`@types/passport-local`
- JWT 発行
  - dependencies:`@nestjs/jwt`
- JWT 検証とガード化 - dependencies:`@nestjs/passport`、`passport` ,`passport-jwt` - devDependencies:`@types/passport-jwt`

インストールしたら、auth フォルダ配下に local.strategy.ts を作成し、以下のコードを記載します。(フォルダの場所とファイル名は任意のもので良いです。)

```tsx
import { Strategy } from "passport-local";
import { PassportStrategy } from "@nestjs/passport";
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { AuthService } from "./auth.service";
@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }
  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

上記コードはリクエストボディかクエリににユーザー名(username)とパスワード(password)が存在しているかを確認し、存在すれば一致するユーザーが存在するかを validateUser メソッドで確認します。
もし存在しなければ、401 エラーを発生させ、存在する場合は取得した値を返すようにしています。
処理の中身は上記のようになりますが、基本的に passport モジュールを使用して何かしらの検証を行うクラスを作成する場合以下の流れになります。
① 引数に使用したいストラテジーを入れて PassportStrategy 関数を呼び出す。
②① を extends して継承する。
③ コンストラクタで親のコンストラクタを実行させる。
④validate メソッドを実装して、独自に検証したい内容を実装する。
② の継承についてですが、こちらは PassportStrategy 関数が抽象クラスを返すため必須となります。
PassportStrategy 関数の詳細は後ほど見ていくので、一旦ここでは抽象クラスを返すから extends しないといけないと認識ください。
③ で親のコンストラクタを実行させるのも使用するストラテジーに必要な値を格納するために必須のコードとなります。
ストラテジーのコードの詳細も JwtStrategy だけですが、後ほど詳細に見ていきます。
なので、一旦はストラテジーを実行できるようにするために必ず親のコンストラクタも呼ぶ必要があると認識してください。
最後に ④ の validate メソッドですが、こちらはストラテジーによる検証が終わった後実行させる独自のメソッドとなります。
ここで、自前のデーターベースとデータが一致するなどの検証を行うことができます。
注意点として、特に独自で検証したいことが無くても必ずこのメソッドは定義をして、何かしらの値を返すようにしてください。
上記理由についてはここで軽く説明します。
まずメソッドを定義する必要がある理由ですが、以下のように PassportStrategy 関数が返す抽象クラスに理由があります。

```tsx
export function PassportStrategy<T extends Type<any> = any>(
  Strategy: T,
  name?: string | undefined,
  callbackArity?: true | number
): {
  new (...args): InstanceType<T>;
} {
  abstract class MixinStrategy extends Strategy {
    abstract validate(...args: any[]): any;
    //...略
  }
  return MixinStrategy;
}
```

PassportStrategy 関数が返している MixinStrategy クラスは validate メソッドが定義されています。
なので、validate メソッドはある前提でこの後の処理が実行されるためない場合はエラーが発生します。
これが validate メソッドを実装する必要がある理由です。
次に validate メソッドは何かしらの値を返す必要がある理由ですが、今回作成したクラスを使用するガード処理に理由があります。

```tsx
function createAuthGuard(type?: string | string[]): Type<IAuthGuard> {
  class MixinAuthGuard<TUser = any> implements CanActivate {
    //...略
    async canActivate(context: ExecutionContext): Promise<boolean> {
      const options = {
        ...defaultOptions,
        ...this.options,
        ...(await this.getAuthenticateOptions(context))
      };
      //...略
      request[options.property || defaultOptions.property] = user;
      return true;
    }
  //...略
}
```

注目して欲しいのは`request[options.property || defaultOptions.property] = user;`です。
request オブジェクトに代入する user は validate メソッドで設定した戻り値となります。
なぜそうなるのかは、後ほど「AuthGuard は何をしているかを見る」で触れる機会があるので、一旦はそういうものだと認識ください。
このように validate メソッドの戻り値を Request オブジェクトに代入するので、validate メソッドが何も返さないようになると後続の処理が上手くいかなくなります。
そのため、validate メソッドには何かしらの戻り値を返すようにしてください。
validate メソッドの必要性は分かったと思いますが、validate メソッドの引数はどうやったら分かるのかという疑問が残ったままです。
これについては、[こちらのサイト](https://www.passportjs.org/packages/)から辿ることができます。
上記サイトは passport が提供するストラテジーの一覧となっています。
そこから使いたいストラテジーの詳細ページに遷移し、ストラテジーをインスタンス化する時のサンプルを確認します。
するとインスタンス化する際に設定する値で、関数を定義している箇所があります。
その関数のコールバック関数を示す done 以外の引数が validate メソッドで受け取る引数となります。
言葉だけでは分かりにくいので、今回使用した passport-local について見てみます。
[passport-local](https://www.passportjs.org/packages/passport-local/)の Usage を確認すると以下のコードがあります。

```tsx
passport.use(
  new LocalStrategy(function (username, password, done) {
    User.findOne({ username: username }, function (err, user) {
      if (err) {
        return done(err);
      }
      if (!user) {
        return done(null, false);
      }
      if (!user.verifyPassword(password)) {
        return done(null, false);
      }
      return done(null, user);
    });
  })
);
```

着目するのは`function(username, password, done)`の部分です。
上記関数は LocalStrategy をインスタンス化する時に設定しており、引数 done 以外は username と password が存在します。
この部分の引数が validate メソッドに渡される引数となります。
なぜそうなるかはこの記事で解説することはありませんが、理由に関わる部分については「PassportStrategy の super()が何をしているかを見る」と「AuthGuard は何をしているかを見る」で対象のコードが出てくるはずなので、そこから推測していただければ幸いです。(すみません。ここまで解説する気力が湧いてこなかったです。)
passport-local を使用したクラスの実装が完了したので、これを実際に使用できるよう auth/auth.module.ts を編集します。

```tsx
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { UsersModule } from "../users/users.module";
import { PassportModule } from "@nestjs/passport";
import { LocalStrategy } from "./local.strategy";
@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

PassportStrategy 関数を使用しているので、@nestjs/passport のモジュールをインポートして、他のファイルで作成した LocalStrategy クラスを使用できるよう providers に追加しています。
これでユーザー名/パスワード認証を行う準備ができたので、最後にエンドポイントを作成します。
app.controller.ts を以下のように変更します。

```tsx
import { Controller, Request, Post, UseGuards } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
@Controller()
export class AppController {
  @UseGuards(AuthGuard("local"))
  @Post("auth/login")
  async login(@Request() req) {
    return req.user;
  }
}
```

@nestjs/passport モジュールが提供する AuthGuard 関数を使用すれば、先程作成した LocalStrategy クラスをガードとして実行できるようにしてくれます。
注意点として、AuthGuard 関数に設定する値は LocalStrategy クラスではなく「local」という文字列になります。
突然出てきて混乱しますが、基本的に passport-local が提供するストラテジーを使用した場合はここの値は「local」となります。
ちなみに、PassportStrategy 関数の第二引数に任意の名前を指定できるので、そこで名前を設定した場合は AuthGuard 関数を呼び出す時の値が、PassportStrategy 関数で設定した名前となります。
上記については後ほど「AuthGuard を呼び出す時の文字列って何？」で解説していますので、説明はここで留めておきます。
これで認証用のエンドポイントが作成できました。
では検証と言いたいですが、現状だとガードの拡張性が無い状態です。
そのため、任意の箇所(今回は auth/local-auth.guard.ts)に以下のコードを記載します。

```tsx
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
@Injectable()
export class LocalAuthGuard extends AuthGuard("local") {}
```

ここでは実装しませんが、上記のように定義しておくとクラスの中で様々なメソッドを設定できます。
このようにしておくと呼び出しの際のタイポ防いだりもできるので、NestJS も推奨しています。
ガード用のファイルを作成したので、先程作成したエンドポイントを修正します。

```tsx
import { Controller, Request, Post, UseGuards } from "@nestjs/common";
import { LocalAuthGuard } from "./auth/local-auth.guard";
import { AuthService } from "./auth/auth.service";
@Controller()
export class AppController {
  constructor(private authService: AuthService) {}
  @UseGuards(LocalAuthGuard)
  @Post("auth/login")
  async login(@Request() req) {
    return req.user;
  }
}
```

では、NestJS を起動させ、以下の curl をそれぞれ実行してみてください。

```
curl -X POST http://localhost:3000/auth/login -d '{"username": "notExistUser", "password": "notExistPass"}' -H "Content-Type: application/json"
curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
```

1 番目の curl コマンドは 401 エラーが返ってきて、2 番目の curl はユーザー情報が返ってくると思います。
これで要件「① ユーザー名/パスワードでの認証」と「② ユーザー名/パスワード認証をガード化」が完了しました。

## JWT の発行

ここからは認証が完了すれば、JWT を発行しそれを返すようにします。
なお、具体的な中身については「[NestJs の認証周りのドキュメントを読み直す](https://zenn.dev/maronn/articles/authenticate-document-in-nestjs#jwt-%E7%99%BA%E8%A1%8C%E5%87%A6%E7%90%86%E3%81%AE%E5%AE%9F%E8%A3%85)」で解説しているので、ここでは記載しません。
ただし、上記記事の signIn メソッドはユーザーの検証まで行っているので、その部分は不要となります。
そのため、signIn メソッドを以下のような login メソッドに変更する作業は今回別途行っています。

```tsx
async login(user: any) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
```

それでは、JWT 発行の準備が完了した後の処理から記載します。
といっても先程の app.controller.ts の login メソッドを以下のように変更して、コンストラクタに AuthService を注入した後、auth/auth.module.ts に`exports: [AuthService],`を追記するだけです。

```tsx
async login(@Request() req) {
    return this.authService.login(req.user);
  }
```

それでは`curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"`をコマンドで実行して、以下のような値が返ってくるのを確認できたら「③ 認証後の JWT 発行」は完了です。

```tsx
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImpvaG4iLCJzdWIiOjEsImlhdCI6MTcwMDcxMzY0OCwiZXhwIjoxNzAwNzEzNzA4fQ.ReYiju2HisO8WTANNdI6Fg9vcqwz4SsIHFl4W8_RY7k"}
```

## JWT の検証とガード化

認証と JWT の発行ができたので、ここからは JWT が適切な署名をされたものかを検証するためのコードとそれを基にしたガードを作っていきます。
まず、以下のコマンドを実行して必要なモジュールをインポートします。

```
$ npm install --save passport-jwt
$ npm install --save-dev @types/passport-jwt
```

その後、任意のファイル(今回は auth/jwt.strategy.ts)で以下のコードを実装します。

```tsx
import { ExtractJwt, Strategy } from "passport-jwt";
import { PassportStrategy } from "@nestjs/passport";
import { Injectable } from "@nestjs/common";
import { jwtConstants } from "./constants";
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: jwtConstants.secret,
    });
  }
  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

構造などの書き方は passport-local の時と大体同じですが、コンストラクタの設定を passport-jwt の場合は行う必要があります。
具体的には大きく分けて 3 つの設定が存在します。
①JWT の取得方法
② 検証用の鍵設定
③JWT で検証する値の設定
④validate メソッドの引数の設定※
※正確には passport-jwt で JWT の検証後に実行するコールバック関数の引数の設定とするのが正確です。ただ、それだとあまりにイメージが付きにくいので、具体名で記載しています。
そして、それぞれの設定と具体的な possport-jwt のプロパティを分類すると以下のようになります。

- ①JWT の取得方法: jwtFromRequest
- ② 検証用の鍵設定: secretOrKey,secretOrKeyProvider
- ③JWT で検証する値の設定:issuer,audience,algorithms,ignoreExpiration,jsonWebTokenOptions
- ④validate メソッドの引数の設定:passReqToCallback

④ についてだけイメージがわきにくいので補足します。
「④validate メソッドの引数の設定」は boolean を記入することができ、この役割はドキュメントを確認すると以下の通りです。

> If true the request will be passed to the verify callback. i.e. verify(request, jwt_payload, done_callback).

つまり、validate メソッドの第一引数に Request オブジェクトを追加するかを決めるプロパティとなります。
明示的に Request オブジェクトに何かしらの操作したいときはこの値を true にすると、validate メソッドで Request オブジェクトを扱えるようになります。
今回は特に操作したいことはないので、設定しません。
passport-jwt を使用する際に最低限必要なのは`jwtFromRequest` と `secretOrKey` OR `secretOrKeyProvider` なので、上記のように記載しています。
jwtFromRequest プロパティに設定している`ExtractJwt.fromAuthHeaderAsBearerToken()`は Authorization Header にある Bearer トークンを取得する際に使用するメソッドです。
その他にも様々なメソッドがあるので、気になる方は[こちらのファイル](https://github.com/mikenicholson/passport-jwt/blob/master/lib/extract_jwt.js)を確認してください。
secretOrKey プロパティは鍵情報を String 型か Buffer 型で渡す必要があります。
今回は JWT 発行の際に仮の鍵を文字列で作成しているので、それをインポートしています。
ここまで出来たら後は passport-local の時と同じです。
auth/auth.module.ts に JwtStrategy を渡します。

```tsx
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { LocalStrategy } from "./local.strategy";
import { JwtStrategy } from "./jwt.strategy";
import { UsersModule } from "../users/users.module";
import { PassportModule } from "@nestjs/passport";
import { JwtModule } from "@nestjs/jwt";
import { jwtConstants } from "./constants";
@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: "60s" },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

その後、JwtStrategy をガードかするためのファイルを作成します。
なお、passport-jwt を使用している場合、AuthGuard に設定する文字列はデフォルトで「jwt」となります。

```tsx
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

最後に app.controller.ts に今回作成したガードを適用したエンドポイントを作成します。

```tsx
import { Controller, Get, Request, Post, UseGuards } from "@nestjs/common";
import { JwtAuthGuard } from "./auth/jwt-auth.guard";
import { LocalAuthGuard } from "./auth/local-auth.guard";
import { AuthService } from "./auth/auth.service";
@Controller()
export class AppController {
  constructor(private authService: AuthService) {}
  @UseGuards(LocalAuthGuard)
  @Post("auth/login")
  async login(@Request() req) {
    return this.authService.login(req.user);
  }
  @UseGuards(JwtAuthGuard)
  @Get("profile")
  getProfile(@Request() req) {
    return req.user;
  }
}
```

では最後に以下の curl コマンドを実行して、動作確認をします。

```
$ curl http://localhost:3000/profile
$ curl http://localhost:3000/profile -H "Authorization: Bearer aaaa"
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ curl http://localhost:3000/profile -H "Authorization: Bearer 取得したアクセストークン"
```

最初のコマンドはアクセストークンがないので、401 エラーが返ってきます。
2 つめのコマンドも Bearer は設定しているものの、トークンの検証によって 401 エラーが返ってきます。
3 つめのコマンドでトークンを取得してから、そのトークンを設定した時のみユーザー情報が含まれた戻り値が設定されます。
これで想定した要件を全て満たしました。
ドキュメントでやっている大まかな部分は解説したので、次からは各機能の内部について詳細に見ていきます。

# PassportStrategy の super()が何をしているかを見る

ここでは JwtStrategy を作る時に使用した下記のコードが何をしているかを見ていきます。

```tsx
import { Strategy } from 'passport-jwt';
//...略
export class JwtStrategy extends PassportStrategy(Strategy) {
    constructor() {
        super({
            jwtFromRequest://...略,
            secretOrKeyProvider: //...略
        });
    }
    async validate(payload:any){
			//...略
		}
}
```

特に見ていきたいのは以下のことです。
① コンストラクタ内で行っている親のコンストラクタが何をしているか。
②jwtFromRequest プロパティの中身
③secretOrKeyProvider プロパティの中身
それでは順に見ていきます。

## ① 親のコンストラクタが何をしているかを確認する

### PassportStrategy 関数

まず、super の対象である[PassportStrategy](https://github.com/nestjs/passport/blob/master/lib/passport/passport.strategy.ts#L4)の中身を見ていきます。

```tsx
export function PassportStrategy<T extends Type<any> = any>(
  Strategy: T,
  name?: string | undefined,
  callbackArity?: true | number
): {
  new (...args): InstanceType<T>;
} {
  abstract class MixinStrategy extends Strategy {
    abstract validate(...args: any[]): any;
    constructor(...args: any[]) {
      const callback = async (...params: any[]) => {
        const done = params[params.length - 1];
        try {
          const validateResult = await this.validate(...params);
          if (Array.isArray(validateResult)) {
            done(null, ...validateResult);
          } else {
            done(null, validateResult);
          }
        } catch (err) {
          done(err, null);
        }
      };
      if (callbackArity !== undefined) {
        const validate = new.target?.prototype?.validate;
        const arity =
          callbackArity === true ? validate.length + 1 : callbackArity;
        if (validate) {
          Object.defineProperty(callback, "length", {
            value: arity,
          });
        }
      }
      super(...args, callback);
      const passportInstance = this.getPassportInstance();
      if (name) {
        passportInstance.use(name, this as any);
      } else {
        passportInstance.use(this as any);
      }
    }
    getPassportInstance() {
      return passport;
    }
  }
  return MixinStrategy;
}
```

ここで行っていることは大きく分けて以下の 4 つです
① 引数 Strategy を継承したクラスを作成して、それを返す。
②validate メソッドを実行する関数を作成している
③ 引数 Strategy のコンストラクタに設定したオプションと ② の関数を渡している
④[passport モジュール](https://github.com/jaredhanson/passport)の use メソッドを使用して、作成したクラスを渡している
これらのことから、まず PassportStrategy 自体は関数だけど戻り値がクラスなので`class JwtStrategy extends PassportStrategy(Strategy)`という形で書けることが分かります。
そして、PassportStrategy 関数は validate メソッドが存在することを前提とした抽象クラスを返すことから、実際に使う時は validate メソッドが必要だと分かります。
引数 Strategy のコンストラクタも呼び出しているので、この PassportStrategy 関数は NestJS と引数 Strategy の機能を繋ぐ中継点のような役割だとも推測できます。
また、passport モジュールとの中継点とも考えられそうです。
PassportStrategy 関数自体の役割はざっくりとつかめたので、この後は引数 Strategy の中身と passport.use が何をしているかを見ていきます。

### 引数 Strategy の役割

当初作成した JwtStrategy を再掲すると、PassportStrategy 関数の引数 Strategy には[passport-jwt モジュール](https://github.com/mikenicholson/passport-jwt)の Strategy を設定していることが分かります。

```tsx
import { Strategy } from 'passport-jwt';
//...略
export class JwtStrategy extends PassportStrategy(Strategy) {
    constructor() {
        super({
            jwtFromRequest://...略,
            secretOrKeyProvider: //...略
        });
    }
    async validate(payload:any){
			//...略
		}
}
```

なので、実際に[Starategy](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L28)を見ると以下のようになっています。

```tsx
function JwtStrategy(options, verify) {
  passport.Strategy.call(this);
  this.name = "jwt";
  this._secretOrKeyProvider = options.secretOrKeyProvider;
  if (options.secretOrKey) {
    if (this._secretOrKeyProvider) {
      throw new TypeError(
        "JwtStrategy has been given both a secretOrKey and a secretOrKeyProvider"
      );
    }
    this._secretOrKeyProvider = function (request, rawJwtToken, done) {
      done(null, options.secretOrKey);
    };
  }
  if (!this._secretOrKeyProvider) {
    throw new TypeError("JwtStrategy requires a secret or key");
  }
  this._verify = verify;
  if (!this._verify) {
    throw new TypeError("JwtStrategy requires a verify callback");
  }
  this._jwtFromRequest = options.jwtFromRequest;
  if (!this._jwtFromRequest) {
    throw new TypeError(
      "JwtStrategy requires a function to retrieve jwt from requests (see option jwtFromRequest)"
    );
  }
  this._passReqToCallback = options.passReqToCallback;
  var jsonWebTokenOptions = options.jsonWebTokenOptions || {};
  //for backwards compatibility, still allowing you to pass
  //audience / issuer / algorithms / ignoreExpiration
  //on the options.
  this._verifOpts = assign({}, jsonWebTokenOptions, {
    audience: options.audience,
    issuer: options.issuer,
    algorithms: options.algorithms,
    ignoreExpiration: !!options.ignoreExpiration,
  });
}
```

ちょくちょく見覚えのあるものも存在します。
ここで行っているのは引数でもらったオプションを自身のプロパティへ代入している処理となっています。
ただ、さっきの流れから関数で書いていることに違和感を感じるとおもいます。
これは、おそらく後の処理を確認する限り、関数ではなくクラスとして記載していると思います。
passport モジュール系は全体的にそうなのですが、class 構文で書かれていないのでなぜ突然関数と戸惑いますが大体がクラスを想定して書かれていそうです。
なので、今回の JwtStrategy も実質以下のようだと認識しても問題なさそうでした。

```tsx
class JwtStrategy {
  constructor() {}
}
```

話を戻すと、引数 Strategy は JwtStrategy であり、PassportStrategy 関数内においてはこの後 JWT 周りの操作するための値を設定していると理解できます。

### passport.use について

passport モジュールのインポート元である index.js を見ると以下の記載があります。

```tsx
// Module dependencies.
var Passport = require("./authenticator"),
  SessionStrategy = require("./strategies/session");
/**
 * Export default singleton.
 *
 * @api public
 */
exports = module.exports = new Passport();
/**
 * Expose constructors.
 */
exports.Passport = exports.Authenticator = Passport;
exports.Strategy = require("passport-strategy");
/*
 * Expose strategies.
 */
exports.strategies = {};
exports.strategies.SessionStrategy = SessionStrategy;
```

特にプロパティを指定せずにインポートした場合は、同階層にある authenticator.js の内容を使用すると考えて良さそうです。
なので、lib/authenticator.js を見て[use メソッド](https://github.com/jaredhanson/passport/blob/master/lib/authenticator.js#L57C1-L66C3)を使用している箇所を探すと以下のコードが見つかります。

```tsx
Authenticator.prototype.use = function (name, strategy) {
  if (!strategy) {
    strategy = name;
    name = strategy.name;
  }
  if (!name) {
    throw new Error("Authentication strategies must have a name");
  }

  this._strategies[name] = strategy;
  return this;
};
```

ここで行っているのは明示的にストラテジー名を引数に渡した場合は、その名前をキーにしていストラテジーを値に設定しています。
一方で、名前が明示的にない場合はストラテジーで指定した name プロパティの値をキーにして、ストラテジーを値に設定しています。
もし名前が存在しない場合はエラーを返すようにしているので、use メソッドを使用する場合は明示的にストラテジーの名前を指定するか、name プロパティを持つオブジェクトを渡す必要があります。
値をプロパティに設定することがわかりましたが、これを行う意味はなんでしょう。
実はこの後 AuthGuard にまつわる章で見ていく、authenticate メソッドを実行する際に\_strategies プロパティに設定したストラテジーを設定した名前で取得する処理があります。
そのため、予めこの use メソッドでストラテジーを登録しておかないと、呼び出したいストラテジーが存在せず処理が走らなくなってしまいます。
以上のことから、PassportStrategy 関数で作成するクラスに passport.use を記載することで予め使用するストラテジーを登録していると分かります。
ここまでで、大本であった PassportStrategy 関数の中で何をしているかを見てきました。
次からは呼び出す時のコンストラクタで設定した jwtFromRequest プロパティと secretOrKeyProvider プロパティの役割を見ていきます。
まずは、jwtFromRequest プロパティです。

## jwtFromRequest プロパティの役割

jwtFromRequest プロパティは先程の話で、[passport-jwt モジュール](https://github.com/mikenicholson/passport-jwt/tree/master)の内容だと分かりました。
なので、passport-jwt モジュールの[README](https://github.com/mikenicholson/passport-jwt/tree/master#configure-strategy)を確認すると以下の記載があります。

> `jwtFromRequest` (REQUIRED) Function that accepts a request as the only parameter and returns either the JWT as a string or *null*.

Request パラメータを受け取り、JWT 形式の文字列もしくは null を返す関数を設定するプロパティだと分かります。
戻り値はわかりましたが、Request パラメータが何を指しているのかが気になります。
そこで、passport-jwt モジュールの型定義がまとめてある、[types/passport-jwt/index.d.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/passport-jwt/index.d.ts)を確認すると以下の記載があります。

```tsx
export interface JwtFromRequestFunction {
  (req: express.Request): string | null;
}
```

上記からわかるように、引数は Express の Request オブジェクトとなっています。
以上から、jwtFromRequest プロパティは受け取った Express のリクエストから JWT 形式のトークンを抽出した後、そのトークンを返す関数を設定するものだと分かります。

## secretOrKeyProvider の役割

まず jwtFromRequest プロパティと同様に secretOrKeyProvider プロパティの内容を[README](https://github.com/mikenicholson/passport-jwt/tree/master#configure-strategy)で確認すると、以下の記載があります。

> - `secretOrKey` is a string or buffer containing the secret (symmetric) or PEM-encoded public key (asymmetric) for verifying the token's signature. REQUIRED unless `secretOrKeyProvider` is provided.
> - `secretOrKeyProvider` is a callback in the format `function secretOrKeyProvider(request, rawJwtToken, done)`, which should call `done` with a secret or PEM-encoded public key (asymmetric) for the given key and request combination. `done` accepts arguments in the format `function done(err, secret)`. Note it is up to the implementer to decode rawJwtToken. REQUIRED unless `secretOrKey` is provided.

secretOrKey はトークンの署名を検証する鍵情報を渡すプロパティとなっています。
一方で、secretOrKeyProvider は署名を検証する鍵情報を引数に渡すコールバックを定義する関数を渡すプロパティとなっています。
どちらか一方のプロパティは必要ですが、両方のプロパティがあるとエラーが発生するのでご注意ください。([参考資料](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L35))
secretOrKey プロパティと secretOrKeyProvider プロパティの使い分けですが、鍵情報の文字列や鍵ファイルが手元にあるなら secretOrKey プロパティで十分です。
secretOrKeyProvider プロパティは、鍵情報の文字列や鍵ファイルが手元になく別途取得する処理が必要となる場合使用します。
役割はこのように鍵情報を渡すための設定をするプロパティとなっていますが、secretOrKeyProvider プロパティを使用する場合一点注意点があります。
それは定義する関数内で必ずコールバック関数を呼び出す処理にする必要があります。
secretOrKeyProvider プロパティの型定義を[types/passport-jwt/index.d.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/passport-jwt/index.d.ts)で確認すると、以下の通りです。

```tsx
export interface SecretOrKeyProvider {
  (
    request: express.Request,
    rawJwtToken: any,
    done: (err: any, secretOrKey?: string | Buffer) => void
  ): void;
}
```

このように第 3 引数にコールバック関数が存在します。
そのため、secretOrKeyProvider プロパティに設定する関数は以下のコードのように、必ずコールバックを呼び出すような定義にしてください。

```tsx
const secretOrKeyProvider: SecretOrKeyProvider = (
  request: express.Request,
  rawJwtToken: any,
  done: (err: any, secretOrKey?: string | Buffer) => void
) => {
  // ここにカスタムの実装を追加
  // この関数は要求、JWTトークンからシークレットまたはキーを取得し、
  // doneコールバックを呼び出して結果を返す必要があります。
  done(null, "your-secret-or-key");
};
```

コールバックを呼び出さないと、ストラテジーを実行する際に後続の処理が走らなくなり、検証が終わらなくなってしまうからです。
実際の処理についてなどはこの後の AuthGuard の方で確認するので、一旦は鍵情報を取得するためのプロパティで、別途鍵情報を取得するための処理を行ったら、鍵情報を渡しつつコールバックを実行するようにするということを把握していただければ幸いです。

# AuthGuard は何をしているかを見る

ここでは作成した Strategy を動かすために使用している下記コードの AuthGuard から JwtStrategy がどのように JWT の検証を行っているか見ていきます。

```tsx
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

まず、[AuthGuard のコード](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L37)を見ると下記のようになっています。

```tsx
export const AuthGuard: (type?: string | string[]) => Type<IAuthGuard> =
  memoize(createAuthGuard);
```

[memoize 関数](https://github.com/nestjs/passport/blob/master/lib/utils/memoize.util.ts)はデータをオブジェクトに格納する処理だけなので、本筋である[createAuthGuard](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L43)を確認すると下記のようになります。

```tsx
function createAuthGuard(type?: string | string[]): Type<IAuthGuard> {
  class MixinAuthGuard<TUser = any> implements CanActivate {
    @Optional()
    @Inject(AuthModuleOptions)
    protected options: AuthModuleOptions = {};
    constructor(@Optional() options?: AuthModuleOptions) {
      this.options = options ?? this.options;
      if (!type && !this.options.defaultStrategy) {
        authLogger.error(NO_STRATEGY_ERROR);
      }
    }
    async canActivate(context: ExecutionContext): Promise<boolean> {
      const options = {
        ...defaultOptions,
        ...this.options,
        ...(await this.getAuthenticateOptions(context)),
      };
      const [request, response] = [
        this.getRequest(context),
        this.getResponse(context),
      ];
      const passportFn = createPassportContext(request, response);
      const user = await passportFn(
        type || this.options.defaultStrategy,
        options,
        (err, user, info, status) =>
          this.handleRequest(err, user, info, context, status)
      );
      request[options.property || defaultOptions.property] = user;
      return true;
    }
    getRequest<T = any>(context: ExecutionContext): T {
      return context.switchToHttp().getRequest();
    }
    getResponse<T = any>(context: ExecutionContext): T {
      return context.switchToHttp().getResponse();
    }
    async logIn<TRequest extends { logIn: Function } = any>(
      request: TRequest
    ): Promise<void> {
      const user = request[this.options.property || defaultOptions.property];
      await new Promise<void>((resolve, reject) =>
        request.logIn(user, this.options, (err) =>
          err ? reject(err) : resolve()
        )
      );
    }
    handleRequest(err, user, info, context, status): TUser {
      if (err || !user) {
        throw err || new UnauthorizedException();
      }
      return user;
    }
    getAuthenticateOptions(
      context: ExecutionContext
    ): Promise<IAuthModuleOptions> | IAuthModuleOptions | undefined {
      return undefined;
    }
  }
  const guard = mixin(MixinAuthGuard);
  return guard as Type<IAuthGuard>;
}
```

コードが多いですが、やりたいことはガード用に使用する CanActivate インターフェースを使ったクラスを作成することです。
今回注目しているのは JWT の検証なので、実際見るのは下記の canActive メソッドだけで良さそうです。

```tsx
async canActivate(context: ExecutionContext): Promise<boolean> {
      const options = {
        ...defaultOptions,
        ...this.options,
        ...(await this.getAuthenticateOptions(context))
      };
      const [request, response] = [
        this.getRequest(context),
        this.getResponse(context)
      ];
      const passportFn = createPassportContext(request, response);
      const user = await passportFn(
        type || this.options.defaultStrategy,
        options,
        (err, user, info, status) =>
          this.handleRequest(err, user, info, context, status)
      );
      request[options.property || defaultOptions.property] = user;
      return true;
    }
```

こちらも色々書いていますが、着目するのは createPassportContext 関数だけです。
それ以外は値を取得するための処理で、検証に関わる操作は行っていなさそうです。
なので、[createPassportContext 関数](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L113)のコードを見ると以下のようになっています。

```tsx
const createPassportContext =
  (request, response) => (type, options, callback: Function) =>
    new Promise<void>((resolve, reject) =>
      passport.authenticate(type, options, (err, user, info, status) => {
        try {
          request.authInfo = info;
          return resolve(callback(err, user, info, status));
        } catch (err) {
          reject(err);
        }
      })(request, response, (err) => (err ? reject(err) : resolve()))
    );
```

ここで passport モジュールの処理である[authenticate メソッド](https://github.com/jaredhanson/passport/blob/master/lib/authenticator.js#L171)が出てきました。
中身を見ると以下のコードがあります。

```tsx
Authenticator.prototype.authenticate = function (strategy, options, callback) {
  return this._framework.authenticate(this, strategy, options, callback);
};
```

\_framework プロパティにある authenticate メソッドを読んでいるそうです。
では、\_framework プロパティを設定している[framework メソッド](https://github.com/jaredhanson/passport/blob/master/lib/authenticator.js#L98)を確認します。

```tsx
Authenticator.prototype.framework = function (fw) {
  this._framework = fw;
  return this;
};
```

引数に設定した関数をそのまま格納していますね。
最後に framework メソッドを実行している init メソッドと各メソッドを持つ[Authenticator オブジェクト](https://github.com/jaredhanson/passport/blob/master/lib/authenticator.js#L6C1-L37C3)のコードを確認します。

```tsx
/**
 * Create a new `Authenticator` object.
 *
 * @public
 * @class
 */
function Authenticator() {
  this._key = "passport";
  this._strategies = {};
  this._serializers = [];
  this._deserializers = [];
  this._infoTransformers = [];
  this._framework = null;

  this.init();
}
/**
 * Initialize authenticator.
 *
 * Initializes the `Authenticator` instance by creating the default `{@link SessionManager}`,
 * {@link Authenticator#use `use()`}'ing the default `{@link SessionStrategy}`, and
 * adapting it to work as {@link https://github.com/senchalabs/connect#readme Connect}-style
 * middleware, which is also compatible with {@link https://expressjs.com/ Express}.
 *
 * @private
 */
Authenticator.prototype.init = function () {
  this.framework(require("./framework/connect")());
  this.use(
    new SessionStrategy({ key: this._key }, this.deserializeUser.bind(this))
  );
  this._sm = new SessionManager(
    { key: this._key },
    this.serializeUser.bind(this)
  );
};
```

init 関数では引数に[lib/framework/connect.js](https://github.com/jaredhanson/passport/blob/master/lib/framework/connect.js)から取得したものを入れて framework メソッドを実行しています。
すなわち、init メソッドが実行されたタイミングで framework メソッドが実行され authenticate メソッドで使用する処理の準備が完了します。
そして、init メソッドは`Authenticator()`の中で実行するように定義しているので、passport オブジェクトがインスタンス化した時点で実行は完了しています。
以上のことから、authenticate メソッドの内部処理は[lib/framework/connect.js](https://github.com/jaredhanson/passport/blob/master/lib/framework/connect.js)を見れば良さそうです。
実際に見ると以下のように、lib/middleware/authenticate.js をインポートしてい、authenticate プロパティに設定しています。

```tsx
/**
 * Module dependencies.
 */
var initialize = require("../middleware/initialize"),
  authenticate = require("../middleware/authenticate");

/**
 * Framework support for Connect/Express.
 *
 * This module provides support for using Passport with Express.  It exposes
 * middleware that conform to the `fn(req, res, next)` signature.
 *
 * @return {Object}
 * @api protected
 */
exports = module.exports = function () {
  return {
    initialize: initialize,
    authenticate: authenticate,
  };
};
```

ようやく確認すべき箇所がわかったので、実際に[lib/middleware/authenticate.js](https://github.com/jaredhanson/passport/blob/master/lib/middleware/authenticate.js#L70)の中身を見てみます。
なお、今回の話にあまり関係ない部分もあるので、中身を全部見ることはしません。
そのため、注目したい部分は確認する方式を取ります。
まず戻り値についてです。
紛らわしいですが、以下の authenticate 関数を返します。

```tsx
return function authenticate(req, res, next) {
  //...略
};
```

そのため、createPassportContext 関数では passport モジュールの authenticate メソッドを実行したら後続の処理を行うようにしたく、カッコで繋げています。
次に戻り値である authenticate 関数が実行される[以下の関数](https://github.com/jaredhanson/passport/blob/master/lib/middleware/authenticate.js#L180C5-L370C11)内の処理についてです。

```tsx
(function attempt(i) {})(0);
```

この関数内で着目するのは以下の部分です。

```tsx
strategy.authenticate(req, options);
```

strategy は以下のように定義しています。

```tsx
var layer = name[i];
// If no more strategies exist in the chain, authentication has failed.
if (!layer) {
  return allFailed();
}

// Get the strategy, which will be used as prototype from which to create
// a new instance.  Action functions will then be bound to the strategy
// within the context of the HTTP request/response pair.
var strategy, prototype;
if (typeof layer.authenticate == "function") {
  strategy = layer;
} else {
  prototype = passport._strategy(layer);
  if (!prototype) {
    return next(new Error('Unknown authentication strategy "' + layer + '"'));
  }

  strategy = Object.create(prototype);
}
```

変数 layer に authenticate プロパティが存在し、それが関数であれば目的の値であるオブジェクトなのでそのまま格納します。
そうでない場合は passport モジュールが提供する[\_strategy メソッド](https://github.com/jaredhanson/passport/blob/cfdbd4a762b51e339ebfea931d65bccbbde53282/lib/authenticator.js#L478)を使用して passport モジュールの[use メソッド](https://github.com/jaredhanson/passport/blob/cfdbd4a762b51e339ebfea931d65bccbbde53282/lib/authenticator.js#L57)で登録したオブジェクトを新たに作成して変数に格納しています。
ちなみに、ここでは詳細な説明はしませんが今回の場合、上記で言及したオブジェクトは[JwtStrategy](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js)となります。
[JwtStrategy](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js)は authenticate プロパティを持っているので、後の処理を問題なく実行できます。
変数 strategy の中身がわかったので、strategy オブジェクトが実行する[authenticate プロパティ](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L90)の中身を見ていきましょう。

```tsx
JwtStrategy.prototype.authenticate = function (req, options) {
  var self = this;
  var token = self._jwtFromRequest(req);
  if (!token) {
    return self.fail(new Error("No auth token"));
  }
  this._secretOrKeyProvider(
    req,
    token,
    function (secretOrKeyError, secretOrKey) {
      if (secretOrKeyError) {
        self.fail(secretOrKeyError);
      } else {
        // Verify the JWT
        JwtStrategy.JwtVerifier(
          token,
          secretOrKey,
          self._verifOpts,
          function (jwt_err, payload) {
            if (jwt_err) {
              return self.fail(jwt_err);
            } else {
              // Pass the parsed token to the user
              var verified = function (err, user, info) {
                if (err) {
                  return self.error(err);
                } else if (!user) {
                  return self.fail(info);
                } else {
                  return self.success(user, info);
                }
              };
              try {
                if (self._passReqToCallback) {
                  self._verify(req, payload, verified);
                } else {
                  self._verify(payload, verified);
                }
              } catch (ex) {
                self.error(ex);
              }
            }
          }
        );
      }
    }
  );
};
```

まず、`self._jwtFromRequest(req)`を実行し、NestJS で JwtStrategy クラスを作る際にみた、JWT 形式のトークンを取得する処理を実行し、トークンを取得しています。
ここでトークンが無ければ、例外を発生させます。
その後、鍵情報を引数に設定した this.\_secretOrKeyProvider を実行し、コールバック関数も定義しています。
ちなみに passport-jwt モジュールの JwtStrategy 内にはなかった`JwtStrategy.JwtVerifier`ですが、[strategy.js の 83 行目](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L83)くらいで以下のように設定しています。

```tsx
JwtStrategy.JwtVerifier = require("./verify_jwt");
```

[verify_jwt.js](https://github.com/mikenicholson/passport-jwt/blob/master/lib/verify_jwt.js)の処理を見ると、jsonwebtoken の verify メソッドでトークンの検証を行っていることが分かります。

```tsx
var jwt = require("jsonwebtoken");
module.exports = function (token, secretOrKey, options, callback) {
  return jwt.verify(token, secretOrKey, options, callback);
};
```

以上のことから、[authenticate プロパティ](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L90)はまず JWS 形式のデータを取得した後、データが存在すれば署名の検証を行い、後続の処理を実行するものだと分かります。
今回でいうと、後続の処理は NestJS で JwtStrategy クラスを作る際に確認した validate メソッドとなります。
ここまで見れば、[createPassportContext 関数](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L113)がどんな関数を返すのかイメージが湧いてきたと思います。
すなわち、トークンを取得し、トークンの署名を検証し、定義した validate メソッドを実行させる関数を返すのが[createPassportContext 関数](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L113)の役割となります。
なので、この AuthGuard が呼ばれて初めて作成した JwtStrategy の処理が走ることとなります。
以上をまとめると AuthGuard は以下の流れで実行されます。
①[passport モジュールの authenticate 関数](https://github.com/jaredhanson/passport/blob/master/lib/middleware/authenticate.js#L70)実行
②passport-jwt の JwtStrategy で設定した[authenticate プロパティ](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L90C1-L90C12)の関数実行
③[jwt.strategy.ts のコンストラクタで設定した jwtFromRequest のメソッド実行](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L93)
④[jwt.strategy.ts で設定した secretOrKeyProvider のメソッド実行してトークンの検証](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L99)
⑤jwt.strategy.ts で設定した validate メソッド実行
⑥ 特にオプションを設定しなければ[passport の option.ts で定義した user というプロパティ名](https://github.com/nestjs/passport/blob/master/lib/options.ts)を使い、[リクエスト内の user プロパティに validate の値を渡すようにする](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L73)
⑦ ガードは通ったので true を返し、ガード処理を終える
なお、AuthGuard に存在するメソッドはカスタマイズすることも可能です。
ドキュメントでは以下のように記載したりと、[createAuthGuard](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L43)関数内で定義されているメソッドに引数や戻り値を設定すれば、自由にカスタマイズすることができます。

```tsx
@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {
  canActivate(context: ExecutionContext) {
    // Add your custom authentication logic here
    // for example, call super.logIn(request) to establish a session.
    return super.canActivate(context);
  }
  handleRequest(err, user, info) {
    // You can throw an exception based on either "info" or "err" arguments
    if (err || !user) {
      throw err || new UnauthorizedException();
    }
    return user;
  }
}
```

# AuthGuard を呼び出す時の文字列って何？

先程 AuthGuard の内部機能について説明をしましたが、実際 AuthGuard を呼び出す時例えば以下のように引数に「jwt」という文字列を指定していました。

```tsx
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

なぜわざわざ文字列を指定するのか疑問に感じますね。
そこで、ここでは AuthGuard を呼び出す際に設定している文字列が何かを見ていきます。
実はすでに答えは出てきています。
それは[lib/middleware/authenticate.js](https://github.com/jaredhanson/passport/blob/master/lib/middleware/authenticate.js#L70)でストラテジーを取得するときに行っていた以下のコードにあります。

```tsx
var layer = name[i];
// If no more strategies exist in the chain, authentication has failed.
if (!layer) {
  return allFailed();
}

// Get the strategy, which will be used as prototype from which to create
// a new instance.  Action functions will then be bound to the strategy
// within the context of the HTTP request/response pair.
var strategy, prototype;
if (typeof layer.authenticate == "function") {
  strategy = layer;
} else {
  prototype = passport._strategy(layer);
  if (!prototype) {
    return next(new Error('Unknown authentication strategy "' + layer + '"'));
  }

  strategy = Object.create(prototype);
}
```

まず注目するのは変数 name です。
変数 name は巡りめぐって AuthGuard を呼び出す時に設定した文字列を配列にしたものとなっています。
なぜそうなるかの詳細をここでは説明しませんが、気になる方は[createAuthGuard 関数](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L43)から[createPassportContext 関数](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L113)を[実行しているコード付近](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L66)を確認してもらい、lib/authenticator.js 内の[authenticate メソッド](https://github.com/jaredhanson/passport/blob/master/lib/authenticator.js#L171)を見たら、内部の処理を行っている[lib/middleware/authenticate.js の authenticate 関数](https://github.com/jaredhanson/passport/blob/master/lib/middleware/authenticate.js#L70)の[89 行目付近](https://github.com/jaredhanson/passport/blob/master/lib/middleware/authenticate.js#L89)を確認してください。
そのように辿って行けば、が格納されているか分かると思います。
変数 name の説明は以上で、次に注目するのは[\_strategy メソッド](https://github.com/jaredhanson/passport/blob/cfdbd4a762b51e339ebfea931d65bccbbde53282/lib/authenticator.js#L478)部分です。
中を見ると以下のように実装されています。

```tsx
Authenticator.prototype._strategy = function (name) {
  return this._strategies[name];
};
```

\_strategies プロパティも何だか見覚えがありますね。
\_strategies プロパティは passport モジュールの[use メソッド](https://github.com/jaredhanson/passport/blob/cfdbd4a762b51e339ebfea931d65bccbbde53282/lib/authenticator.js#L57)でキーにストラテジーの名前を、値に使用するストラテジーを格納したものでした。
そのため、[\_strategy メソッド](https://github.com/jaredhanson/passport/blob/cfdbd4a762b51e339ebfea931d65bccbbde53282/lib/authenticator.js#L478)は使用するストラテジーを取得するための処理だと分かります。
以上のことから、AuthGuard の引数に設定した文字列の正体が分かります。
その正体はガードで使用するストラテジーを特定するためのキーです。
正体は分かったのですが、「jwt」という文字列はいつ登録されたのでしょう。
それは Strategy の[31 行目](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js#L31)をみると分かります。

```tsx
this.name = "jwt";
```

このように name プロパティに「jwt」という文字列を格納しています。
そして、思い出してください。
passport モジュールの[use メソッド](https://github.com/jaredhanson/passport/blob/cfdbd4a762b51e339ebfea931d65bccbbde53282/lib/authenticator.js#L57)は名前を指定しなかった場合、ストラテジーの name プロパティの値をキーとして設定します。
よって、何も名前を設定しなかった場合今回見てきたストラテジーは「jwt」という文字列をキーにして、保存されることとなります。
なので、AuthGuard の引数に「jwt」を指定すれば動くことが分かりました。
なお、名前を設定しなければ「jwt」という文字列で保存されると言いましたが、裏を返せば独自の名前で定義することもできます。
実際[PassportStrategy](https://github.com/nestjs/passport/blob/master/lib/passport/passport.strategy.ts)の第二引数には以下のように名前を定義することができます。

```tsx
export function PassportStrategy<T extends Type<any> = any>(
  Strategy: T,
  name?: string | undefined,
  callbackArity?: true | number
): {
  new (...args): InstanceType<T>;
};
```

そして、passport.use 部分はもし引数 name が存在すれば、その名前を設定するようにしています。
以上から、仮に JwtStrategy を作成する際に以下のように作成すると、

```tsx
export class JwtStrategy extends PassportStrategy(
  Strategy,
  "custom_strategy"
) {}
```

AuthGuard を呼び出す際は次のようにすれば、作成したストラテジーを使うようにできます。

```tsx
export class JwtAuthGuard extends AuthGuard("custom_strategy") {}
```

中々面白いですね。
この機能を使えば環境ごとに呼び出すガードを変更するなども出来そうですね。
ついでにもう一点補足で、AuthGuard は以下のように配列でストラテジー名を指定できます。

```tsx
export class JwtAuthGuard extends AuthGuard(['strategy_jwt_1', 'strategy_jwt_2', '...']) { ... }
```

こうすれば、どれか一つのストラテジーの処理が完了すれば、エンドポイントを実行するようになります。
そのため、複数の形で来るリクエストに対しても問題なく検証が出来たりとカスタマイズができます。

# JwtStrategy をカスタマイズする

ここまでで JwtStrategy の機能について見てきました。
機能を確認することで、JwtStrategy も色々要件に合わせた拡張が出来そうだと感じますね。
そこで、この章は JwtStrategy を拡張させ、様々な方法でトークンが作成・保存されていても対応できるようにカスタマイズしていきます。

## 要件

今回は各トークンを以下のように想定しています。
①Cookie に保存しているトークンは自前で作成したトークンです。
②Authorization ヘッダーにあるトークンは Auth0 から取得したトークンです。
そのため、Cookie のトークンを取得したときは自前の鍵で検証を行い、Authorization ヘッダーのトークンを取得したときは Auth0 から鍵を取得して検証を行います。

## 完成したコード

先に jwt.strategy.ts 全体のコードを展開します。

```tsx
import { ExtractJwt, SecretOrKeyProvider, Strategy } from "passport-jwt";
import { PassportStrategy } from "@nestjs/passport";
import { Injectable } from "@nestjs/common";
import { Request } from "express";
import { passportJwtSecret } from "jwks-rsa";
const secretOrKeyProvider: SecretOrKeyProvider = (
  request,
  rawJwtToken,
  done
) => {
  if (request.cookies?.token) {
    return done(null, "自前で作成した検証用の鍵");
  }
  return passportJwtSecret({ jwksUri: "url" })(request, rawJwtToken, done);
};
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, "jwt") {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromExtractors([
        (req) =>
          req.cookies.token ?? ExtractJwt.fromAuthHeaderAsBearerToken()(req),
      ]),
      ignoreExpiration: false,
      secretOrKeyProvider: secretOrKeyProvider,
    });
  }
  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

ポイントは大きく二つです。
一つ目は JWT を取得する処理に`ExtractJwt.fromExtractors`を使用して、JWT トークンを取得するための関数を独自に定義できるようにします。
内部の処理としては以下のことを行っています。
①Cookie にトークンがあればそこから取得した値を返します。
② 無ければ、Authorization ヘッダーにあるトークンを返します。
二つ目は secretOrKeyProvider に設定する値を自前で作成しているところです。
Cookie にトークンがある場合は自身のプロジェクト内にある鍵で検証を行い、ない場合は JWKS(JSON Web Key Set)から鍵を取得して検証するようにしています。
JWKS から鍵を取得する処理は passportJwtSecret 関数が行ってくれます。
passportJwtSecret 関数 JWKS の URI を設定すれば、鍵を取得してくれます。
そして、その鍵を使いつつ SecretOrKeyProvider 型のコールバック関数の形式に沿った関数を返してくれます。
そして、もらった関数を実行するようにしています。
なお、Auth0 から JWKS を取得する場合は[ドキュメント](https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-key-sets)に記載があるように、`https://{Auth0のテナントドメイン}/.well-known/jwks.json`を URI に指定すれば取得できます。

## 動作確認

まずガードを以下のように作成します。

```tsx
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

JWT を作成して、Cookie に設定するメソッドを用意します。
NestJS で Cookie を使えるように別途[NestJS のドキュメント](https://docs.nestjs.com/techniques/cookies)をもとに用意していることはご留意ください。

```tsx
@Get('create/jswt')
    async setJwt(@Res({ passthrough: true }) res: Response) {
        const payload = { sub: 'userId', username: 'username' };
        const token = await this.jwtService.signAsync(payload)
        res.cookie('token', token)
    }
```

JwtService クラス呼び出すためのモジュールの設定です。
今回は以下のリンクを参考にして公開鍵方式で作成しており、その情報を設定します。
[https://weblabo.oscasierra.net/openssl-genrsa-secret-1/](https://weblabo.oscasierra.net/openssl-genrsa-secret-1/)
[https://weblabo.oscasierra.net/openssl-genrsa-public-1/](https://weblabo.oscasierra.net/openssl-genrsa-public-1/)
これによって、signAsync メソッドを呼び出せば自動的に設定した鍵を使い署名をしてくれます。

```tsx
imports: [JwtModule.register({
    secret: "鍵情報(RS256の時は秘密鍵を指定)",
		signOptions: {  algorithm: 'HS256による暗号でない場合明示的に設定する' },
  }),],
```

なお、先程示した JwtStrategy の方には公開鍵の情報をセットしています。
動作確認用のメソッドです。
トークンの検証が OK の場合は「test」という文字列が返ってきます。

```tsx
@Get('jwt-strategy-test')
    @UseGuards(JwtAuthGuard)
    async testFn() {
        return 'test'
    }
```

ここまで出来たら curl コマンドで Cookie にトークンをセットした場合と、Authorization Header に Auth0 発行の Bearer トークンが存在するときのみ「test」という文字列が返ってくることが確認できます。

# PassportStrategy 関数をカスタマイズする

最後にストラテジーをラップする PassportStrategy 関数についても、少しカスタマイズします。
PassportStrategy 関数は非常に便利なものですが、以下の欠点も存在します。

- コンストラクタでのオプション設定の際型補完が効かない
- secretOrKey,secretOrKeyProvider どちらを設定すればよいかが、内部の処理を把握していないと分かりにくい。
- secretOrKey,secretOrKeyProvider どちらもオプショナルのため、誤って両方設定しても静的なエラーが発生しない
- validate メソッドを実装しなくても静的なエラーが発生しない。

そこで、ここでは PassportStrategy 関数をさらにラップして、機能を限定することで内部の構造を知らなくても使い易くしていきます。

## 完成したコード

完成したコードは以下のようになっています。

```tsx
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy, StrategyOptions } from "passport-jwt";
interface AbstractStrategyClass {
  validate(...payload: any): any;
}
type KeyOptions = Omit<
  StrategyOptions,
  "secretOrKeyProvider" | "jwtFromRequest"
> &
  Required<Pick<StrategyOptions, "secretOrKey">>;
type KeyProviderOptions = Omit<
  StrategyOptions,
  "secretOrKey" | "jwtFromRequest"
> &
  Required<Pick<StrategyOptions, "secretOrKeyProvider">>;
type JwtFromRequest = Pick<StrategyOptions, "jwtFromRequest">;
const COOKIE_STRATEGY_NAME_WITH_KEY = "COOKIE_STRATEGY_NAME_WITH_KEY";
export const CookieStrategyWithKey = {
  strategyName: COOKIE_STRATEGY_NAME_WITH_KEY,
  strategy: (options: KeyOptions, cookieName: string) => {
    const getTokenObject: JwtFromRequest = {
      jwtFromRequest: (req) => req?.cookies[cookieName] ?? null,
    };
    abstract class CookieStrategyClassWithKey
      extends PassportStrategy(Strategy, COOKIE_STRATEGY_NAME_WITH_KEY)
      implements AbstractStrategyClass
    {
      constructor() {
        super({ ...options, ...getTokenObject });
      }
      abstract validate(...payload: any): any;
    }
    return CookieStrategyClassWithKey;
  },
};
CookieStrategyWithKey.strategy({ secretOrKey: "" }, "token");
const COOKIE_STRATEGY_NAME_WITH_KEY_PROVIDER =
  "COOKIE_STRATEGY_NAME_WITH_KEY_PROVIDER";
export const CookieStrategyWithKeyProvider = {
  strategyName: COOKIE_STRATEGY_NAME_WITH_KEY_PROVIDER,
  strategy: (options: KeyProviderOptions, cookieName: string) => {
    const getTokenObject: JwtFromRequest = {
      jwtFromRequest: (req) => req?.cookies[cookieName] ?? null,
    };
    abstract class CookieStrategyClassWithKey
      extends PassportStrategy(Strategy, COOKIE_STRATEGY_NAME_WITH_KEY_PROVIDER)
      implements AbstractStrategyClass
    {
      constructor() {
        super({ ...options, ...getTokenObject });
      }
      abstract validate(...payload: any): any;
    }
    return CookieStrategyClassWithKey;
  },
};
const AUTHORIZATION_HEADER_STRATEGY_NAME_WITH_KEY =
  "AUTHORIZATION_HEADER_STRATEGY_NAME_WITH_KEY";
export const AuthorizationHeaderStrategyWithKey = {
  strategyName: AUTHORIZATION_HEADER_STRATEGY_NAME_WITH_KEY,
  strategy: (options: KeyOptions) => {
    const getTokenObject: JwtFromRequest = {
      jwtFromRequest: (req) => ExtractJwt.fromAuthHeaderAsBearerToken()(req),
    };
    abstract class CookieStrategyClassWithKey
      extends PassportStrategy(
        Strategy,
        AUTHORIZATION_HEADER_STRATEGY_NAME_WITH_KEY
      )
      implements AbstractStrategyClass
    {
      constructor() {
        super({ ...options, ...getTokenObject });
      }
      abstract validate(...payload: any): any;
    }
    return CookieStrategyClassWithKey;
  },
};
const AUTHORIZATION_HEADER_STRATEGY_NAME_WITH_KEY_PROVIDER =
  "AUTHORIZATION_HEADER_STRATEGY_NAME_WITH_KEY_PROVIDER";
export const AuthorizationHeaderStrategyWithKeyProvider = {
  strategyName: AUTHORIZATION_HEADER_STRATEGY_NAME_WITH_KEY_PROVIDER,
  strategy: (options: KeyProviderOptions) => {
    const getTokenObject: JwtFromRequest = {
      jwtFromRequest: (req) => ExtractJwt.fromAuthHeaderAsBearerToken()(req),
    };
    abstract class CookieStrategyClassWithKey
      extends PassportStrategy(
        Strategy,
        AUTHORIZATION_HEADER_STRATEGY_NAME_WITH_KEY_PROVIDER
      )
      implements AbstractStrategyClass
    {
      constructor() {
        super({ ...options, ...getTokenObject });
      }
      abstract validate(...payload: any): any;
    }
    return CookieStrategyClassWithKey;
  },
};
```

記載したコードは以下のケースにおいて、JWT の検証ができるようになっています。

- Cookie に JWT 形式のトークンが存在しており、検証用の鍵が手元にある
- Cookie に JWT 形式のトークンが存在しており、別途鍵を取得する処理が必要
- Authorization ヘッダーに Bearer トークンが存在しており、検証用の鍵が手元にある
- Authorization ヘッダーに Bearer トークンが存在しており、別途鍵を取得する処理が必要

このファイルで出来ることを確認したので、コードにおけるポイントを見ていきます。
まず下記のインターフェースを設定することで、ストラテジーを作成する際に validate メソッドがないとエラーを発生させるようにしました。

```tsx
interface AbstractStrategyClass {
  validate(...payload: any): any;
}
```

引数をスプレッドにしているのは passReqToCallback プロパティが true の時、第一引数に Request オブジェクトが設定される際の対応をするためです。
次に以下の型定義を行うことで、自前の鍵がある場合と別途鍵情報を取得が必要な場合で設定するオプションが異なるようにしました。
また、今回はトークンを取得する方法も限定したいので、トークンを取得するためのオブジェクトを別途作成できるよう型を定義しました。

```tsx
type KeyOptions = Omit<
  StrategyOptions,
  "secretOrKeyProvider" | "jwtFromRequest"
> &
  Required<Pick<StrategyOptions, "secretOrKey">>;
type KeyProviderOptions = Omit<
  StrategyOptions,
  "secretOrKey" | "jwtFromRequest"
> &
  Required<Pick<StrategyOptions, "secretOrKeyProvider">>;
type JwtFromRequest = Pick<StrategyOptions, "jwtFromRequest">;
```

この型を使うことで、呼び出す時のオプションを最小限にすることができます。
ここまでで準備が完了したので、PassportStrategy 関数を拡張したコードを確認します。

```bash
const COOKIE_STRATEGY_NAME_WITH_KEY = 'COOKIE_STRATEGY_NAME_WITH_KEY'
export const CookieStrategyWithKey = {
    strategyName: COOKIE_STRATEGY_NAME_WITH_KEY,
    strategy: (options: KeyOptions, cookieName: string) => {
        const getTokenObject: JwtFromRequest = {
            jwtFromRequest: (req) => req?.cookies[cookieName] ?? null
        }
        abstract class CookieStrategyClassWithKey extends PassportStrategy(Strategy, COOKIE_STRATEGY_NAME_WITH_KEY) implements AbstractStrategyClass {
            constructor() {
                super({ ...options, ...getTokenObject })
            }
            abstract validate(...payload: any): any
        }
        return CookieStrategyClassWithKey;
    }
}
```

このオブジェクトは自前の鍵で署名したトークンを検証するための情報をまとめたオブジェクトです。
そして、定義したオブジェクトの各プロパティは以下の想定で定義しています。
strategyName: 定義した時のストラテジーを呼び出すための名前
strategy: JwtStrategy をもとにした抽象クラスを取得する関数
strategy プロパティの中身をもう少し見ていきます。

```bash
strategy: (options: KeyOptions, cookieName: string) => {
        const getTokenObject: JwtFromRequest = {
            jwtFromRequest: (req) => req?.cookies[cookieName] ?? null
        }
        abstract class CookieStrategyClassWithKey extends PassportStrategy(Strategy, COOKIE_STRATEGY_NAME_WITH_KEY) implements AbstractStrategyClass {
            constructor() {
                super({ ...options, ...getTokenObject })
            }
            abstract validate(...payload: any): any
        }
        return CookieStrategyClassWithKey;
    }
```

まず引数には先程作成した KeyOptions 型を設定しています。
これによって、このオブジェクトを使用する場合 secretOrKey プロパティでストラテジーを動かすことができます。
引数の後には、先程定義した JwtFromRequest 型を使用して、JWT を取得するオブジェクトを定義しています。
JwtFromRequest 型にある jwtFromRequest プロパティの型は以下のように、Request オブジェクトを引数にして、文字列か null を返す関数となっています。

```bash
export interface JwtFromRequestFunction {
    (req: express.Request): string | null;
}
```

そして、今見ている CookieStrategyWithKey オブジェクトは Cookie からトークンを取得するので Request オブジェクトの Cookie から指定のトークンを取るようにして、無ければ null を返すようにしています。
なお、ここで取得する Cookie 名を指定することはできないので、strategy プロパティで定義した関数の引数に取得する Cookie 名を設定しています。
Authorization Header から取得する場合は、トークンの形式が定まっているため引数に設定していません。
最後に AbstractStrategyClass インターフェースを使った抽象クラスを作成しています。
基本的にはこれまで見てきた PassportStrategy 関数を使った JwtStrategy クラスの作成の抽象化版です。
ただ、コンストラクタに設定する値は StrategyOptions 型のプロパティが存在しないといけないので、先程作成トークンを取得するオブジェクトと引数のオブジェクトをスプレッドで結合したものを渡しています。
以上が実際に使用するオブジェクトの説明となります。
secretOrKeyProvider を使用するオブジェクトや Authorization Header からトークンを取得するオブジェクトなどで、引数の有無や使用する型の違いなど細かい違いはあります。
ただ、別途解説するほどでもないので、コードの中身については以上とします。
実際に使用する場合は以下のように呼び出せばこれまでと変わらず使用できます。

```bash
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';
import { AuthorizationHeaderStrategyWithKey } from 'customized-jwt-strategy';
@Injectable()
export class JwtStrategy extends AuthorizationHeaderStrategyWithKey.strategy({ secretOrKey: jwtConstants.secret }) {
    constructor() {
        super();
    }
    async validate(payload: any) {
        return { userId: payload.sub, username: payload.username };
    }
}
```

こうすれば、必須オプションが secretOrKey だけになり、インターフェースを設定しているので validate 関数を定義していないと静的にエラーを発生するようにしてくれます。
個人的には使いやすくなったかなと思うので、満足です。

# おわりに

今回は[ドキュメント](https://docs.nestjs.com/recipes/passport)で passport を NestJS で使う方法から始まり、JwtStraetgy 周りのカスタマイズまで行っていきました。
説明してきたのは、一ガードを作成するためだけのとても細かい話がほとんどでしたが、そこには深い世界がありました。
内部の構造を詳細に見ていくことで、ストラテジー周りの理解と便利さをより実感できたので良かったです。
ただ、とにかく疲れました。
扱う領域はかなり狭いのに、Docker の記事を書いた時よりも疲れました。
せっかく頑張ったので、NestJS においてのストラテジー周りで困っている人に届いてくれたら良いなと思っています。
ここまで読んでいただきありがとうございました。
