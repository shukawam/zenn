---
title: "KongのIdP機能(Kong Identity)を試してみる"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: true
---

## はじめに

こんにちは 🖐️ ちょうど先月くらいに Kong Identity という機能が Beta 版で公開されました。端的に言うと、Kong が提供する IdP(Identity Provider)で、OAuth 2.0, OpenID Connect 1.0 を実装しています。ただし、ドキュメントを見たり試している感じエンドユーザーは対象ではなく、あくまで M2M の認証・認可が対象みたいです。この記事では、API クライアントと Kong Gateway で管理している API 間の認証・認可を Kong の OpenID Connect プラグインと Kong Identity を利用して実現してみます。

https://developer.konghq.com/kong-identity/

## Kong Identity とは

:::message
冒頭で記載した通り、Kong Identity は Beta 版の機能となります。API は、仕様変更される可能性もあるため、試す際はよくご確認いただければと思います。
:::

Kong が提供する IdP であり、OAuth 2.0, OpenID Connect 1.0 の実装です。作成済みの認可サーバーから公開されている Discovery Endpoint を叩いてみると以下のような情報が取得できます。

```json
{
  "issuer": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth",
  "authorization_endpoint": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth/authorize",
  "token_endpoint": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth/oauth/token",
  "introspection_endpoint": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth/introspect",
  "userinfo_endpoint": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth/userinfo",
  "revocation_endpoint": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth/revoke",
  "end_session_endpoint": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth/end_session",
  "device_authorization_endpoint": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth/device_authorization",
  "jwks_uri": "https://0tbsbd90t5xhamkz.us.identity.konghq.com/auth/.well-known/jwks",
  "scopes_supported": ["openid"],
  "response_types_supported": ["code", "id_token", "id_token token"],
  "grant_types_supported": [
    "authorization_code",
    "implicit",
    "client_credentials",
    "urn:ietf:params:oauth:grant-type:jwt-bearer"
  ],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "token_endpoint_auth_methods_supported": [
    "none",
    "client_secret_basic",
    "client_secret_post"
  ],
  "revocation_endpoint_auth_methods_supported": [
    "none",
    "client_secret_basic",
    "client_secret_post"
  ],
  "revocation_endpoint_auth_signing_alg_values_supported": ["RS256"],
  "introspection_endpoint_auth_methods_supported": ["client_secret_basic"],
  "introspection_endpoint_auth_signing_alg_values_supported": ["RS256"],
  "ui_locales_supported": ["en"],
  "request_uri_parameter_supported": false
}
```

これを見ると、`$.grant_types_supported` には ROPC(Resource Owner Password Credentials)が存在しないことがわかります。明確にドキュメンテーションされている訳ではないので不正確な可能性がありますが、ROPC は [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics-29) でも言及されている[^1]通り、_MUST NOT be used_ なので、Kong Identity として実装されてなくてもそんなに困らないはずです。

[^1]: The resource owner password credentials grant [RFC6749](https://www.rfc-editor.org/info/rfc6749) MUST NOT be used.

## 試してみる

以下のようなフローを作ってみます。

```mermaid
sequenceDiagram
  Client ->> Kong Identity: Token Request
  Kong Identity -->> Client: Generate Token(Access Token)
  Client ->> Kong Gateway: API Request w/ Access Token
  Kong Gateway -->> Kong Gateway: Validate Access Token
  Kong Gateway -->> Client: API Response
```

また、すでに Konnect のアカウントを有しており、任意の環境で Data Plane が起動されていることを前提とします。（本記事では、手元の Mac に構築済み）

### PAT(Personal Access Token) を作成する

Konnect の概要画面右上のユーザーアイコンから Personal Access Token を選択します。

![PAT](/images/try-kong-identity/pat.png)

次に、"+ Generate Token"をクリックし、PAT を生成します。

![generate pat](/images/try-kong-identity/generate-pat.png)

以降の手順のために、生成された PAT を以下のように環境変数に設定しておきます。

```sh
export KONNECT_TOKEN=kpat_...
```

### Kong Identity 周り（Auth Server, Client）を作る

まずは、Auth Server を作成します。

```sh
curl -X POST "https://us.api.konghq.com/v1/auth-servers" \
    -H "Authorization: Bearer $KONNECT_TOKEN" \
    -H "Content-Type: application/json" \
    --json '{
        "name": "kong-auth-server",
        "description": "Kong Auth Server",
        "audience": "https://example.com",
        "signing_algorithm": "RS256",
        "labels": {
            "env": "local"
        }
    }'
```

実行結果：

```json
{
  "audience": "https://example.com",
  "created_at": "2025-09-26T05:30:50.551157Z",
  "description": "Kong Auth Server",
  "id": "10be4ade-d810-4cc8-8bf6-aa75691b8473",
  "issuer": "https://hw8bfcjlw0woj0bj.us.identity.konghq.com/auth",
  "labels": { "env": "local" },
  "metadata_uri": "https://hw8bfcjlw0woj0bj.us.identity.konghq.com/auth/.well-known/openid-configuration",
  "name": "kong-auth-server",
  "signing_algorithm": "RS256",
  "updated_at": "2025-09-26T05:30:50.551157Z"
}
```

後の手順のために、実行結果に含まれる Auth Server の ID と Metadata URI を環境変数に設定します。

```sh
export AUTH_SERVER_ID="10be4ade-d810-4cc8-8bf6-aa75691b8473"
# decKと呼ばれるツールから読み込む必要があるため、DECK_というプレフィックスをつけています。
export DECK_METADATA_URI="https://hw8bfcjlw0woj0bj.us.identity.konghq.com/auth/.well-known/openid-configuration"
```

Client を作成します。

```sh
curl -X POST https://us.api.konghq.com/v1/auth-servers/$AUTH_SERVER_ID/clients \
    -H "Authorization: Bearer $KONNECT_TOKEN" \
    -H "Content-Type: application/json" \
    --json '{
        "name": "kong-openid-connect-plugin",
        "grant_types": [
            "client_credentials"
        ],
        "response_types": [
            "token",
            "code",
            "id_token"
        ],
        "access_token_duration": 300,
        "id_token_duration": 300,
        "labels": {
            "env": "local"
        }
    }'
```

実行結果（Client Secret など機微情報も含まれていますが、すでに利用できない値です）：

```json
{
  "access_token_duration": 300,
  "allow_all_scopes": false,
  "allow_scopes": [],
  "client_secret": "1togb2brqsj3kkjsw0ssnotf",
  "created_at": "2025-09-26T05:47:32.272228Z",
  "grant_types": ["client_credentials"],
  "id": "12yvvmecnlghhnfg",
  "id_token_duration": 300,
  "labels": { "env": "local" },
  "login_uri": null,
  "name": "kong-openid-connect-plugin",
  "redirect_uris": [],
  "response_types": ["token", "code", "id_token"],
  "token_endpoint_auth_method": "client_secret_post",
  "updated_at": "2025-09-26T05:47:32.272228Z"
}
```

後の手順のために、Client ID, Client Secret を環境変数として設定します。

```sh
# decK と呼ばれるツールから読み込む必要があるため、DECK_ というプレフィックスをつけています。
export DECK_CLIENT_ID="12yvvmecnlghhnfg"
export DECK_CLIENT_SECRET="1togb2brqsj3kkjsw0ssnotf"
```

### (おまけ) カスタムの Scope を追加する

認可制御に利用するために、カスタムの Scope を追加します。ここでは、環境情報を示す `local` というスコープを追加してみます。

```sh
curl -X POST https://us.api.konghq.com/v1/auth-servers/$AUTH_SERVER_ID/scopes \
    -H "Authorization: Bearer $KONNECT_TOKEN" \
    -H "Content-Type: application/json" \
    --json '{
        "name": "local",
        "description": "Scope for environment info.",
        "default": false,
        "include_in_metadata": true,
        "enabled": true
    }'
```

実行結果：

```json
{
  "created_at": "2025-09-26T05:59:51.025472Z",
  "default": false,
  "description": "Scope for local environment info.",
  "enabled": true,
  "id": "9157ad42-c612-443e-8b1b-ece77b2735f8",
  "include_in_metadata": true,
  "name": "local",
  "updated_at": "2025-09-26T05:59:51.025472Z"
}
```

後続の手順のために、Scope ID を環境変数に設定します。

```sh
export SCOPE_ID=9157ad42-c612-443e-8b1b-ece77b2735f8
```

メタデータに含めるように設定してあるので、Discovery Endpoint を叩いて、Scope に `local` が含まれていることを確認します。

```sh
curl https://hw8bfcjlw0woj0bj.us.identity.konghq.com/auth/.well-known/openid-configuration
```

実行結果：

```json
{
  "...省略...": "...省略...",
  "scopes_supported": ["openid", "local"],
  "...省略...": "...省略..."
}
```

作成した Scope を許可するように Client を更新します。

```sh
curl -X PATCH https://us.api.konghq.com/v1/auth-servers/$AUTH_SERVER_ID/clients/$DECK_CLIENT_ID \
    -H "Authorization: Bearer $KONNECT_TOKEN" \
    -H "Content-Type: application/json" \
    --json '{
        "allow_scopes": [
            "'$SCOPE_ID'"
        ]
    }'
```

実行結果：

```json
{
  "access_token_duration": 300,
  "allow_all_scopes": false,
  "allow_scopes": ["dfd94ace-1796-41d6-926f-298e64c26599"],
  "created_at": "2025-09-26T05:47:32.272228Z",
  "grant_types": ["client_credentials"],
  "id": "12yvvmecnlghhnfg",
  "id_token_duration": 300,
  "labels": { "env": "local" },
  "login_uri": null,
  "name": "kong-openid-connect-plugin",
  "redirect_uris": [],
  "response_types": ["token", "code", "id_token"],
  "token_endpoint_auth_method": "client_secret_post",
  "updated_at": "2025-09-26T07:04:10.920028Z"
}
```

### Kong の OIDC プラグインと組み合わせて使ってみる

まずは、プロキシする API とそれを保護するためのプラグインを設定します。decK を利用して、宣言的に設定しますが Konnect の GUI から設定いただいても大丈夫です。

```yaml:kong.yaml
_format_version: "3.0"
_konnect:
  control_plane_name: local-gateway

services:
  - name: httpbin-service
    url: https://httpbin.org
    routes:
      - name: httpbin-route
        paths:
          - /mock
        strip_path: true

plugins:
  - name: openid-connect
    service: httpbin-service
    config:
      issuer: ${{ env "DECK_METADATA_URI" }}
      client_id:
        - ${{ env "DECK_CLIENT_ID" }}
      client_secret:
        - ${{ env "DECK_CLIENT_SECRET" }}
      auth_methods:
        - client_credentials
        - bearer
      audience:
        - https://example.com
      authenticated_groups_claim:
        - scope
  - name: acl
    service: httpbin-service
    config:
      allow:
        - local
```

OpenID Connect プラグインだけではなく、Access Control List(ACL)のプラグインも併用していることがポイントです。ACL プラグインは認可制御に用いられ、今回の場合 scope という Claim に `local` が含まれていることを期待します。つまり、OIDC プラグインによる検証が成功しても、ACL プラグインの検証に失敗するとクライアントには、403 エラーが返却されるはずです。

上記の宣言を用いて、設定します。

```sh
deck gateway sync kong.yaml --konnect-token $KONNECT_TOKEN
```

実行結果：

```sh
creating service httpbin-service
creating route httpbin-route
creating plugin acl for service httpbin-service
creating plugin openid-connect for service httpbin-service
Summary:
  Created: 4
  Updated: 0
  Deleted: 0
```

これで、Kong の Service, Route, Plugin の設定が完了しました。試しに API を実行してみると、未認証のため 401 が返却されることが確認できます。

```sh
curl -v  http://localhost:8000/mock/anything
```

実行結果：

```sh
* Host localhost:8000 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:8000...
* Connected to localhost (::1) port 8000
> GET /mock/anything HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/8.7.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 401 Unauthorized
< Date: Fri, 26 Sep 2025 06:22:15 GMT
< Content-Type: application/json; charset=utf-8
< Connection: keep-alive
< WWW-Authenticate: Bearer realm="hw8bfcjlw0woj0bj.us.identity.konghq.com", error="invalid_token"
< Content-Length: 26
< X-Kong-Response-Latency: 2
< Server: kong/3.11.0.0-enterprise-edition
< X-Kong-Request-Id: 9e8803229c6bd821c5c80e4a6625084a
<
* Connection #0 to host localhost left intact
{"message":"Unauthorized"}
```

次に、Kong Identity から Access Token を取得します。

```sh
curl -X POST "https://hw8bfcjlw0woj0bj.us.identity.konghq.com/auth/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$DECK_CLIENT_ID" \
  -d "client_secret=$DECK_CLIENT_SECRET"
```

実行結果：

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjljMmU0NTVjLTEwYWEtNGNjZS04ZDVhLWRmZDVkMjliOWMwOSIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiaHR0cHM6Ly9leGFtcGxlLmNvbSJdLCJjbGllbnRfaWQiOiIxMnl2dm1lY25sZ2hobmZnIiwiZXhwIjoxNzU4ODY4MjcxLCJpYXQiOjE3NTg4Njc5NzEsImlzcyI6Imh0dHBzOi8vaHc4YmZjamx3MHdvajBiai51cy5pZGVudGl0eS5rb25naHEuY29tL2F1dGgiLCJqdGkiOiI4Mjc5MmI3OS05Yjk4LTQ3MWQtYmI5ZS05Yzg0MWFmZmZkZDciLCJuYmYiOjE3NTg4Njc5NzEsInNjb3BlIjoiIiwic3ViIjoiMTJ5dnZtZWNubGdoaG5mZyJ9.Bz0AzZ4Fl066dB8TxcisfBYRaB5ISMOIYYSCtk9a9m98MMbHdBCWnSzzKz019QrOuc04Krj2XjWS-6BBH8b7zWXx0j5BAnohJ67LrM_th_6llemdGJwaN6BdICx28hHg4Ars-_Qoy8aGACRPhVYmyGaVMWjilsq8eHHDZkUfnXRAX7sCAZzYsPTp0dWcf2y_-aHv19PWwoKLGZxvs8phujT4TVlWFcIk05D9rU06hJoHmCwv-QrOQ7HdGgyD-d7u2253xSLhhc4gIAhbQTNWlIoxl0LvzvccM2QA3uqMe-cZeWHNRShZFJwIBf9qkIA7OlAkVQzlouww4G1lXfJ41Q",
  "token_type": "Bearer",
  "expires_in": 299
}
```

次の手順のために、Access Token を環境変数に設定します。

```sh
export AT=eyJhb...
```

また、この Access Token をデコードし、中身の確認をしてみましょう。（実行には、`jwt` コマンドが必要です。インストールが面倒な場合は、[https://jwt.io](https://jwt.io)などで確認するのも良いでしょう。）

```sh
jwt decode $AT
```

実行結果：

```sh
Token header
------------
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "9c2e455c-10aa-4cce-8d5a-dfd5d29b9c09"
}

Token claims
------------
{
  "aud": [
    "https://example.com"
  ],
  "client_id": "12yvvmecnlghhnfg",
  "exp": 1758868271,
  "iat": 1758867971,
  "iss": "https://hw8bfcjlw0woj0bj.us.identity.konghq.com/auth",
  "jti": "82792b79-9b98-471d-bb9e-9c841afffdd7",
  "nbf": 1758867971,
  "scope": "",
  "sub": "12yvvmecnlghhnfg"
}
```

再度、プロキシしている API を Access Token を含めて叩いてみます。

```sh
curl http://localhost:8000/mock/anything \
    -v \
    -H "Authorization: Bearer $AT"
```

実行結果：

```json
* Host localhost:8000 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:8000...
* Connected to localhost (::1) port 8000
> GET /mock/anything HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/8.7.1
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjljMmU0NTVjLTEwYWEtNGNjZS04ZDVhLWRmZDVkMjliOWMwOSIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiaHR0cHM6Ly9leGFtcGxlLmNvbSJdLCJjbGllbnRfaWQiOiIxMnl2dm1lY25sZ2hobmZnIiwiZXhwIjoxNzU4ODY5MzE3LCJpYXQiOjE3NTg4NjkwMTcsImlzcyI6Imh0dHBzOi8vaHc4YmZjamx3MHdvajBiai51cy5pZGVudGl0eS5rb25naHEuY29tL2F1dGgiLCJqdGkiOiI0Zjc5OTRhYy1iMDdkLTQ3OWQtOTJjYi04Y2YxMGM4NWMxNzEiLCJuYmYiOjE3NTg4NjkwMTcsInNjb3BlIjoiIiwic3ViIjoiMTJ5dnZtZWNubGdoaG5mZyJ9.n8mSfZNG9kpSllyndy4R9jmACajNKE1m9mI8aHsPlECVNTL4aElifoH_nB9Ob0LhYom7eCkjnMLqIJPhh40Jmi0TgVJicINkP5gcvakFI1ft3pwAatJWsx0tbcGv5BSM8MDTkTLHdVsIgURQQPlMJKv-QNTWXXba7ppF7bWao-MgBoTStuJ7fJ2iraa9aofbFyBrzTkIXc9yutIn7UrvHJ8ivp7vpUFUJ8wNVnuBHDiIsIY1Cb2ShSiK_HuV7L2g6TGOQwakn_yYc4CRUVpRo-PSMcYy4xxKKBKWfOFeVB5vERT4fV-hufmhcIyrb3JFpRPkPmhXTVu6phuyCEaGUQ
>
* Request completely sent off
< HTTP/1.1 403 Forbidden
< Date: Fri, 26 Sep 2025 06:43:57 GMT
< Content-Type: application/json; charset=utf-8
< Connection: keep-alive
< Content-Length: 100
< X-Kong-Response-Latency: 5
< Server: kong/3.11.0.0-enterprise-edition
< X-Kong-Request-Id: 787c6441c47dd42c7a39b06eb433051f
<
{
  "message":"You cannot consume this service",
  "request_id":"787c6441c47dd42c7a39b06eb433051f"
* Connection #0 to host localhost left intact
}
```

期待通り、403 エラーが返却されました。今度は、Scope として環境情報（local）を要求し Access Token を取得してみましょう。

```sh
curl -X POST "https://hw8bfcjlw0woj0bj.us.identity.konghq.com/auth/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$DECK_CLIENT_ID" \
  -d "client_secret=$DECK_CLIENT_SECRET" \
  -d "scope=local"
```

実行結果：

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjljMmU0NTVjLTEwYWEtNGNjZS04ZDVhLWRmZDVkMjliOWMwOSIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiaHR0cHM6Ly9leGFtcGxlLmNvbSJdLCJjbGllbnRfaWQiOiIxMnl2dm1lY25sZ2hobmZnIiwiZXhwIjoxNzU4ODcwNTk0LCJpYXQiOjE3NTg4NzAyOTQsImlzcyI6Imh0dHBzOi8vaHc4YmZjamx3MHdvajBiai51cy5pZGVudGl0eS5rb25naHEuY29tL2F1dGgiLCJqdGkiOiIwOTRhOTZkOC02YTI0LTRlZjMtOWIwNC0wYmNiYjVlZTllN2MiLCJuYmYiOjE3NTg4NzAyOTQsInNjb3BlIjoibG9jYWwiLCJzdWIiOiIxMnl2dm1lY25sZ2hobmZnIn0.Uw-QqmyDqNJO2w-PD4_GmuSv75AY3tJaWnAMJ5RyAUl6MszFqC6d-uxT0lZsh1HMNu8tdMsSHpCCJPafP0QqBiRIBrDHO51XbThIpeWmJEVd6V6yOr2sFrlPxalYxDVQZVvZQbk8vWZdv7HvzGZUzP0mlAsnQYTpZIjYqOSwXYQYAXWrS11L1NSEgMW1yy03GIoXFZgL0qUIk3m41EqcFe8JLLbu82m_lrpiWHfPoFGwGZI93f8ZNOrQCBDzpTxJcV7Ff99WKsytARff7KlbomhTF2I2WQn0D78X_yPuHHrzgAqyzmPfB62xV5bdg2rscJC2HW9sy6IxbxEv48vwPA",
  "token_type": "Bearer",
  "expires_in": 299,
  "scope": "local"
}
```

環境変数に Access Token を設定します。

```sh
export AT=eyJh...
```

スコープに `local` が含まれていることを確認するためにデコードします。

```sh
jwt decode $AT
```

実行結果：

```sh
Token header
------------
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "9c2e455c-10aa-4cce-8d5a-dfd5d29b9c09"
}

Token claims
------------
{
  "aud": [
    "https://example.com"
  ],
  "client_id": "12yvvmecnlghhnfg",
  "exp": 1758870594,
  "iat": 1758870294,
  "iss": "https://hw8bfcjlw0woj0bj.us.identity.konghq.com/auth",
  "jti": "094a96d8-6a24-4ef3-9b04-0bcbb5ee9e7c",
  "nbf": 1758870294,
  "scope": "local",
  "sub": "12yvvmecnlghhnfg"
}
```

この Access Token を用いて再度リクエストを送ってみます。

```sh
curl http://localhost:8000/mock/anything \
    -H "Authorization: Bearer $AT"
```

実行結果：

```json
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Authorization": "Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjljMmU0NTVjLTEwYWEtNGNjZS04ZDVhLWRmZDVkMjliOWMwOSIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiaHR0cHM6Ly9leGFtcGxlLmNvbSJdLCJjbGllbnRfaWQiOiIxMnl2dm1lY25sZ2hobmZnIiwiZXhwIjoxNzU4ODcwNTk0LCJpYXQiOjE3NTg4NzAyOTQsImlzcyI6Imh0dHBzOi8vaHc4YmZjamx3MHdvajBiai51cy5pZGVudGl0eS5rb25naHEuY29tL2F1dGgiLCJqdGkiOiIwOTRhOTZkOC02YTI0LTRlZjMtOWIwNC0wYmNiYjVlZTllN2MiLCJuYmYiOjE3NTg4NzAyOTQsInNjb3BlIjoibG9jYWwiLCJzdWIiOiIxMnl2dm1lY25sZ2hobmZnIn0.Uw-QqmyDqNJO2w-PD4_GmuSv75AY3tJaWnAMJ5RyAUl6MszFqC6d-uxT0lZsh1HMNu8tdMsSHpCCJPafP0QqBiRIBrDHO51XbThIpeWmJEVd6V6yOr2sFrlPxalYxDVQZVvZQbk8vWZdv7HvzGZUzP0mlAsnQYTpZIjYqOSwXYQYAXWrS11L1NSEgMW1yy03GIoXFZgL0qUIk3m41EqcFe8JLLbu82m_lrpiWHfPoFGwGZI93f8ZNOrQCBDzpTxJcV7Ff99WKsytARff7KlbomhTF2I2WQn0D78X_yPuHHrzgAqyzmPfB62xV5bdg2rscJC2HW9sy6IxbxEv48vwPA",
    "Host": "httpbin.org",
    "User-Agent": "curl/8.7.1",
    "X-Amzn-Trace-Id": "Root=1-68d63bbd-6f422fc37ecd79bf4f9b5c9a",
    "X-Authenticated-Groups": "local",
    "X-Forwarded-Host": "localhost",
    "X-Forwarded-Path": "/mock/anything",
    "X-Forwarded-Prefix": "/mock",
    "X-Kong-Request-Id": "0cdf3a644cd53a749b216c2b2ee1de3e"
  },
  "json": null,
  "method": "GET",
  "origin": "192.168.65.1, 180.131.211.150",
  "url": "https://localhost/anything"
}
```

OIDC プラグインによる検証と ACL プラグインによる検証が成功し、クライアントに結果が返却されていることが確認できました。

### （おまけ）Terraform を用いた構成例

ここまでは、curl を用いて命令的に Kong Identity の設定を行いました。簡単にできる反面、再利用性にはかける欠点があります。Kong Identity は、Terraform Provider も公開されているので、Terraform を利用した宣言的な構成も行います。

まずは、変数用に `variables.tf` を作成します。空っぽにしておき、環境変数からでもいいですし `*.tfvars` からの参照でも大丈夫です。

```tf:variables.tf
variable "konnect_token" {

}
```

次に、 `provider.tf` を作成します。途中まで気が付かなかったのですが、`konnect-beta` の `server_url` はデフォルトが `https://global.api.konghq.com` になっています。Kong Identity はリージョン固有なリソースのため、自分の利用リージョン（例えば、`us` など）を設定する必要があります。

```tf:provider.tf
terraform {
  required_providers {
    konnect-beta = {
      source  = "Kong/konnect-beta"
      version = "0.8.0"
    }
  }
}

provider "konnect-beta" {
  personal_access_token = var.konnect_token
  server_url            = "https://us.api.konghq.com"
}
```

次に、`auth-server.tf` を作成します。このファイルに Auth Server, Client, Scope の宣言をします。

```tf:auth-server.tf
resource "konnect_auth_server" "konnect_auth_server" {
  provider    = konnect-beta
  audience    = "https://example.com"
  name        = "kong-auth-server-tf"
  description = "Kong Auth Server(terraform)"
  labels = {
    "createdBy" : "terraform"
  }
}

resource "konnect_auth_server_clients" "konnect_auth_server_clients" {
  provider       = konnect-beta
  name           = "kong-auth-server-client"
  auth_server_id = konnect_auth_server.konnect_auth_server.id
  grant_types    = ["client_credentials"]
  response_types = ["token", "code", "id_token"]
  allow_scopes = [
    konnect_auth_server_scopes.konnect_auth_server_scopes.id
  ]
  access_token_duration = 300
  id_token_duration     = 300
  labels = {
    "createdBy" : "terraform"
  }
}

resource "konnect_auth_server_scopes" "konnect_auth_server_scopes" {
  provider            = konnect-beta
  name                = "local"
  auth_server_id      = konnect_auth_server.konnect_auth_server.id
  description         = "Scope for environment info."
  default             = false
  enabled             = true
  include_in_metadata = true
}
```

あとは、`terraform apply` で関連リソースが作成されます。

## おわりに

今回は、Kong Identity という機能を試してみました。今のところ、API 操作しかサポートされていませんが仕様書やチュートリアルドキュメントがしっかりしており特に詰まることなく一連のシナリオ検証ができました。

## 参考情報

https://developer.konghq.com/how-to/configure-kong-identity-oidc/

https://developer.konghq.com/api/konnect/kong-identity/v1/
