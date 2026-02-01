---
title: "node-jsonwebtokeの検証処理についての流れを確認する"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JWT", "Auth0", "Node", "jsonwebtoken"]
published: true
---

## はじめに

開発の都合上、[node-jwonwebtoke](https://github.com/auth0/node-jsonwebtoken/tree/master)の verify 部分の処理について理解する必要がありました。
今回は verify の役割を担う、[verify.js](https://github.com/auth0/node-jsonwebtoken/blob/master/verify.js)の処理についてみていきます。
でははじめます。

## jsonwebtoken の verify.js の流れ

jsonwebtoken の verify.js は、JWT 形式のトークンの検証を行うための中心的な機能を提供しています。
このファイルでは、与えられた JWT が有効であることを確認するために、様々なチェックが行われます。
以下では、verify.js の全体的な流れを大まかな構成に分けて説明します。

### 全体の大まかな構成

**コールバックの設定**
まず以下の部分で引数で受け取ったコールバックを設定しています。

```tsx
if (typeof options === "function" && !callback) {
  callback = options;
  options = {};
}
if (!options) {
  options = {};
}
//clone this object since we are going to mutate it.
options = Object.assign({}, options);
let done;
if (callback) {
  done = callback;
} else {
  done = function (err, data) {
    if (err) throw err;
    return data;
  };
}
```

**オプションの設定のチェック**
以下のコード部分で、トークンの検証オプションが適切な形で定義されているかを確認しています。

```tsx
if (options.clockTimestamp && typeof options.clockTimestamp !== "number") {
  return done(new JsonWebTokenError("clockTimestamp must be a number"));
}
if (
  options.nonce !== undefined &&
  (typeof options.nonce !== "string" || options.nonce.trim() === "")
) {
  return done(new JsonWebTokenError("nonce must be a non-empty string"));
}
if (
  options.allowInvalidAsymmetricKeyTypes !== undefined &&
  typeof options.allowInvalidAsymmetricKeyTypes !== "boolean"
) {
  return done(
    new JsonWebTokenError("allowInvalidAsymmetricKeyTypes must be a boolean")
  );
}
const clockTimestamp = options.clockTimestamp || Math.floor(Date.now() / 1000);
```

不正な設定にすると、検証の時に本来適切なトークンであってもエラーが発生してしまい、エラーの原因も分かりにくくなるので、予めここではじいていると推測します。
**トークンの形式チェック**
該当コードは以下の部分です。

```tsx
if (!jwtString) {
  return done(new JsonWebTokenError("jwt must be provided"));
}
if (typeof jwtString !== "string") {
  return done(new JsonWebTokenError("jwt must be a string"));
}
const parts = jwtString.split(".");
if (parts.length !== 3) {
  return done(new JsonWebTokenError("jwt malformed"));
}
```

与えられた JWT が正しい形式であることを確認します。
JWT は、ヘッダー、ペイロード、シグネチャの 3 つの部分からなる文字列で、それぞれがドット（.）で区切られています。
トークンを分割し、3 つの部分で構成されていることを確認します。
jsonwebtoken は基本的に[JWS](https://tex2e.github.io/rfc-translater/html/rfc7515.html)の検証を行うトークンだとコードから推測できます。
なので、JWS 形式でないものは必ず検証がエラーになるので、早めにエラーを返すようにしています
**JWT 部分のデコード**

```jsx
let decodedToken;
try {
  decodedToken = decode(jwtString, { complete: true });
} catch (err) {
  return done(err);
}
if (!decodedToken) {
  return done(new JsonWebTokenError("invalid token"));
}
const header = decodedToken.header;
```

decode 関数でトークンをデコードして、JWS のヘッダーとペイロード部分を取得しています。
[decode 関数のコード](https://github.com/auth0/node-jsonwebtoken/blob/master/decode.js#L22C1-L28C4)を見ると、以下のように complete プロパティを true にするとトークンのすべての項目を返すようにしています。

```tsx
if (options.complete === true) {
  return {
    header: decoded.header,
    payload: payload,
    signature: decoded.signature,
  };
}
```

今回は検証で使用するために、全てを取得した中から header 部分をひとまず抽出しています。
署名の検証用の関数

```tsx
let getSecret;
if (typeof secretOrPublicKey === "function") {
  if (!callback) {
    return done(
      new JsonWebTokenError(
        "verify must be called asynchronous if secret or public key is provided as a callback"
      )
    );
  }
  getSecret = secretOrPublicKey;
} else {
  getSecret = function (header, secretCallback) {
    return secretCallback(null, secretOrPublicKey);
  };
}
```

**署名の検証用関数の作成**
以下の部分で、署名の検証用の関数を定義します。
この時、鍵そのものを渡した場合はその鍵を使用した関数を作成しますし、関数を渡すこともできます。
ただ、関数を渡す場合はドキュメントにもあるように第一引数がトークンのヘッダー部分に該当するものを渡し、第二引数では callback 関数を設定する必要があります。

```tsx
function getKey(header, callback) {
  client.getSigningKey(header.kid, function (err, key) {
    var signingKey = key.publicKey || key.rsaPublicKey;
    callback(null, signingKey);
  });
}
```

なお、varify メソッドは callback 関数内で署名の検証をしているのと、getKey 関数を最後に実行するようになっています。
なので、上記 getKey 関数内で callback を呼ばないと適切な検証がされない可能性があります。
この後は verify.js 内の getKey 関数の callback 部分の処理をみていきます。
**署名の有無と検証鍵の有無の判定**
以下の部分では署名の有無と鍵の有無による判定をしています。

```tsx
    const hasSignature = parts[2].trim() !== '';
    if (!hasSignature && secretOrPublicKey){
      return done(new JsonWebTokenError('jwt signature is required'));
    }
    if (hasSignature && !secretOrPublicKey) {
      return done(new JsonWebTokenError('secret or public key must be provided'));
    }
    if (!hasSignature && !options.algorithms) {
      return done(new JsonWebTokenError('please specify "none" in "algorithms" to verify unsigned tokens'));
    }、JWTの署名の有無と、secretOrPublicKeyの提供状況に基づいて、エラーチェックを行っています。
```

それぞれの処理を見ていきましょう。
まず、`const hasSignature = parts[2].trim() !== '';`という行では、トークンの 3 番目の部分（署名部分）が空ではないかを確認しています。
`parts[2]`は、JWT をドット（.）で分割した結果の 3 番目の要素を指し、`trim()`メソッドを用いて署名部分の前後の空白文字を取り除いています。
署名部分が空でなければ、`hasSignature`は`true`になります。
次に、`if (!hasSignature && secretOrPublicKey)`という条件では、JWT に署名がないのに`secretOrPublicKey`が提供されている場合、エラーを発生させます。
これは、署名なしの JWT を検証するための鍵が提供されているという矛盾を示しています。
その後、`if (hasSignature && !secretOrPublicKey)`という条件では、JWT に署名があるにも関わらず、`secretOrPublicKey`が提供されていない場合にエラーを発生させます。
署名付きの JWT を検証するためには、鍵が必要です。
鍵が提供されていなければ、検証は不可能です。
最後に、`if (!hasSignature && !options.algorithms)`という条件では、JWT に署名がない上に`options.algorithms`が指定されていない場合にエラーを発生させます。
署名なしの JWT を検証する際には、`options.algorithms`に`'none'`を指定する必要があります。
これによって、署名なしの JWT を明示的に許可することが可能になります。
ただし、`algorithms`に`'none'`が指定されている場合は、署名がなくてもエラーは発生しません。この際、JWT の検証は署名の検証をスキップし、その他のクレームの検証のみを行います。
これらのチェックは、JWT の署名の有無と、検証に必要な鍵の提供状況が適切であることを確認するために行われます。
署名なしの JWT を許可する場合には、`options.algorithms`に`'none'`を明示的に指定する必要があります。
**鍵の作成**
以下の部分で引数で受け取った鍵を crypto の KeyObject に変換しています。

```tsx
if (secretOrPublicKey != null && !(secretOrPublicKey instanceof KeyObject)) {
  try {
    secretOrPublicKey = createPublicKey(secretOrPublicKey);
  } catch (_) {
    try {
      secretOrPublicKey = createSecretKey(
        typeof secretOrPublicKey === "string"
          ? Buffer.from(secretOrPublicKey)
          : secretOrPublicKey
      );
    } catch (_) {
      return done(
        new JsonWebTokenError("secretOrPublicKey is not valid key material")
      );
    }
  }
}
```

もともと KeyObject の時は特に何もしません。
内部の処理としては同じく crypto の createPublicKey 関数を使い、公開鍵をもとにした KeyObject を作ろうとします。
それがエラーだったら、受け取った鍵は共有鍵である可能性が高いので、createSecretKey 関数をつかい KeyObject を作ります。
それすらエラーだったら、鍵として適切ではないので、エラーが発生します。
ちなみに KeyObject とは、Node.js の暗号化 API で使用される鍵を表すオブジェクトです。こ
れは、公開鍵、秘密鍵、または HMAC キーを表すために使用されます。
KeyObject は、以下のようなプロパティを持ちます。

1. type: 鍵のタイプを表す文字列です。以下のいずれかの値を取ります。
   - 'secret': 対称鍵暗号化アルゴリズム（HMAC など）で使用される秘密鍵を表します。
   - 'public': 非対称鍵暗号化アルゴリズム（RSA や ECDSA など）で使用される公開鍵を表します。
   - 'private': 非対称鍵暗号化アルゴリズム（RSA や ECDSA など）で使用される秘密鍵を表します。
2. asymmetricKeyType: 非対称鍵のタイプを表す文字列です。以下のいずれかの値を取ります。
   - 'rsa': RSA 鍵ペアを表します。
   - 'dsa': DSA 鍵ペアを表します。
   - 'ec': 楕円曲線（EC）鍵ペアを表します。
   - 'ed25519': Ed25519 鍵ペアを表します。
   - 'ed448': Ed448 鍵ペアを表します。
   - 'x25519': X25519 鍵ペアを表します。
   - 'x448': X448 鍵ペアを表します。
3. symmetricKeySize: 対称鍵のサイズをビット単位で表す数値です。
4. publicKeyEncoding: 公開鍵のエンコーディング形式を指定するオブジェクトです。
   - type: エンコーディングタイプを表す文字列です（'pkcs1'、'spki'など）。
   - format: エンコーディングフォーマットを表す文字列です（'pem'、'der'など）。
5. privateKeyEncoding: 秘密鍵のエンコーディング形式を指定するオブジェクトです。
   - type: エンコーディングタイプを表す文字列です（'pkcs1'、'pkcs8'など）。
   - format: エンコーディングフォーマットを表す文字列です（'pem'、'der'など）。

KeyObject は、PEM 形式や JWK 形式の鍵材料から作成することができます。
今回使用されている createPublicKey 関数や createSecretKey 関数、または createPrivateKey 関数などの関数を使って、鍵材料から作ることができます。
KeyObject については合っているか自信ないので、ここが違うよなど分かる方がいらっしゃいましたらコメントもらえると幸いです。
**アルゴリズムの決定と判定**
署名を生成する時に使用したアルゴリズムを取得し、判定しています。

```tsx
    if (!options.algorithms) {
      if (secretOrPublicKey.type === 'secret') {
        options.algorithms = HS_ALGS;
      } else if (['rsa', 'rsa-pss'].includes(secretOrPublicKey.asymmetricKeyType)) {
        options.algorithms = RSA_KEY_ALGS
      } else if (secretOrPublicKey.asymmetricKeyType === 'ec') {
        options.algorithms = EC_KEY_ALGS
      } else {
        options.algorithms = PUB_KEY_ALGS
      }
    }

		if (options.algorithms.indexOf(decodedToken.header.alg) === -1) {
      return done(new JsonWebTokenError('invalid algorithm'));
    }
    if (header.alg.startsWith('HS') && secretOrPublicKey.type !== 'secret') {
      return done(new JsonWebTokenError((`secretOrPublicKey must be a symmetric key when using ${header.alg}`)))
    } else if (/^(?:RS|PS|ES)/.test(header.alg) && secretOrPublicKey.type !== 'public') {
      return done(new JsonWebTokenError((`secretOrPublicKey must be an asymmetric key when using ${header.alg}`)))

```

`secretOrPublicKey`の種類に応じて、異なるアルゴリズムのセットが`options.algorithms`に割り当てられます。

- 対称鍵（秘密鍵）の場合は、HMAC ベースのアルゴリズム（HS256、HS384、HS512）のセットが選択されます。
- RSA 公開鍵の場合は、RSA ベースのアルゴリズム（RS256、RS384、RS512）のセットが選択されます。
- 楕円曲線公開鍵の場合は、楕円曲線ベースのアルゴリズム（ES256、ES384、ES512）のセットが選択されます。
- その他の非対称鍵（公開鍵）の場合は、全ての公開鍵ベースのアルゴリズムのセットが選択されます。

この処理により、`options.algorithms`が指定されていない場合でも、`secretOrPublicKey`の種類に応じて適切なアルゴリズムのセットが自動的に選択されます。
アルゴリズムが決定したら、鍵の種類と署名を生成した時のアルゴリズムが一致するかを判定し、一致しない場合はエラーを発生させます。
**アルゴリズムの判定 Part2**
以下部分で、署名を付与した時に使用したアルゴリズムと鍵の種類が問題ないかを確認しています。

```tsx
if (!options.allowInvalidAsymmetricKeyTypes) {
  try {
    validateAsymmetricKey(header.alg, secretOrPublicKey);
  } catch (e) {
    return done(e);
  }
}
```

ここで分岐を作っているのは下位互換に対応するため行っており、原則この分岐は通ることが推奨されています。([参考部分](https://github.com/auth0/node-jsonwebtoken/tree/master#:~:text=allowInvalidAsymmetricKeyTypes%3A%20true%20%E3%81%AE%E5%A0%B4%E5%90%88%E3%80%81%E6%8C%87%E5%AE%9A%E3%81%95%E3%82%8C%E3%81%9F%E3%82%A2%E3%83%AB%E3%82%B4%E3%83%AA%E3%82%BA%E3%83%A0%E3%81%AB%E4%B8%80%E8%87%B4%E3%81%97%E3%81%AA%E3%81%84%E9%9D%9E%E5%AF%BE%E7%A7%B0%E3%82%AD%E3%83%BC%E3%82%92%E8%A8%B1%E5%8F%AF%E3%81%97%E3%81%BE%E3%81%99%E3%80%82%E3%81%93%E3%81%AE%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AF%E4%B8%8B%E4%BD%8D%E4%BA%92%E6%8F%9B%E6%80%A7%E3%81%AE%E3%81%9F%E3%82%81%E3%81%A0%E3%81%91%E3%81%AB%E7%94%A8%E6%84%8F%E3%81%95%E3%82%8C%E3%81%A6%E3%81%8A%E3%82%8A%E3%80%81%E4%BD%BF%E7%94%A8%E3%81%AF%E9%81%BF%E3%81%91%E3%81%A6%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82))
このことから、jsonwebtoken としては[validateAsymmetricKey 関数](https://github.com/auth0/node-jsonwebtoken/blob/master/lib/validateAsymmetricKey.js)で使用しているアルゴリズムを推奨していることが推測できます。
ここでコードの解説はしませんが気になる方は確認してみてください。
**署名の検証**
JWT に付与されている署名を jws.verify 関数を使って検証します。

```jsx
let valid;
try {
  valid = jws.verify(jwtString, decodedToken.header.alg, secretOrPublicKey);
} catch (e) {
  return done(e);
}
if (!valid) {
  return done(new JsonWebTokenError("invalid signature"));
}
```

実は jsonwebtoken 自体は検証を行っておらず、jws モジュールのラッパーライブラリだということがここで分かります。
jws からさらに掘って行きたいですが、マリアナ海溝が待ち構えている気がするので、この記事ではやらないです。
**payload の検証**
この部分のコードは、JWT のペイロードに含まれる各種クレームの検証を行っています。

```tsx
const payload = decodedToken.payload;
if (typeof payload.nbf !== "undefined" && !options.ignoreNotBefore) {
  if (typeof payload.nbf !== "number") {
    return done(new JsonWebTokenError("invalid nbf value"));
  }
  if (payload.nbf > clockTimestamp + (options.clockTolerance || 0)) {
    return done(
      new NotBeforeError("jwt not active", new Date(payload.nbf * 1000))
    );
  }
}
if (typeof payload.exp !== "undefined" && !options.ignoreExpiration) {
  if (typeof payload.exp !== "number") {
    return done(new JsonWebTokenError("invalid exp value"));
  }
  if (clockTimestamp >= payload.exp + (options.clockTolerance || 0)) {
    return done(
      new TokenExpiredError("jwt expired", new Date(payload.exp * 1000))
    );
  }
}
if (options.audience) {
  const audiences = Array.isArray(options.audience)
    ? options.audience
    : [options.audience];
  const target = Array.isArray(payload.aud) ? payload.aud : [payload.aud];
  const match = target.some(function (targetAudience) {
    return audiences.some(function (audience) {
      return audience instanceof RegExp
        ? audience.test(targetAudience)
        : audience === targetAudience;
    });
  });
  if (!match) {
    return done(
      new JsonWebTokenError(
        "jwt audience invalid. expected: " + audiences.join(" or ")
      )
    );
  }
}
if (options.issuer) {
  const invalid_issuer =
    (typeof options.issuer === "string" && payload.iss !== options.issuer) ||
    (Array.isArray(options.issuer) &&
      options.issuer.indexOf(payload.iss) === -1);
  if (invalid_issuer) {
    return done(
      new JsonWebTokenError("jwt issuer invalid. expected: " + options.issuer)
    );
  }
}
if (options.subject) {
  if (payload.sub !== options.subject) {
    return done(
      new JsonWebTokenError("jwt subject invalid. expected: " + options.subject)
    );
  }
}
if (options.jwtid) {
  if (payload.jti !== options.jwtid) {
    return done(
      new JsonWebTokenError("jwt jwtid invalid. expected: " + options.jwtid)
    );
  }
}
if (options.nonce) {
  if (payload.nonce !== options.nonce) {
    return done(
      new JsonWebTokenError("jwt nonce invalid. expected: " + options.nonce)
    );
  }
}
if (options.maxAge) {
  if (typeof payload.iat !== "number") {
    return done(new JsonWebTokenError("iat required when maxAge is specified"));
  }
  const maxAgeTimestamp = timespan(options.maxAge, payload.iat);
  if (typeof maxAgeTimestamp === "undefined") {
    return done(
      new JsonWebTokenError(
        '"maxAge" should be a number of seconds or string representing a timespan eg: "1d", "20h", 60'
      )
    );
  }
  if (clockTimestamp >= maxAgeTimestamp + (options.clockTolerance || 0)) {
    return done(
      new TokenExpiredError("maxAge exceeded", new Date(maxAgeTimestamp * 1000))
    );
  }
}
```

最初に、デコードされたトークンのペイロード（`payload`）を取得します。
`payload.nbf`（Not Before）クレームが存在し、`options.ignoreNotBefore`が`false`の場合、`nbf`の値が数値であることを確認し、現在のタイムスタンプ（`clockTimestamp`）と比較します。
`nbf`が現在のタイムスタンプよりも大きい場合は、トークンがまだアクティブではないことを示す`NotBeforeError`が返されます。
`payload.exp`（Expiration Time）クレームが存在し、`options.ignoreExpiration`が`false`の場合、`exp`の値が数値であることを確認し、現在のタイムスタンプと比較します。
現在のタイムスタンプが`exp`以上の場合は、トークンの有効期限が切れていることを示す`TokenExpiredError`が返されます。
`options.audience`が指定されている場合、`payload.aud`（Audience）クレームの値と比較します。
`options.audience`が配列の場合は、`payload.aud`も配列として扱われます。
`payload.aud`の各要素に対して、`options.audience`の各要素との一致を確認します。
`options.issuer`が指定されている場合、`payload.iss`（Issuer）クレームの値と比較します。`options.issuer`が文字列の場合は、`payload.iss`と完全一致するかどうかを確認します。`options.issuer`が配列の場合は、`payload.iss`がその配列に含まれているかどうかを確認します。
これによって、トークンの発行者が想定と一致するかを判定できます。
`options.subject`が指定されている場合、`payload.sub`（Subject）クレームの値と比較します。
これによって、対象トークンの識別子が想定と一致するかを判定できます。
`options.jwtid`が指定されている場合、`payload.jti`（JWT ID）クレームの値と比較します。
`options.nonce`が指定されている場合、`payload.nonce`（Nonce）クレームの値と比較します。
`options.maxAge`が指定されている場合、`payload.iat`（Issued At）クレームが数値であることを確認します。`maxAge`を使用して、トークンの最大有効期間を計算します。
`maxAgeTimestamp`が`undefined`の場合は、`maxAge`の値が不正であることを示す`JsonWebTokenError`が返されます。
現在のタイムスタンプが`maxAgeTimestamp`以上の場合は、トークンの最大有効期間が超過したことを示す`TokenExpiredError`が返されます。
これらの検証により、JWT のペイロードに含まれるクレームが、指定されたオプションや期待される値と一致していることを確認します。
**全部返すか・一部返すか**
最後にフラグによって、ヘッダー・デコードしたペイロード・署名を全部返すか、ペイロード部分だけ返すかを決めます。

```tsx
if (options.complete === true) {
  const signature = decodedToken.signature;
  return done(null, {
    header: header,
    payload: payload,
    signature: signature,
  });
}
return done(null, payload);
```

これで一通りの流れは全てみました。
色々処理はありますが、大きく分けると以下の 4 つに分けれるのかなと思います。
① オプションやトークンの形式チェック
② トークンのデコード
③ 署名の検証
④ ペイロードの検証
この構造を意識してみれば、今自分は何に注目したいかが分かるような気がしています。
