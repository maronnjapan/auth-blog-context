---
title: ""
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## CIBAとは
CIBAはOpenID Connect Client-Initiated Backchannel Authenticationの略。
Relying PartyがOpenID Providerで直接ユーザーを認証する際の仕様
直接バックエンドでやり取りをする。
Authorization Code Flowとの差異は、ブラウザでのリダイレクトが不要
登場人物として、CIBAを開始する端末であるConsumption Deviceとユーザーが認証する端末であるAuthentication Deviceが存在する。
フローとしては以下の通り
1. Relying Partyが対象のユーザーを特定し、OpenID ProviderのBackchannel Authentication EndpointにCIBAの開始リクエストを送る。
2. OPはユーザーを例えばスマホのプッシュ通知といったOut Of Bandな方法でユーザー認証を行う
## CIBAが解決すること
CIBAは以下のケースに対応している
- ブラウザのリダイレクト処理ができない端末での認証
- UI機能が乏しく安全なログインができない端末でのログイン
- 以下のような
    - コールセンターでの使用：コールセンターの人が電話先のユーザーの人が承認することで認証を完了する
    - 顔合わせでの認証：銀行行員が、顧客のスマホに承認リクエストを送ることで本人確認を完了とする
    - 支払いを承認を対象ユーザーのスマホで行うことができる
## CIBAのリクエストモデル
CIBAはそれぞれ以下の3つのパターンでトークンのリクエストを行う
- Poll
- Ping
- Push




## CIBA三つのモード(Poll,Ping,Push)
Pollモード：クライアントが一定間隔で、grant typeが`urn:openid:params:grant-type:ciba`で、トークンエンドポイントへリクエストを行う。レスポンスはまだユーザー認証が完了していない場合は`authorization_pending error`を返し、認証が成功したらアクセストークンなどのいわゆるトークンエンドポイントが成功した時に返却される値が取得できる。30秒くらいはCIBAのフロー実行後、対象のトークンリクエストを受け付けるようにすることが求められる。

Pingモード：OPが結果を渡す準備ができたら、クライアント側にPostリクエストで通知を行う。そして、クライアント側は通知を受け取ったらトークンエンドポイントをリクエストし、アクセストークンなどを取得する。

Pushモード：OPがクライアントに直接トークンを渡す。すなわち、クライアント側からOPへトークンエンドポイントへリクエストすることは求めず、認証が完了した時点でアクセストークンなどをクライアントに渡す。渡すエンドポイントは通知用のエンドポイント。このモードを使用するときは、想定通りのトークンを渡していることを担保するためにat_hashとauth_req_idをIDトークンの中に含める必要がある。

### Pollモードを適用するのが好ましいケース
Pollモードは以下の利点がある
- ネットワークやアーキテクチャの制限ない
- クライアントの事前登録も不要（`backchannel_client_notification_endpoint`の事前登録など）

### Auth0の対応状況
`https://{Auth0テナントのドメイン}/.well-known/openid-configuration`でアクセスすると以下のようなJSONを取得でき、CIBAのモードが確認できる`backchannel_token_delivery_modes_supported`は`["poll"]`となっており、pollモードのみ対応していることがわかる。
```json
{"issuer":"https://{Auth0テナントのドメイン}/","authorization_endpoint":"https://{Auth0テナントのドメイン}/authorize","token_endpoint":"https://{Auth0テナントのドメイン}/oauth/token","device_authorization_endpoint":"https://{Auth0テナントのドメイン}/oauth/device/code","userinfo_endpoint":"https://{Auth0テナントのドメイン}/userinfo","mfa_challenge_endpoint":"https://{Auth0テナントのドメイン}/mfa/challenge","jwks_uri":"https://{Auth0テナントのドメイン}/.well-known/jwks.json","registration_endpoint":"https://{Auth0テナントのドメイン}/oidc/register","revocation_endpoint":"https://{Auth0テナントのドメイン}/oauth/revoke","scopes_supported":["openid","profile","offline_access","name","given_name","family_name","nickname","email","email_verified","picture","created_at","identities","phone","address"],"response_types_supported":["code","token","id_token","code token","code id_token","token id_token","code token id_token"],"code_challenge_methods_supported":["S256","plain"],"response_modes_supported":["query","fragment","form_post"],"subject_types_supported":["public"],"token_endpoint_auth_methods_supported":["client_secret_basic","client_secret_post","private_key_jwt","tls_client_auth","self_signed_tls_client_auth"],"token_endpoint_auth_signing_alg_values_supported":["RS256","RS384","PS256"],"claims_supported":["aud","auth_time","created_at","email","email_verified","exp","family_name","given_name","iat","identities","iss","name","nickname","phone_number","picture","sub"],"request_uri_parameter_supported":false,"request_parameter_supported":true,"id_token_signing_alg_values_supported":["HS256","RS256","PS256"],"tls_client_certificate_bound_access_tokens":true,"request_object_signing_alg_values_supported":["RS256","RS384","PS256"],"backchannel_logout_supported":true,"backchannel_logout_session_supported":true,"end_session_endpoint":"https://{Auth0テナントのドメイン}/oidc/logout","backchannel_authentication_endpoint":"https://{Auth0テナントのドメイン}/bc-authorize","backchannel_token_delivery_modes_supported":["poll"],"global_token_revocation_endpoint":"https://{Auth0テナントのドメイン}/oauth/global-token-revocation/connection/{connectionName}","global_token_revocation_endpoint_auth_methods_supported":["global-token-revocation+jwt"],"dpop_signing_alg_values_supported":["ES256"]}
```
## Authentication Request
正直ドキュメントに書いてあるし、わざわざ説明する必要があるのか疑問になった。（労力の割に求められているものではない気がする。一応Notebook LMの回答は以下の通り。
```markdown
Here are the explanations for your questions based on the provided sources.

### **Question 1: Request Parameters for the /bc-authorize Endpoint**

The Backchannel Authentication Endpoint is used to initiate the authentication flow. The parameters include:

*   **scope**: **REQUIRED**. This must contain the `openid` value to indicate an OpenID Connect request. Other values like `profile`, `email`, `address`, and `phone` can also be included.
*   **client_id**: While the client must authenticate to the endpoint, the `client_id` identifies the Relying Party (RP).
*   **login_hint, login_hint_token, or id_token_hint**: The Client **MUST** provide exactly one of these to identify the end-user.
    *   **login_hint**: A hint such as an email, phone number, or username.
    *   **login_hint_token**: A token containing user information, with details specific to the deployment.
    *   **id_token_hint**: A previously issued ID Token passed back as a hint.
*   **client_notification_token**: **REQUIRED** for Ping or Push modes. It is a bearer token used by the OP to authenticate the callback to the Client.
*   **binding_message**: **OPTIONAL**. A short, human-readable message displayed on both the consumption and authentication devices to visually link the transaction for the user.
*   **user_code**: **OPTIONAL**. A secret code (like a PIN) known only to the user, used to prevent unsolicited requests.
*   **requested_expiry**: **OPTIONAL**. A positive integer allowing the Client to request a specific lifetime for the `auth_req_id`.
*   **acr_values**: **OPTIONAL**. A space-separated string requesting specific Authentication Context Class Reference values.

---

### **Question 2: Format of login_hint (iss_sub format)**

Based on the provided sources, there is **no explicit definition or mention of an "iss_sub" format** for the `login_hint` parameter. 

The sources state that `login_hint` can contain an email address, phone number, account number, subject identifier, or username. While OpenID Connect generally uses `iss` (issuer) and `sub` (subject) to identify users, this specific combined "iss_sub" string format is not described in this document. **You may want to independently verify if this format is defined in a specific profile or a different specification.**

---

### **Question 3: Fields in the Successful Response from /bc-authorize**

When a request is successful, the OP returns an HTTP 200 OK response with the following:

*   **auth_req_id**: **REQUIRED**. A unique, opaque identifier for the authentication request. The Client uses this to retrieve tokens or validate callbacks.
*   **expires_in**: **REQUIRED**. A positive integer representing the number of seconds until the `auth_req_id` expires.
*   **interval**: **OPTIONAL**. The minimum time in seconds the Client must wait between polling requests. If not provided, the default is **5 seconds**.

---

### **Question 4: Request Parameters for /oauth/token using CIBA**

To obtain tokens in Poll or Ping mode, the Client makes a POST request with these parameters:

*   **grant_type**: **REQUIRED**. The value must be `urn:openid:params:grant-type:ciba`.
*   **auth_req_id**: **REQUIRED**. The identifier received from the `/bc-authorize` endpoint.
*   **Client Authentication**: The Client **MUST** authenticate itself to the token endpoint using its registered method (e.g., private_key_jwt or Mutual TLS).

---

### **Question 5: Polling Error Responses and Countermeasures**

During polling, the following error codes may be returned by the OP:

*   **authorization_pending**:
    *   **Meaning**: The end-user has not yet completed the authentication/authorization.
    *   **Action**: The Client should **continue polling** at the appropriate interval.
*   **slow_down**:
    *   **Meaning**: The request is still pending, but the Client is polling too frequently.
    *   **Action**: The Client must **increase its polling interval** by at least 5 seconds for all subsequent requests for that `auth_req_id`.
*   **access_denied**:
    *   **Meaning**: The end-user explicitly denied the authorization request.
    *   **Action**: The Client must **stop polling** for this `auth_req_id`.
*   **expired_token**:
    *   **Meaning**: The `auth_req_id` has reached its expiration time (`expires_in`).
    *   **Action**: The Client must **stop polling** and, if it still wishes to authenticate the user, initiate a **new Authentication Request**.
```

