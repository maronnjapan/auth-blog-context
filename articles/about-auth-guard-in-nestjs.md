---
title: "NestJSのAuthGuard周りをアレンジしたり、深掘りしたり"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS", "passport", "Strategy"]
published: true
---

## はじめに

以前、NestJS でストラテジーを使う時のガード処理についてのブログを書きました。
それのおかげでストラテジー自体の理解であったり、NestJS でストラテジーを登録することであったりの理解は深まりました。
ただ、AuthGuard が具体的にどうやってガード処理とストラテジーとの接続を行っているか曖昧なままでした。
そこで今回は AuthGuard に着目し、AuthGuard がパスポートを用いて具体的にどのように検証を実行しているかを確認したり、AuthGuard の簡単なアレンジを行っていきます。

## この記事のこれだけ！！！

[AuthGuard 関数](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts)の CanActivate メソッドは以下の処理を行います。
①「passport-」で始まる[ストラテジーモジュール](https://www.passportjs.org/packages/)の strategy.js OR strategy.ts において、`ストラテジーオブジェクト.prototype.authenticate`で定義されている関数を実行します。
②① の内部処理の最後に、NestJS のドキュメントにある[JwtStrategy](https://docs.nestjs.com/recipes/passport#implementing-passport-jwt)や[LocalStrategy](https://docs.nestjs.com/recipes/passport#implementing-passport-strategies)で定義した validate メソッドを実行します。
③handleRequest を実行します。戻り値の user は処理が成功していれば ② の validate メソッドで返した値です。何かしら失敗している場合の値は`false`です。

## 設定するストラテジーをリクエストの値で動的に設定する

### 達成したいこと

リクエストの Authorization ヘッダーに値があれば、JwtStrategy を設定しなければ LocalStrategy を使て検証するようにします。

### 完成したコード

```tsx
import {
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from "@nestjs/common";
import { AuthGuard, AuthModuleOptions } from "@nestjs/passport";
import { Request, Response } from "express";
import * as passport from "passport";

@Injectable()
export class HandleAuthGuard extends AuthGuard() {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const strategyName = request.headers["authorization"] ? "jwt" : "local";
    const response = context.switchToHttp().getResponse<Response>();
    const options: AuthModuleOptions = {
      defaultStrategy: strategyName,
      session: false,
      property: "user",
    };
    const passportFn = this.createPassportContext(request, response);
    const user = await passportFn(
      options.defaultStrategy,
      options,
      (err, user, info, status) =>
        this.handleRequest(err, user, info, context, status)
    );
    request[options.property] = user;
    return true;
  }
  handleRequest<TUser = any>(
    err: any,
    user: any,
    info: any,
    context: ExecutionContext,
    status?: any
  ): TUser {
    if (err || !user) {
      throw err || new UnauthorizedException();
    }
    return user;
  }
  createPassportContext(request: Request, response: Response) {
    return (type: string, options, callback: Function) =>
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
  }
}
```

### 着目点

コードで着目する点は以下の点です。

```tsx
const request = context.switchToHttp().getRequest<Request>();
const strategyName = request.headers["authorization"] ? "jwt" : "local";
```

というか、ここが独自に設定した全てです。
後は[auth.guard.ts](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts)のコードをほぼコピペしています。
その他の機能については後ほど解説しますが、一旦ストラテジーを動的にするという目的は以上で達成できます。
CanAcitivate の引数にはリクエストオブジェクトを取得できるメソッドがあるので、それに型を付与しつつ呼び出しています。
そして、リクエストオブジェクトの中に Authorization ヘッダーが含まれているのかを確認し、使用するストラテジーの値を設定しています。
やりたいことは簡単ですね。
ただ、これを行うためには CanActivate メソッドをある程度自分で実装する必要があるので、[auth.guard.ts](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts)のコードをもとに大部分を引っ張っている形となっています。

### その他の着目点

今回のアレンジとは直接関係ないですが、CanActivate メソッドで根幹を示すのは createPassportContext メソッドです。

```tsx
    createPassportContext(request: Request, response: Response) {
        return (type: string, options, callback: Function) =>
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
    }
```

なにやら複雑なことを行っていますが、このメソッドの役割は 3 つだけ覚えておけば良いです。
①「passport-」で始まる[ストラテジーモジュール](https://www.passportjs.org/packages/)の strategy.js OR strategy.ts において、`ストラテジーオブジェクト.prototype.authenticate`で定義されている関数を実行します。
②① の内部処理の最後※に、NestJS のドキュメントにある[JwtStrategy](https://docs.nestjs.com/recipes/passport#implementing-passport-jwt)や[LocalStrategy](https://docs.nestjs.com/recipes/passport#implementing-passport-strategies)で定義した validate メソッドを実行します。
③handleRequest を実行します。戻り値の user は処理が成功していれば ② の validate メソッドで返した値です。何かしら失敗している場合の値は`false`です。
上記に至るまで色々なファイルをまたいでいるので、流れを追うのは大変です。
ただし、実際やっていることは先程挙げた 3 つに集約されているので、開発で使う場合はそれさえ意識しておけば問題なく使いこなせます。
※私が確認した範囲では処理の最後でしたが、全てを確認していないので場合によっては最後ではないかもしれません。ご承知おきください。

## コードを追いかける

ここからは createPassportContext 周りが 3 つの主要機能を担うと判断した過程を追っていきます。
なお、ここからの話は読まなくても AuthGuard は使いこなせるので、お時間ある方だけ読んでいただけますと幸いです。
まずは createPassportContext の外枠を確認します。

```tsx
    createPassportContext(request: Request, response: Response) {
        return (type: string, options, callback: Function) =>
            new Promise<void>((resolve, reject) =>
                //...略
            );
    }
```

戻り値が Promise の関数を返すのが外枠の機能となります。
実は後のコードを見る限り、内部の処理は全て同期関数として定義されています。
そのため、同期関数として返しても動くとは思います。
ただ、今後中で使用しているモジュールが非同期処理を使用した場合、予期せず処理が進んでしまう可能性があります。
そういったケースを想定して、ここでは Promise を返すようにしていそうです。(間違っていたらすみません)
外枠は Promise として返すだけなので、主要な機能は以下の部分になります。

```tsx
passport.authenticate(type, options, (err, user, info, status) => {
  try {
    request.authInfo = info;
    return resolve(callback(err, user, info, status));
  } catch (err) {
    reject(err);
  }
})(request, response, (err) => (err ? reject(err) : resolve()));
```

ここは passport オブジェクトの authenticate メソッドを実行し、戻り値がどうやら関数ぽいので、その関数を実行しているのが役割となっていそうです。
それでは、passport オブジェクトを見ていきましょう。
passport モジュールの[index.js](https://github.com/jaredhanson/passport/blob/master/lib/index.js)を確認すると以下の通りです。

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

passport オブジェクトから直接メソッドを呼び出し時は、`exports = module.exports = new Passport();`で定義したクラスのメソッドを呼び出していそうです。
なので、[lib/authenticator.js](https://github.com/jaredhanson/passport/blob/master/lib/authenticator.js)の中身を確認していきます。
今回関わる処理は以下の部分です。

```tsx
function Authenticator() {
  //...略
  this._framework = null;

  this.init();
}
Authenticator.prototype.init = function () {
  this.framework(require("./framework/connect")());
  //...略
};
Authenticator.prototype.framework = function (fw) {
  this._framework = fw;
  return this;
};
Authenticator.prototype.authenticate = function (strategy, options, callback) {
  return this._framework.authenticate(this, strategy, options, callback);
};
```

Authenticator オブジェクトを生成したタイミングで、lib/framework/connect.js をインポートし、返ってきた値を\_framework プロパティに格納しています。
そして、authenticate メソッドは\_framework プロパティに存在する authenticate メソッドを実行しています。
では、lib/framework/connect.js の中身を確認します。

```tsx
var initialize = require("../middleware/initialize"),
  authenticate = require("../middleware/authenticate");
exports = module.exports = function () {
  return {
    initialize: initialize,
    authenticate: authenticate,
  };
};
```

さらに外部ファイルをインポートして、それをオブジェクトとして返しているだけですね。
なので、今回中心になる処理は lib/middleware/authenticate.js だと言えそうです。
[lib/middleware/authenticate.js](https://github.com/jaredhanson/passport/blob/master/lib/middleware/authenticate.js)を確認します。

```tsx
module.exports = function authenticate(passport, name, options, callback) {
  if (typeof options == "function") {
    callback = options;
    options = {};
  }
  options = options || {};

  var multi = true;

  if (!Array.isArray(name)) {
    name = [name];
    multi = false;
  }

  return function authenticate(req, res, next) {
    req.login = req.logIn = req.logIn || IncomingMessageExt.logIn;
    req.logout = req.logOut = req.logOut || IncomingMessageExt.logOut;
    req.isAuthenticated =
      req.isAuthenticated || IncomingMessageExt.isAuthenticated;
    req.isUnauthenticated =
      req.isUnauthenticated || IncomingMessageExt.isUnauthenticated;

    req._sessionManager = passport._sm;

    var failures = [];

    function allFailed() {
      if (callback) {
        if (!multi) {
          return callback(
            null,
            false,
            failures[0].challenge,
            failures[0].status
          );
        } else {
          var challenges = failures.map(function (f) {
            return f.challenge;
          });
          var statuses = failures.map(function (f) {
            return f.status;
          });
          return callback(null, false, challenges, statuses);
        }
      }

      var failure = failures[0] || {},
        challenge = failure.challenge || {},
        msg;

      if (options.failureFlash) {
        var flash = options.failureFlash;
        if (typeof flash == "string") {
          flash = { type: "error", message: flash };
        }
        flash.type = flash.type || "error";

        var type = flash.type || challenge.type || "error";
        msg = flash.message || challenge.message || challenge;
        if (typeof msg == "string") {
          req.flash(type, msg);
        }
      }
      if (options.failureMessage) {
        msg = options.failureMessage;
        if (typeof msg == "boolean") {
          msg = challenge.message || challenge;
        }
        if (typeof msg == "string") {
          req.session.messages = req.session.messages || [];
          req.session.messages.push(msg);
        }
      }
      if (options.failureRedirect) {
        return res.redirect(options.failureRedirect);
      }

      var rchallenge = [],
        rstatus,
        status;

      for (var j = 0, len = failures.length; j < len; j++) {
        failure = failures[j];
        challenge = failure.challenge;
        status = failure.status;

        rstatus = rstatus || status;
        if (typeof challenge == "string") {
          rchallenge.push(challenge);
        }
      }

      res.statusCode = rstatus || 401;
      if (res.statusCode == 401 && rchallenge.length) {
        res.setHeader("WWW-Authenticate", rchallenge);
      }
      if (options.failWithError) {
        return next(
          new AuthenticationError(http.STATUS_CODES[res.statusCode], rstatus)
        );
      }
      res.end(http.STATUS_CODES[res.statusCode]);
    }

    (function attempt(i) {
      var layer = name[i];
      if (!layer) {
        return allFailed();
      }

      var strategy, prototype;
      if (typeof layer.authenticate == "function") {
        strategy = layer;
      } else {
        prototype = passport._strategy(layer);
        if (!prototype) {
          return next(
            new Error('Unknown authentication strategy "' + layer + '"')
          );
        }

        strategy = Object.create(prototype);
      }

      strategy.success = function (user, info) {
        if (callback) {
          return callback(null, user, info);
        }

        info = info || {};
        var msg;

        if (options.successFlash) {
          var flash = options.successFlash;
          if (typeof flash == "string") {
            flash = { type: "success", message: flash };
          }
          flash.type = flash.type || "success";

          var type = flash.type || info.type || "success";
          msg = flash.message || info.message || info;
          if (typeof msg == "string") {
            req.flash(type, msg);
          }
        }
        if (options.successMessage) {
          msg = options.successMessage;
          if (typeof msg == "boolean") {
            msg = info.message || info;
          }
          if (typeof msg == "string") {
            req.session.messages = req.session.messages || [];
            req.session.messages.push(msg);
          }
        }
        if (options.assignProperty) {
          req[options.assignProperty] = user;
          if (options.authInfo !== false) {
            passport.transformAuthInfo(info, req, function (err, tinfo) {
              if (err) {
                return next(err);
              }
              req.authInfo = tinfo;
              next();
            });
          } else {
            next();
          }
          return;
        }

        req.logIn(user, options, function (err) {
          if (err) {
            return next(err);
          }

          function complete() {
            if (options.successReturnToOrRedirect) {
              var url = options.successReturnToOrRedirect;
              if (req.session && req.session.returnTo) {
                url = req.session.returnTo;
                delete req.session.returnTo;
              }
              return res.redirect(url);
            }
            if (options.successRedirect) {
              return res.redirect(options.successRedirect);
            }
            next();
          }

          if (options.authInfo !== false) {
            passport.transformAuthInfo(info, req, function (err, tinfo) {
              if (err) {
                return next(err);
              }
              req.authInfo = tinfo;
              complete();
            });
          } else {
            complete();
          }
        });
      };

      strategy.fail = function (challenge, status) {
        if (typeof challenge == "number") {
          status = challenge;
          challenge = undefined;
        }

        // push this failure into the accumulator and attempt authentication
        // using the next strategy
        failures.push({ challenge: challenge, status: status });
        attempt(i + 1);
      };

      strategy.redirect = function (url, status) {
        res.statusCode = status || 302;
        res.setHeader("Location", url);
        res.setHeader("Content-Length", "0");
        res.end();
      };

      strategy.pass = function () {
        next();
      };

      strategy.error = function (err) {
        if (callback) {
          return callback(err);
        }

        next(err);
      };

      strategy.authenticate(req, options);
    })(0); // attempt
  };
};
```

中々長いですが、今回はコールバック関数を設定しているのと、今回登録したストラテジーでは使用しないメソッドもあるので、実際に見る場所はもっと少なくなります。

```tsx
module.exports = function authenticate(passport, name, options, callback) {
  if (typeof options == "function") {
    callback = options;
    options = {};
  }
  options = options || {};

  var multi = true;

  //ストラテジー名が文字列で渡された場合は配列に変換する。
  if (!Array.isArray(name)) {
    name = [name];
    multi = false;
  }

  return function authenticate(req, res, next) {
    //...略

    var failures = [];

    function allFailed() {
      if (callback) {
        if (!multi) {
          return callback(
            null,
            false,
            failures[0].challenge,
            failures[0].status
          );
        } else {
          var challenges = failures.map(function (f) {
            return f.challenge;
          });
          var statuses = failures.map(function (f) {
            return f.status;
          });
          return callback(null, false, challenges, statuses);
        }
      }
      //...略
    }

    (function attempt(i) {
      var layer = name[i];
      if (!layer) {
        return allFailed();
      }

      var strategy, prototype;
      if (typeof layer.authenticate == "function") {
        strategy = layer;
      } else {
        prototype = passport._strategy(layer);
        if (!prototype) {
          return next(
            new Error('Unknown authentication strategy "' + layer + '"')
          );
        }

        strategy = Object.create(prototype);
      }

      strategy.success = function (user, info) {
        if (callback) {
          return callback(null, user, info);
        }
        //...略
      };

      strategy.fail = function (challenge, status) {
        if (typeof challenge == "number") {
          status = challenge;
          challenge = undefined;
        }

        // push this failure into the accumulator and attempt authentication
        // using the next strategy
        failures.push({ challenge: challenge, status: status });
        attempt(i + 1);
      };

      strategy.error = function (err) {
        if (callback) {
          return callback(err);
        }
        //...略
      };

      strategy.authenticate(req, options);
    })(0); // attempt
  };
};
```

大分スッキリできたので、もう少し詳細をみていきます。

```tsx
if (typeof options == "function") {
  callback = options;
  options = {};
}
options = options || {};

var multi = true;

//ストラテジー名が文字列で渡された場合は配列に変換する。
if (!Array.isArray(name)) {
  name = [name];
  //ストラテジーの複数登録フラグをfalseにする。
  multi = false;
}
```

こちらは引数をもとに、変数を定義しております。
そこまで重要な部分はないですが、変数 name については引数で文字列・文字列の配列どちらであろうとも配列に変換されます。
そのため、[AuthGuard](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L37)では文字列と配列両方を許容していました。
次に進みます。

```tsx
return function authenticate(req, res, next) {};
```

authenticate メソッドは上記関数を返す処理となっています。
引数 req や res はその名の通り、リクエストオブジェクトとレスポンスオブジェクトを想定しています。
引数 next はコールバック関数を定義していますが、今回は authenticate メソッドでコールバック関数を設定しいるので、使いません。
返す値は分かったので、authenticate 関数の中身を順にみていきます。

```tsx
var failures = [];
function allFailed() {
  if (callback) {
    if (!multi) {
      return callback(null, false, failures[0].challenge, failures[0].status);
    } else {
      var challenges = failures.map(function (f) {
        return f.challenge;
      });
      var statuses = failures.map(function (f) {
        return f.status;
      });
      return callback(null, false, challenges, statuses);
    }
  }
  //...略
}
```

登録した全てのストラテジーがエラーだった場合に実行される関数となっています。
これは passport モジュールを使う場合の取り決めみたいなものですが、ストラテジーによる検証が成功した時は第二引数に検証した結果を返すようにします。
その他引数ですが、第一引数はエラー情報を格納します。
なんですが、allFailed 関数では null を返していますし、第二引数も失敗した時は false を渡すようにしているので、あまり使用しません。
第三引数は成功・失敗した時のメッセージが格納されます。
第四引数は失敗した時のみ、ステータスコードが格納されます。
結構厳密にエラーハンドリングをする時は使えそうです。
このようにコールバック関数が設定されている時は、失敗時 passport モジュール側ではなく、コールバック関数内でエラーハンドリングを行う必要があります。
そのため、auth.guard.ts にはエラーハンドリングを行う[メソッド](https://github.com/nestjs/passport/blob/master/lib/auth.guard.ts#L96C1-L101C6)が定義されています。

```tsx
handleRequest(err, user, info, context, status): TUser {
      if (err || !user) {
        throw err || new UnauthorizedException();
      }
      return user;
    }
```

allFailed 関数の役割をみたので、実際に実行される処理部分の外枠を確認します。

```tsx
(function attempt(i) {
  var layer = name[i];
  if (!layer) {
    return allFailed();
  }
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
  strategy.fail = function (challenge, status) {
    //...略
    attempt(i + 1);
  };
  //...略

  strategy.authenticate(req, options);
})(0);
```

ここが今回の肝の一つです。
まず以下の部分で、登録した配列の指定インデックスにストラテジー名が存在しているかを確認しており、なければ先程説明した allFailed 関数を実行します。

```tsx
var layer = name[i];
if (!layer) {
  return allFailed();
}
```

指定インデックスと書いたので、ループすることが分かります。
そして、ループは以下の部分で行うようになっています。

```tsx
(function attempt(i) {
  //...略
  strategy.fail = function (challenge, status) {
    //...略
    attempt(i + 1);
  };
  //...略
})(0);
```

attempt 関数を括弧で囲んだ後、`(0)`を記載することで定義した attempt 関数に引数 0 を設定して実行しています。
そして、fail メソッドが呼ばれた時のみ再帰的に attempt 関数を実行しています。
これによって、複数のストテラジーによる検証を可能としており、どれか一つでも検証が成功すれば処理を成功とする機能を可能とします。
複数のストテラジーによる検証ができる仕組みは分かったのですが、そもそも登録したストラテジーの呼び出し方が分からない状態です。
実は上記のことを行っているのが、下記の部分です。

```tsx
var layer = name[i];
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

引数で設定した name がオブジェクトで、authenticate メソッドを持っていれば、そのままストラテジーとして定義し、そうでない場合はストラテジーのキーなので、\_strategy メソッドにキーを渡し、予め登録したストラテジーを生成しています。
ストラテジーを予め登録しておく処理については、以前書いた Strategy の記事がありますので、そちらを参照してください。
すごいザックリいうと、こちらのファイルで[ストラテジー](https://github.com/nestjs/passport/blob/master/lib/passport/passport.strategy.ts)を登録する処理があるのですが、これを呼び出すことで登録ができます。
最後に残りの処理をみていきます。

```tsx
strategy.success = function (user, info) {
  if (callback) {
    return callback(null, user, info);
  }
  //...略
};

strategy.fail = function (challenge, status) {
  if (typeof challenge == "number") {
    status = challenge;
    challenge = undefined;
  }

  // push this failure into the accumulator and attempt authentication
  // using the next strategy
  failures.push({ challenge: challenge, status: status });
  attempt(i + 1);
};

strategy.error = function (err) {
  if (callback) {
    return callback(err);
  }
  //...略
};

strategy.authenticate(req, options);
```

fail メソッドの主要な役割はすでに解説していますし、success メソッドと error メソッドはコールバック関数に値を渡して実行しているだけです。
一応 success メソッドを実行した時は、第二引数に検証が成功して取得できた値を格納しています。
とはいえ、上記 3 つのメソッドはパッと見で理解できます。
ただ、最後の `strategy.authenticate(req, options);`は突然出てきました。
こちらですが、passport モジュールがサポートするストラテジーは基本的に autheticate メソッドを有していることが前提となっています。
なので、ここで突然呼び出していても実行が可能となっています。
全ては確認していませんが、実際[passport-local](https://github.com/jaredhanson/passport-local/blob/master/lib/strategy.js)や[passport-jwt](https://github.com/mikenicholson/passport-jwt/blob/master/lib/strategy.js)のストラテジーは autheticate メソッドが定義されています。
後は、各自で登録したストラテジーの authenticate が実行され、ストラテジー独自に実装された検証を行い、問題が無ければ success メソッドを実行して、取得した値を返すようにします。
エラーが発生した時は、error メソッドを実行し、検証に失敗した時は fail メソッドを実行します。
大分流れが鮮明に見えました。
以上が createPassportContext 関数周りの仕組みとなります。

## おわりに

今回は AuthGuard について詳しくみていきました。
中々追うのは大変でしたが、知れば知るほどガードとストラテジーというのはよくできているなと感じます。
これで NestJS と passport ストラテジーの関係性は一通りつかめた思うので、今度認証・認可系の情報を扱う時は真っ先に選択肢として挙げてしまいそうです。
達成感いっぱいなのでここで終わらせます。
読んでいただきありがとうございました。
