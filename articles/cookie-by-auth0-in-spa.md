---
title: "SPAでAuth0のライブラリを使ってログインした際にセットされるCookieについて"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Auth0","SPA","Cookie","Javascript"]
published: true
---
## この記事の内容について
正直、[Auth0のコミュニティサイト](https://community.auth0.com/t/auth0-is-authenticated-cookie-legacy-auth0-is-authenticated-cookie/108219)で展開された内容を読めば十分です。
上記リンクの内容を自分で改めてまとめたにすぎません。
## はじめに
SPAでAuth0を使用し、ログインを行うと以下画像のようなCookieがセットされます。
![image1.png](/images/cookie-by-auth0-in-spa/image1.png)
何かしら使用されていそうですが、実際に何に使用しているかは分かっていませんでした。
そこで、今回はauth0から始まるCookieが何のために存在しているかについて、まとめた記事です。
## 結論
- 当該Cookieはブラウザ上でログインしているかを検証するフラグ的な役割です
    - そのため、盗まれても特に不都合はありません
- トークンを取得する時や、checkSession関数を呼び出す際にセットされます
- Auth0のライブラリの際は基本的に活用されています
- ただし、インメモリに保存されているリフレッシュトークンを使用して、アクセストークンを取得している場合は活用されていなさそうです

## auth0.is.authenticated Cookieについて
auth0.is.authenticated Cookieは[Auth0のコミュニティサイト](https://community.auth0.com/t/auth0-is-authenticated-cookie-legacy-auth0-is-authenticated-cookie/108219)で記載されているように、ログインを一から行うことを避けるためのフラグ的な役割で使用しています。
そのため、ブラウザ側でCookieを設定しています。
上記のような役割だとAuth0も言っていますし、Cookieの名前は「auth0.clientId.is.authenticated」となっています。
誰でも簡単に取得できるものなので、このCookieを機微な情報を扱うためのキーにすることは向いていません。
## auth0.is.authenticated Cookieに関わるライブラリの実装を追う
[@auth0/auth0-spa-js](https://github.com/auth0/auth0-spa-js)を元に確認します。
まずCookieを扱っているのは、[CookieStorage](https://github.com/auth0/auth0-spa-js/blob/main/src/storage.ts#L20)オブジェクトです。このオブジェクトでCookieの設定や取得を行っています。
次に、「はじめに」でみたCookie名を取得しているところです。これは以下の[isAuthenticatedCookieName](https://github.com/auth0/auth0-spa-js/blob/main/src/Auth0Client.ts#L187)で設定しています。
```tsx
this.isAuthenticatedCookieName = buildIsAuthenticatedCookieName(
    this.options.clientId
);
```
[buildIsAuthenticatedCookieName関数](https://github.com/auth0/auth0-spa-js/blob/main/src/Auth0Client.utils.ts#L29)は以下のように、Auth0内のclientIdを元に文字列を作成しています：
```tsx
export const buildIsAuthenticatedCookieName = (clientId: string) =>
  `auth0.${clientId}.is.authenticated`;
```
Cookieを扱うオブジェクトとCookieのキーを作成している部分を確認できたので、実際にそれらを呼び出している場面を確認します。詳細はAuth0Client.tsファイルを追ってもらえばわかるので、ここでは省略版を記載します：
```tsx
export class Auth0Client {
  private readonly cookieStorage: ClientStorage;
  private readonly isAuthenticatedCookieName: string;
  
  constructor(options: Auth0ClientOptions) {
    this.cookieStorage =
      options.legacySameSiteCookie === false
        ? CookieStorage
        : CookieStorageWithLegacySameSite;
    this.isAuthenticatedCookieName = buildIsAuthenticatedCookieName(
      this.options.clientId
    );
    this.sessionCheckExpiryDays =
      options.sessionCheckExpiryDays || DEFAULT_SESSION_CHECK_EXPIRY_DAYS;
  }
	/**
	* Cookieの有無で、Cookieとトークンを更新するメソッド
	* 生でAuth0Clientクラスを呼び出さない限りは、大抵呼ばれています
	* 実際、@auth0/auth0-react,@auth0/auth0-vue両方とも初回アクセス時に呼ばれる設定になっています。
	*/
  public async checkSession(options?: GetTokenSilentlyOptions) {
    if (!this.cookieStorage.get(this.isAuthenticatedCookieName)) {
      if (!this.cookieStorage.get(OLD_IS_AUTHENTICATED_COOKIE_NAME)) {
        return;
      } else {
        // Migrate the existing cookie to the new name scoped by client ID
        this.cookieStorage.save(this.isAuthenticatedCookieName, true, {
          daysUntilExpire: this.sessionCheckExpiryDays,
          cookieDomain: this.options.cookieDomain
        });
        this.cookieStorage.remove(OLD_IS_AUTHENTICATED_COOKIE_NAME);
      }
    }
    try {
			// トークン更新処理
      await this.getTokenSilently(options);
    } catch (_) {}
  }
  /**
	* ログアウト時は、認証していることを示すフラグを削除したいので、削除処理を実行している
	*/
  public async logout(options: LogoutOptions = {}): Promise<void> {
    /** 略 */
    this.cookieStorage.remove(this.isAuthenticatedCookieName, {
      cookieDomain: this.options.cookieDomain
    });
    /** 略 */
  }
	/**
	* Iframe経由でアクセストークンを取得するメソッド
	*/
  private async _getTokenFromIFrame(
    options: GetTokenSilentlyOptions & {
      authorizationParams: AuthorizationParams & { scope: string };
    }
  ): Promise<GetTokenSilentlyResult> {
    /** 略 */
      const tokenResult = await this._requestToken(
        {
          ...options.authorizationParams,
          code_verifier,
          code: codeResult.code as string,
          grant_type: 'authorization_code',
          redirect_uri,
          timeout: options.authorizationParams.timeout || this.httpTimeoutMs
        },
        {
          nonceIn,
          organization: params.organization
        }
      );
  }
  /**
	* リフレッシュトークン経由でアクセストークンを取得するメソッド
	*/
  private async _getTokenUsingRefreshToken(
    options: GetTokenSilentlyOptions & {
      authorizationParams: AuthorizationParams & { scope: string };
    }
  ): Promise<GetTokenSilentlyResult> {
      const tokenResult = await this._requestToken({
        ...options.authorizationParams,
        grant_type: 'refresh_token',
        refresh_token: cache && cache.refresh_token,
        redirect_uri,
        ...(timeout && { timeout })
      });
      /** 略 */
  }
	/**
	* アクセストークンを取得するリクエストを行うメソッド
	* ここでリクエストに成功できれば、OpenID Connectのフローは完了なので、認証したことを示すフラグを設定している
	*/
  private async _requestToken(
    options: PKCERequestTokenOptions | RefreshTokenRequestTokenOptions,
    additionalParameters?: RequestTokenAdditionalParameters
  ) {
    /** 略 */
    this.cookieStorage.save(this.isAuthenticatedCookieName, true, {
      daysUntilExpire: this.sessionCheckExpiryDays,
      cookieDomain: this.options.cookieDomain
    });
    /** 略 */
  }
```
凄くざっくりいうと、ログアウト・ログイン※時に追加・削除され、トークンを更新する便利メソッドを実行するかの判定に使用されます。
※ログイン時ではなく、認可コードを元にアクセストークンを取得する際にCookieは設定されます。ただ、イメージしやすいようにログイン時と記載しました。
## auth0.is.authenticated Cookieの活用方法
※あくまでソースコードを読んだ推測になります。Auth0の想定通りかは定かではないので、その点はご注意ください。
セットしたCookieを使用するのはcheckSession関数のみです。
なので、checkSession関数がどの場面で呼び出されるかを理解すれば良さそうです。
まず、checkSession関数はトークンの更新ができるので、明示的に呼び出すケースが考えられます。
また、Auth0を使用するための初期設定として使用する以下のコードには必ず呼び出されています：
- [@auth0/auth0-spa-jsのcreateAuth0Client関数](https://github.com/auth0/auth0-spa-js/blob/main/src/index.ts#L19)
- [@auth0/auth0-reactのAuth0Providerコンポーネント](https://github.com/auth0/auth0-react/blob/6429fdd31959548906a2ee1ecc8ec4d3a137ebc3/src/auth0-provider.tsx#L162)
- [@auth0/auth0-vueのAuth0Pluginクラス](https://github.com/auth0/auth0-vue/blob/main/src/plugin.ts#L89)

そのため、ライブラリを使っていれば、基本的にはアプリケーションにアクセスしたタイミングでCookieを参照して、存在していればトークンの更新が行われています。
このことから、[Auth0のコミュニティサイト](https://community.auth0.com/t/auth0-is-authenticated-cookie-legacy-auth0-is-authenticated-cookie/108219)で記載があったように、[loginWithRedirect](https://github.com/auth0/auth0-spa-js/blob/main/src/Auth0Client.ts#L450)のような1からログインの流れを行う処理を呼び出さなくても、最新のトークンを取得できることが分かります：
> It is just an optimization cookie that minimizes unnecessary log-ins for a better user experience.
> 

以上から、Auth0のライブラリを使用している時点で、auth0.is.authenticated Cookieは活用しており、プラスαで活用したい場合はcheckSession関数を呼び出せばよいことが分かります。
## auth0.is.authenticated Cookieの活用の注意点
先程Cookieの活用場面を見ていきました。その時にライブラリを使用している時点で、活用されているという記載をしました。上記記載は誤りではないのですが、注意点はあります。
それは、リフレッシュトークンを使用してトークンを取得する設定にしており、各種トークンをインメモリに保存している場合はcheckSession関数を活用していない点です。
インメモリに保存していると、画面を更新するたびにトークンは無くなります。
なので、リフレッシュトークンも存在しません。
となると、ライブラリで初回アクセス時に呼び出されるcheckSession関数は、処理を完了しません。
checkSession関数の実装は以下のようにtry-catchをしているので、エラーにはなりませんが本来の処理である、トークンの更新はできません：
```tsx
public async checkSession(options?: GetTokenSilentlyOptions) {
  //...略
  try {
    // トークン更新処理
    await this.getTokenSilently(options);
  } catch (_) {}
}
```
このようにリフレッシュトークンをインメモリに保存している場合は、上手く活用できていないという点はご注意ください。
ただし、以下の場合は初回アクセス時でもcheckSession関数は上手く動くと思います：
- リフレッシュトークンをストレージなどに保存している場合
- そもそもリフレッシュトークンではなく、iframeを使用してトークンの更新を行っている場合

## おわりに
今回はAuth0が設定するauth0.is.authenticated Cookieについて見ていきました。
Cookieを使用しているので、何か重要なことをしているのではないかと思ったのですが、やっていることはフラグ管理の機能くらいだったのは驚きでした。
ただ、汎用的にかつタブ間で状態を共有するには、Cookieの方がしやすいなとも思い、なるほどなと思いました。
まだまだ分かってない機能があるので、どこかのタイミングで調べられたなと思います。
ここまで読んでいただきありがとうございました。