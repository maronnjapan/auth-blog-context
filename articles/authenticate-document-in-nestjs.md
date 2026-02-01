---
title: "NestJsの認証周りのドキュメントを読み直す"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Typescript", "NestJS", "JWT"]
published: true
---

# はじめに

ある日 NestJS で書かれた認証・認可周りのコードを眺めていました。
すると、「これってなんやっけ？」と内部の実装が読めなくなっていました。
これはまずい！と思い、再勉強も兼ねて NestJS のドキュメントを読み直すことにしました。
そのため、今回は NestJS が展開している[Authentication](https://docs.nestjs.com/security/authentication)を読み直した際のメモとなっています。
JWT モジュールの使い方やガードの実装などおぼろげになっている方は眺めていただけると幸いです。

# ドキュメントの導入

ドキュメントはまず認証周りの要件を以下のように定義しています。
① ユーザー名とパスワードで認証する処理を実装する。
② 認証が終わったら、Bearer Token として送る Json Web Token(JWT)を発行する。
③ 適切な JWT を搭載したリクエストのみ許可するための設定を作成する。
これら機能の完成を目的としてドキュメントは書かれています。

# 認証機能作成

## 必要なモジュール作成

まず以下のコマンドを実行し、auth.module.ts, auth.service.ts ,auth.controller.ts を作成して認証用のモジュールを作成するようにしています。

```
$ nest g module auth
$ nest g controller auth
$ nest g service auth
```

次にユーザー情報を扱う用に、コマンドで user.module.ts, user.service.ts を作成しています。
user.service.ts は以下のようなコードを実装します。

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

ユーザー名に対して、一致する値を返すメソッドを搭載しています。
なお、今回は認証の流れを重視しているので、ドキュメントではデータベースを構築せずリテラルでユーザー情報を定義しています。
言わずもがなですが、このコードは本番環境では使えないコードです。
ただ、実際に使うには適していないコードは出てきますので、あくまでドキュメントは流れを記載しているものだとご留意ください。
UserService クラスを外部モジュールでも使用するために、user.module.ts は以下のように記載します。

```tsx
import { Module } from "@nestjs/common";
import { UsersService } from "./users.service";
@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

## 認証用のエンドポイントの作成

先程作成した AuthService クラスに signIn メソッドを以下のように実装します。

```tsx
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { UsersService } from "../users/users.service";
@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  async signIn(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: Generate a JWT and return it here
    // instead of the user object
    return result;
  }
}
```

これはログイン処理を疑似的に実装したもので、一致するユーザー情報があれば、パスワード以外のユーザー情報を返すようにしています。
処理の準備ができたので、AuthController クラスにエンドポイントを作成します。

```tsx
import { Body, Controller, Post, HttpCode, HttpStatus } from "@nestjs/common";
import { AuthService } from "./auth.service";
@Controller("auth")
export class AuthController {
  constructor(private authService: AuthService) {}
  @HttpCode(HttpStatus.OK)
  @Post("login")
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }
}
```

ここまでで、「① ユーザー名とパスワードで認証する処理を実装する。」が完了しました。
次に「② 認証が終わったら、Bearer Token として送る Json Web Token(JWT)を発行する。」を行っていきます。

# JWT 発行処理の実装

まず`npm install --save @nestjs/jwt`で JWT 発行に必要なモジュールをインポートします。
そして、auth.module.ts に上記モジュールを使用するための設定をします。

```tsx
import { Module } from "@nestjs/common";
import { AuthController } from "./auth.controller";
import { AuthService } from "./auth.service";
import { UsersModule } from "../users/users.module";
import { JwtModule } from "@nestjs/jwt";
@Module({
  controllers: [AuthController],
  providers: [AuthService],
  imports: [UsersModule, JwtModule],
})
export class AuthModule {}
```

インポートしたら、@nestjs/jwt が持っている JwtService クラスを AuthService クラスに注入します。

```tsx
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { UsersService } from "../users/users.service";
import { JwtService } from "@nestjs/jwt";
@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}
  async signIn(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.userId, username: user.username };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

そして、signIn メソッドを JwtService クラスが持っている signAsync メソッドを使用するように変更します。
これによって、JWT 部分が変数 payload の値である JWS 形式が作成されます。
ただ、現状では署名するための鍵がないためこの処理を設定するとエラーが発生します。signAsync メソッド内に鍵を設定するオプションはあるのですが、署名する鍵を複数設定することはないと思います。
そのため、今回は JwtModule に情報を登録するメソッドが存在しているのでそちらで設定を行います。
auth フォルダ配下に constants.ts を作成し、以下の内容を記載します。

```tsx
export const jwtConstants = {
  secret:
    "DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.",
};
```

これは署名をするときに使用する鍵です。
もちろん鍵としてはあまりに推測されやすいものなので、本番環境では使用しないでください。
次に、auth.module.ts の JwtModule を修正します。

```tsx
@Module({
  controllers: [AuthController],
  providers: [AuthService],
  imports: [UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ]
})
```

register メソッドを使用し、global オプション、secret オプション、signOptions オプションをそれぞれ設定します。
global オプションを true にすると、他のフォルダで JwtModule をインポートしなくても、ライブラリの中身を使用できます。
今回で言うと、user.module.ts に JwtModule をインポートしなくても、JwtService を user.service.ts などで使えるようになります。
secret オプションは鍵情報を登録します。
基本的に JwtService の signAsync メソッドなど署名や検証を行う処理はここで登録された情報を使用します。
`signOptions: { expiresIn: '60s' },`は JWT の有効期限を決めています。
今回は 60 秒にしています。
それではターミナルで`curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"`を実行して見ます。
すると、以下のようなオブジェクトが返ってきます。

```tsx
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjEsInVzZXJuYW1lIjoiam9obiIsImlhdCI6MTY5ODk5MTkzNywiZXhwIjoxNjk4OTkxOTk3fQ._qd51vXSfOAIwJGl4xR7BXSUPb62Q85N0IaxkXGkwDA"}
```

このアクセストークンを[jwt.io](https://jwt.io/)に入力してみると画像のようにアクセストークンの情報が展開されます。
![2023-11-03_15h15_08.png](/images/authenticate-document-in-nestjs/2023-11-03_15h15_08.png)
ここまででパスワード認証から、アクセストークンの発行まで完了しました。
では最後に、適切な JWT が含まれていないリクエストの場合 API が実行できないようにするガード機能を作成します。

# ガード機能作成

auth フォルダ配下に auth.gurad.ts を作成し、以下のコードを記載します。

```tsx
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { Request } from "express";
import { jwtConstants } from "./constants";
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret: jwtConstants.secret,
      });
      // 💡 We're assigning the payload to the request object here
      // so that we can access it in our route handlers
      request["user"] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }
  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(" ") ?? [];
    return type === "Bearer" ? token : undefined;
  }
}
```

ここで行っている処理は、次の通りです。

1. リクエストから、Authorization ヘッダーにある Bearer トークンを取得しています。
2. トークンが無ければ例外を発生させます。
3. トークンの署名の検証を行います。
4. 検証が失敗すれば例外を投げ、成功すれば Request オブジェクトの user プロパティに payload の値を格納します。
5. 全ての処理が完了した場合、true を返すようにします。

ガードを作成できたら、auth.controller.ts に以下のエンドポイントを作成します。

```tsx
@UseGuards(AuthGuard)
@Get('profile')
getProfile(@Request() req) {
  return req.user;
}
```

ポイントは`@UseGuards(AuthGuard)`です。
これを付与することで、AuthGuard の canActivate メソッドが true を返さない限り、対象のエンドポイントは実行できないようになります。
実際に、アクセストークンのある場合と無い場合の curl を実行すると以下のようになります。

```
$ # GET /profile
$ curl http://localhost:3000/auth/profile
{"statusCode":401,"message":"Unauthorized"}
$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/auth/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
{"sub":1,"username":"john","iat":...,"exp":...}
```

アクセストークンが適切に設定されていない場合は auth.guard.ts で設定した 401 エラーが返ってきて、適切なアクセストークンがある場合は payload の値が表示されます。
ここまでで最初の要件は満たすことできました。
では最後に、作成したガードを全ての API エンドポイントに適用するのと、ガードのチェックを行わないようにする処理について見ていきます。

# ガードのグローバル化とガードをスルーさせる方法

先程作成したガードを全てのエンドポイントに適用するには次の二つがあります。
①app.module.ts に設定する。
②main.ts で`useGlobalGuards`を使い登録する。
まずは ① の方法です。
app.module.ts の providers へ以下のように AuthGuard を追加します。

```tsx
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { AuthModule } from "./auth/auth.module";
import { UsersModule } from "./users/users.module";
import { APP_GUARD } from "@nestjs/core";
import { AuthGuard } from "./auth/auth.gurad";
@Module({
  imports: [AuthModule, UsersModule],
  controllers: [AppController],
  providers: [
    AppService,
    //以下を追加
    {
      provide: APP_GUARD,
      useClass: AuthGuard,
    },
  ],
})
export class AppModule {}
```

`APP_GUARD`は全てのエンドポイントに対して、ガードを適用するための定数となっています。
これを module に設定することで、全ての API を実行する前に useClass で設定したガードが走るようになります。
なお、この providers に設定する方法は app.module.ts 以外の module ファイルで行っても同様の動きをします。
ただ、グローバルのものを各 module に設定してしまうとわけわからなくなるので、個人的には app.module.ts に書いておくのが安心だと思います。
つづいて、② について解説します。
とはいえ、main.ts に以下の内容を記載するだけです。

```tsx
app.useGlobalGuards(new AuthGuard(new JwtService()));
```

このように書いても先程と同様に全てのエンドポイントにガードが適用されます。
ここまでで、グローバルなガードを設定しました。
グローバルに設定するということはほとんどのエンドポイントで実行して欲しいことが分かります。
ただ、一部のエンドポイントに関してはガードが適用されたくないと思う可能性はあります。
そのため、ここからはグローバルなガードを適用させないためのデコレーターを作成します。
まず任意の TS ファイルを作成し、以下のコードを記載します。

```tsx
import { SetMetadata } from "@nestjs/common";
export const IS_PUBLIC_KEY = "isPublic";
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

次に、auth.guard.ts を以下のように変更します。

```tsx
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { Request } from "express";
import { jwtConstants } from "./constants";
import { Reflector } from "@nestjs/core";
import { IS_PUBLIC_KEY } from "src/public";
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService, private reflector: Reflector) {}
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true;
    }
    const request = context.switchToHttp().getRequest<Request>();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException("Fooooooooooooooooooooooooooooooooo");
    }
    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret: jwtConstants.secret,
      });
      // 💡 We're assigning the payload to the request object here
      // so that we can access it in our route handlers
      request["user"] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }
  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(" ") ?? [];
    return type === "Bearer" ? token : undefined;
  }
}
```

ポイントは以下のコードです。

```tsx
const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
  context.getHandler(),
  context.getClass(),
]);
if (isPublic) {
  return true;
}
```

context の getHandler メソッドか getClass メソッド内に、`IS_PUBLIC_KEY` があれば true を返すようにします。
これによって、後の処理が走らなくなります。
では、ガードをグローバルで有効にしたら、app.controller.ts に以下のエンドポイントを追加します。

```tsx
@Get()
@Public()
async getHello() {
  return 'hello'
}
```

先程作成した Public 関数をデコレーターで設定することで、この処理はガード内に書いた Public 用の分岐を通るようになります。
実際に、`curl [http://localhost:3000/](http://localhost:3000/)`を実行しても、401 エラーにならず「hello」が表示されることを確認できます。
以上で、NestJs が展開している[Authentication](https://docs.nestjs.com/security/authentication)の章の概要となります。

# 余談 ① ガード周りの処理

先程ガードの作成を行い、実際に動くところまで確認しました。
ただ、もう少しコードを追ってみたいと思います。
なお、展開するコードは基本的にhttps://github.com/nestjs/nestのリポジトリ内に存在するものを指します。
まずはガード本体を作成する時に使用した CanActivate インターフェースです。
これは packages/common/interfaces/features/can-activate.interface.ts にコードがあり、以下のような記載となっています。

```tsx
import { Observable } from "rxjs";
import { ExecutionContext } from "./execution-context.interface";
/**
 * Interface defining the `canActivate()` function that must be implemented
 * by a guard.  Return value indicates whether or not the current request is
 * allowed to proceed.  Return can be either synchronous (`boolean`)
 * or asynchronous (`Promise` or `Observable`).
 *
 * @see [Guards](https://docs.nestjs.com/guards)
 *
 * @publicApi
 */
export interface CanActivate {
  /**
   * @param context Current execution context. Provides access to details about
   * the current request pipeline.
   *
   * @returns Value indicating whether or not the current request is allowed to
   * proceed.
   */
  canActivate(
    context: ExecutionContext
  ): boolean | Promise<boolean> | Observable<boolean>;
}
```

canActivate というメソッドだけが存在するインターフェースとなっています。
だから、ガードの実際の処理は canActivate というメソッド名になっていたんですね。
戻り値は boolean 系を戻せば良さそうです。
案外単純なんですね。
なら、そもそも CanActivate インターフェースを継承せず boolean を返すメソッドがるクラスでも良さそうに感じます。
そして、実際に CanActivate インターフェースを継承しなくてガードは作成できます。
ただ、必ず canActivate というメソッド名を持つ必要があります。
その理由について見ていきます。
まず RouterExecutionContext クラスの[createGuardsFn メソッド](https://github.com/nestjs/nest/blob/7359a35773617134e462b4b2a9f8cffd81182216/packages/core/router/router-execution-context.ts#L356)を確認します。

```tsx
public createGuardsFn<TContext extends string = ContextType>(
    guards: CanActivate[],
    instance: Controller,
    callback: (...args: any[]) => any,
    contextType?: TContext,
  ): (args: any[]) => Promise<void> | null {
    const canActivateFn = async (args: any[]) => {
      const canActivate = await this.guardsConsumer.tryActivate<TContext>(
        guards,
        args,
        instance,
        callback,
        contextType,
      );
      if (!canActivate) {
        throw new ForbiddenException(FORBIDDEN_MESSAGE);
      }
    };
    return guards.length ? canActivateFn : null;
  }
```

ガードの設定を行った場合、ここの処理が走り変数 canActivate の値が true でないとエラーが発生します。
とはいえ、まだ canActive メソッドは出ていません。
次に、createGuardsFn メソッド内の[tryActivate メソッド](https://github.com/nestjs/nest/blob/7359a35773617134e462b4b2a9f8cffd81182216/packages/core/guards/guards-consumer.ts#L8)の中身を確認します。

```tsx
public async tryActivate<TContext extends string = ContextType>(
    guards: CanActivate[],
    args: unknown[],
    instance: Controller,
    callback: (...args: unknown[]) => unknown,
    type?: TContext,
  ): Promise<boolean> {
    if (!guards || isEmpty(guards)) {
      return true;
    }
    const context = this.createContext(args, instance, callback);
    context.setType<TContext>(type);
    for (const guard of guards) {
      const result = guard.canActivate(context);
      if (await this.pickResult(result)) {
        continue;
      }
      return false;
    }
    return true;
  }
```

ここでようやく canActivate メソッドが出てきました。
そして、canActivate を実行している guard の型を確認すると CanActivate となっています。
このことから、ガードに設定しているクラスに canActivate メソッドを設定しなければならない理由が分かります。
ちなみにですが、pickResult メソッドですがこちらは boolean を受け取り boolean を返すメソッドとなっています。
なので、ガードで設定した canActivate メソッドは処理に問題が無ければ true を返すことが求められていたんですね。
ここまでで、CanAcitivate インターフェースに関わることを見ていき、CanAcitivate インターフェースは要らないのではと言いました。
実際に無くても動かせると言えば動かせるのですが、あった方がガードに必要なメソッドや戻り値を設定しやすいです。
なので、インターフェースを見ると気分が悪くなる人以外は、ガードを使用する際 CanAcitivate インターフェースを実装するようにした方が安全です。
ガードの中身について見てきたので、ガードそのものを設定する[UseGuards](https://github.com/nestjs/nest/blob/7359a35773617134e462b4b2a9f8cffd81182216/packages/common/decorators/core/use-guards.decorator.ts#L28)の処理も確認してみます。

```tsx
export function UseGuards(
  ...guards: (CanActivate | Function)[]
): MethodDecorator & ClassDecorator {
  return (
    target: any,
    key?: string | symbol,
    descriptor?: TypedPropertyDescriptor<any>,
  ) => {
    const isGuardValid = <T extends Function | Record<string, any>>(guard: T) =>
      guard &&
      (isFunction(guard) ||
        isFunction((guard as Record<string, any>).canActivate));
    if (descriptor) {
      validateEach(
        target.constructor,
        guards,
        isGuardValid,
        '@UseGuards',
        'guard',
      );
      extendArrayMetadata(GUARDS_METADATA, guards, descriptor.value);
      return descriptor;
    }
    validateEach(target, guards, isGuardValid, '@UseGuards', 'guard');
    extendArrayMetadata(GUARDS_METADATA, guards, target);
    return target;
  };
```

細かい処理については解説しませんが、引数の guards が関数型か canActivate メソッドを持っているかをチェックした後、Reflecter オブジェクトのメタデータにガードがあることを示すデータを登録します。
これによって、ガードチェックを API チェック前に走ることを可能にします。

# 余談 ② ガード内の Public の処理

ここでは auth.guard.ts で行っていた以下のコードについて見ていきます。

```tsx
const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
  context.getHandler(),
  context.getClass(),
]);
```

まず、[getAllAndOverride](https://github.com/nestjs/nest/blob/7359a35773617134e462b4b2a9f8cffd81182216/packages/core/services/reflector.service.ts#L227)メソッドの中身を見てみます。

```tsx
public getAllAndOverride<TResult = any, TKey = any>(
    metadataKeyOrDecorator: TKey,
    targets: (Type<any> | Function)[],
  ): TResult {
    for (const target of targets) {
      const result = this.get(metadataKeyOrDecorator, target);
      if (result !== undefined) {
        return result;
      }
    }
    return undefined;
  }
```

次に、getAllAndOverride メソッドの根幹に関わりそうな get メソッドを見てみます。

```tsx
public get<TResult = any, TKey = any>(
    metadataKeyOrDecorator: TKey,
    target: Type<any> | Function,
  ): TResult {
    const metadataKey =
      (metadataKeyOrDecorator as ReflectableDecorator<unknown>).KEY ??
      metadataKeyOrDecorator;
    return Reflect.getMetadata(metadataKey, target);
  }
```

判定したいメタキーを取得して、それが target のキーに存在するかを確認しています。
存在する場合は true を返し、存在しない場合は undefined を返します。
なお、Reflect オブジェクトは[reflect-metadata](https://github.com/rbuckton/reflect-metadata/tree/3aeb98af4030be664a66f49bfd164936e0ba1825)というライブラリのものを使用しています。
reflect-metadata を有効にすれば Reflect オブジェクトはグローバルに使用できるので、ファイル内でインポートする必要がありません。
初見だと逆に分かりにくいので、ご注意ください。
また、getMetadata メソッドの中身について解説がしたかったのですが、読んでもいまいち分かりませんでした。
何とか getMetadata メソッドは Reflect オブジェクト内に対象のキーが存在するかを判定していることは読み取れましたが、具体的にどうやってキーを取得・判定しているかは理解できませんでした。
以上のことから、内部の詳細な操作は曖昧だが getAllAndOverride メソッドは Reflect オブジェクト内に指定のキーが存在しているかを判定している処理だと分かります。
やりたいことは単純ですが、中身は難しかったです。

# おわりに

今回は[NestJS のドキュメント](https://docs.nestjs.com/security/authentication)の読み直し記事でした。
使い方を忘れていたところや、そもそもなぜそのコードを書くのか理解できなかった部分を整理できました。
[認証に関わる NestJS のドキュメント](https://docs.nestjs.com/recipes/passport)はまだ passport を使った章もあるので、こちらもいつか読んで展開したいと思います。
ここまで読んでいただきありがとうございました。
