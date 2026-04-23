---
title: "Kong Gateway 3.14.0で追加されたToken Exchangeを試してみる"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: true
---

## はじめに

こんにちは。今回は、Kong Gateway 3.14.0で追加された[RFC 8693 - OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)を試してみたいと思います。

## RFC 8693 - OAuth 2.0 Token Exchange

### なにものか

Token Exchangeは、OAuth 2.0の拡張仕様の一つであり、認可サーバーから発行してもらったアクセストークンを別の対象サービス向けに交換するためのフローです。

### どこで使われるのか

当たり前ですが、別のサービス向けにアクセストークンを交換する必要がある箇所で使われます。例えば、フロントのAPIがユーザー主体で発行されたアクセストークンを受け取り、バックエンドのAPIを呼び出す際に、そのバックエンドAPI向けのアクセストークンを再発行するケースなどです。簡易的に図にすると以下のようなイメージになります。

![token-exchange-overview](/images/token-exchange-with-kong-gateway/token-exchange-overview.png)
_Token Exchangeの簡易的なイメージ_

モチベーションは図に記載している通りさまざまですが、主に

- APIの呼び出し元を明示的にしたい（図で言うと、API-1がAPI-2を呼び出していることをトークンとして表現したい）
- 下流のサービスごとに必要なスコープを絞りたい
- 別のトークン形式に変換したい（Salesforce, SAPなどが要求するアクセストークンへの変換、など）

のようなケースが考えられます。またAI時代においては、複数のエージェントが協調してタスクをこなす際に、緩く権限を渡すのではなく、そのエージェントが実行可能なタスクをアクセストークンとして表現するようなケースも考えられます。

### どうやるのか

新しいアクセストークンを発行するためには、トークンエンドポイントに対して以下のようなパラメータを含めてリクエストを送る必要があります。

| パラメータ             | 必須/任意 | 説明                                                                                                             |
| ---------------------- | --------- | ---------------------------------------------------------------------------------------------------------------- |
| `grant_type`           | 必須      | `urn:ietf:params:oauth:grant-type:token-exchange`を指定する                                                      |
| `resource`             | 任意      | クライアントが要求するトークンを使用予定の対象サービスを指定する（通常は、URL形式）                              |
| `audience`             | 任意      | クライアントが要求するトークンを使用予定の対象サービスを指定する。`resource`とは異なり、論理的な名称を提供する。 |
| `scope`                | 任意      | クライアントがトークンの要求範囲を指定する                                                                       |
| `requested_token_type` | 任意      | 要求されたトークンの種類を示す識別子                                                                             |
| `subject_token`        | 必須      | 交換元のトークンを指定する                                                                                       |
| `subject_token_type`   | 必須      | 交換元のトークンのタイプを指定する。例えば、`urn:ietf:params:oauth:token-type:access_token`など                  |
| `actor_token`          | 任意      | 実行主体の身元を表すトークンを指定する                                                                           |
| `actor_token_type`     | 任意      | 実行主体の身元を表すトークンのタイプを指定する。例えば、`urn:ietf:params:oauth:token-type:access_token`など      |

また、アクセストークンを扱う場合は、クライアントがリソースサーバーを兼ねるケースとなるため、交換元のアクセストークンの検証も実施する必要があります。

## Kong GatewayにおけるToken Exchangeの設定

上記のように（アクセストークンを扱う）Token Exchangeを実装するためには、リソースサーバーとしてアクセストークンの検証が必要だったり、新しいトークンを要求するためのリクエストを送信する実装を行う必要があります。Kong Gatewayの3.14.0でOpenID Connectプラグインに追加された機能によって、これらの実装をKong Gatewayのプラグイン設定で完結させることができるようになりました。イメージ図は以下の通りです。

![kong-token-exchange-overview](/images/token-exchange-with-kong-gateway/token-exchange-with-kong.png)

今回の設定例では、さらに単純化した形で以下のようにします。

![todays-scope](/images/token-exchange-with-kong-gateway/todays-scope.png)

### 前提

- Kong Konnectの環境を有しており、利用可能なControl Planeが存在していること
- decKをインストールしていること
- Docker, Docker Composeがインストールされていること

それではやっていきます。

### 1. 前提となる環境の用意

まずは、Kong Gateway, KeycloakをDocker Composeで起動します。

:::message
起動には、`.env`ファイルにControl PlaneのIDを`CONTROL_PLANE_ID`という名前で記載しておく必要があります。

```.env
CONTROL_PLANE_ID=<your-control-plane-id>
```

また、KonnectのControl PlaneとData PlaneでmTLS接続をするためのクライアント証明書を`./config/kong/certs/cluster.crt`、`./config/kong/certs/cluster.key`にそれぞれ配置しておく必要があります。
これらは、KonnectでData Planeを作成した際に参照可能なものとなります。

![konnect](/images/token-exchange-with-kong-gateway/konnect.png)
※スクリーンショットで載せている秘密鍵や証明書、Control Plane IDはすでにローテーション済みのため、このまま利用はできません。
:::

```yaml:compose.yaml
x-default: &default
  networks:
    - sandbox-network

networks:
  sandbox-network:

services:
  # ============================================================
  # Kong Gateway
  # ============================================================
  kong:
    <<: *default
    container_name: kong
    image: kong/kong-gateway:3.14.0.1
    environment:
      KONG_ROLE: data_plane
      KONG_DATABASE: off
      KONG_VITALS: off
      KONG_CLUSTER_MTLS: pki
      KONG_CLUSTER_CONTROL_PLANE: ${CONTROL_PLANE_ID:-}.us.cp.konghq.com:443
      KONG_CLUSTER_SERVER_NAME: ${CONTROL_PLANE_ID:-}.us.cp.konghq.com
      KONG_CLUSTER_TELEMETRY_ENDPOINT: ${CONTROL_PLANE_ID:-}.us.tp.konghq.com:443
      KONG_CLUSTER_TELEMETRY_SERVER_NAME: ${CONTROL_PLANE_ID:-}.us.tp.konghq.com
      KONG_CLUSTER_CERT: /etc/kong/cluster-certs/cluster.crt
      KONG_CLUSTER_CERT_KEY: /etc/kong/cluster-certs/cluster.key
      KONG_LUA_SSL_TRUSTED_CERTIFICATE: system
      KONG_KONNECT_MODE: on
      KONG_CLUSTER_DP_LABELS: type:docker-macOsArmOS
      KONG_ROUTER_FLAVOR: expressions
      KONG_STATUS_LISTEN: 0.0.0.0:8100
      KONG_TRACING_INSTRUMENTATIONS: all
      KONG_TRACING_SAMPLING_RATE: 1.0
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
    ports:
      - 8000:8000
      - 8100:8100
    volumes:
      - ./config/kong/certs:/etc/kong/cluster-certs
    healthcheck:
      test: ["CMD", "kong", "health"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================================
  # Identity Provider
  # ============================================================
  keycloak:
    <<: *default
    container_name: keycloak
    image: quay.io/keycloak/keycloak:26.6.1
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin
      KC_HOSTNAME: http://localhost:8080
    ports:
      - 127.0.0.1:8080:8080
    volumes:
      - ./config/keycloak/realm-export.json:/opt/keycloak/data/import/realm-export.json
    command: ["start-dev", "--import-realm", "--features", "token-exchange"]

  # ============================================================
  # Observability（任意）
  # ============================================================
  otel-lgtm:
    <<: *default
    container_name: otel-lgtm
    image: grafana/otel-lgtm:0.25.0
    ports:
      - 3000:3000
      - 4317:4317
      - 4318:4318
```

`compose.yaml` に記載している通り、オブザーバビリティ基盤(`docker-otel-lgtm`)については任意ですが、あるとデバッグなどで便利なので一緒に起動させておくことをおすすめします。

あとは、以下のコマンドで起動させます。

```sh
docker compose up -d
```

以下のようにサービスが起動されていれば、準備完了です。

```sh
docker compose ps
```

実行結果

```sh
NAME        IMAGE                              COMMAND                   SERVICE     CREATED          STATUS                    PORTS
keycloak    quay.io/keycloak/keycloak:26.6.1   "/opt/keycloak/bin/k…"   keycloak    15 minutes ago   Up 15 minutes             127.0.0.1:8080->8080/tcp
kong        kong/kong-gateway:3.14.0.1         "/entrypoint.sh kong…"   kong        15 minutes ago   Up 15 minutes (healthy)   0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp, 0.0.0.0:8100->8100/tcp, [::]:8100->8100/tcp
otel-lgtm   grafana/otel-lgtm:0.25.0           "/otel-lgtm/run-all.…"   otel-lgtm   15 minutes ago   Up 15 minutes (healthy)   0.0.0.0:4317-4318->4317-4318/tcp, [::]:4317-4318->4317-4318/tcp, 0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
```

### 2. Keycloakの設定

Kong GatewayでToken Exchangeを行うため、Keycloak側でRealmを作成し、設定を行います。Realmの作成時は、事前に用意済みである `realm-export.json` を用います。

![import-realm](/images/token-exchange-with-kong-gateway/import-realm.png)

:::details realm-export.json の内容

```json:realm-export.json
{
  "id": "3bf88b95-73e4-4aff-bf34-e21aacb3639d",
  "realm": "kong",
  "notBefore": 0,
  "defaultSignatureAlgorithm": "RS256",
  "revokeRefreshToken": false,
  "refreshTokenMaxReuse": 0,
  "accessTokenLifespan": 300,
  "accessTokenLifespanForImplicitFlow": 900,
  "ssoSessionIdleTimeout": 1800,
  "ssoSessionMaxLifespan": 36000,
  "ssoSessionIdleTimeoutRememberMe": 0,
  "ssoSessionMaxLifespanRememberMe": 0,
  "offlineSessionIdleTimeout": 2592000,
  "offlineSessionMaxLifespanEnabled": false,
  "offlineSessionMaxLifespan": 5184000,
  "clientSessionIdleTimeout": 0,
  "clientSessionMaxLifespan": 0,
  "clientOfflineSessionIdleTimeout": 0,
  "clientOfflineSessionMaxLifespan": 0,
  "accessCodeLifespan": 60,
  "accessCodeLifespanUserAction": 300,
  "accessCodeLifespanLogin": 1800,
  "actionTokenGeneratedByAdminLifespan": 43200,
  "actionTokenGeneratedByUserLifespan": 300,
  "oauth2DeviceCodeLifespan": 600,
  "oauth2DevicePollingInterval": 5,
  "enabled": true,
  "sslRequired": "external",
  "registrationAllowed": false,
  "registrationEmailAsUsername": false,
  "rememberMe": false,
  "verifyEmail": false,
  "loginWithEmailAllowed": true,
  "duplicateEmailsAllowed": false,
  "resetPasswordAllowed": false,
  "editUsernameAllowed": false,
  "bruteForceProtected": false,
  "permanentLockout": false,
  "maxTemporaryLockouts": 0,
  "bruteForceStrategy": "MULTIPLE",
  "maxFailureWaitSeconds": 900,
  "minimumQuickLoginWaitSeconds": 60,
  "waitIncrementSeconds": 60,
  "quickLoginCheckMilliSeconds": 1000,
  "maxDeltaTimeSeconds": 43200,
  "failureFactor": 30,
  "maxSecondaryAuthFailures": 0,
  "defaultRole": {
    "id": "7b8fb904-5405-4e9a-b435-ce3f2b45d486",
    "name": "default-roles-kong",
    "description": "${role_default-roles}",
    "composite": true,
    "clientRole": false,
    "containerId": "3bf88b95-73e4-4aff-bf34-e21aacb3639d"
  },
  "requiredCredentials": [
    "password"
  ],
  "otpPolicyType": "totp",
  "otpPolicyAlgorithm": "HmacSHA1",
  "otpPolicyInitialCounter": 0,
  "otpPolicyDigits": 6,
  "otpPolicyLookAheadWindow": 1,
  "otpPolicyPeriod": 30,
  "otpPolicyCodeReusable": false,
  "otpSupportedApplications": [
    "totpAppFreeOTPName",
    "totpAppGoogleName",
    "totpAppMicrosoftAuthenticatorName"
  ],
  "localizationTexts": {},
  "webAuthnPolicyRpEntityName": "keycloak",
  "webAuthnPolicySignatureAlgorithms": [
    "ES256",
    "RS256"
  ],
  "webAuthnPolicyRpId": "",
  "webAuthnPolicyAttestationConveyancePreference": "not specified",
  "webAuthnPolicyAuthenticatorAttachment": "not specified",
  "webAuthnPolicyRequireResidentKey": "not specified",
  "webAuthnPolicyUserVerificationRequirement": "not specified",
  "webAuthnPolicyCreateTimeout": 0,
  "webAuthnPolicyAvoidSameAuthenticatorRegister": false,
  "webAuthnPolicyAcceptableAaguids": [],
  "webAuthnPolicyExtraOrigins": [],
  "webAuthnPolicyPasswordlessRpEntityName": "keycloak",
  "webAuthnPolicyPasswordlessSignatureAlgorithms": [
    "ES256",
    "RS256"
  ],
  "webAuthnPolicyPasswordlessRpId": "",
  "webAuthnPolicyPasswordlessAttestationConveyancePreference": "not specified",
  "webAuthnPolicyPasswordlessAuthenticatorAttachment": "not specified",
  "webAuthnPolicyPasswordlessRequireResidentKey": "Yes",
  "webAuthnPolicyPasswordlessUserVerificationRequirement": "required",
  "webAuthnPolicyPasswordlessCreateTimeout": 0,
  "webAuthnPolicyPasswordlessAvoidSameAuthenticatorRegister": false,
  "webAuthnPolicyPasswordlessAcceptableAaguids": [],
  "webAuthnPolicyPasswordlessExtraOrigins": [],
  "users": [
    {
      "id": "b4768e19-2d00-43ac-b675-647e370c6f7c",
      "username": "service-account-source",
      "emailVerified": false,
      "enabled": true,
      "createdTimestamp": 1776917912927,
      "totp": false,
      "serviceAccountClientId": "source",
      "disableableCredentialTypes": [],
      "requiredActions": [],
      "notBefore": 0
    }
  ],
  "scopeMappings": [
    {
      "clientScope": "offline_access",
      "roles": [
        "offline_access"
      ]
    }
  ],
  "clientScopeMappings": {
    "account": [
      {
        "client": "account-console",
        "roles": [
          "manage-account",
          "view-groups"
        ]
      }
    ]
  },
  "clients": [
    {
      "id": "54fec273-ed12-493c-a5ce-a17a002fbdd8",
      "clientId": "account",
      "name": "${client_account}",
      "rootUrl": "${authBaseUrl}",
      "baseUrl": "/realms/kong/account/",
      "surrogateAuthRequired": false,
      "enabled": true,
      "alwaysDisplayInConsole": false,
      "clientAuthenticatorType": "client-secret",
      "redirectUris": [
        "/realms/kong/account/*"
      ],
      "webOrigins": [],
      "notBefore": 0,
      "bearerOnly": false,
      "consentRequired": false,
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": false,
      "publicClient": true,
      "frontchannelLogout": false,
      "protocol": "openid-connect",
      "attributes": {
        "realm_client": "false",
        "post.logout.redirect.uris": "+"
      },
      "authenticationFlowBindingOverrides": {},
      "fullScopeAllowed": false,
      "nodeReRegistrationTimeout": 0,
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
      ],
      "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "microprofile-jwt"
      ]
    },
    {
      "id": "f4bd0b5e-11fa-408b-8faa-9e6f6faeca22",
      "clientId": "account-console",
      "name": "${client_account-console}",
      "rootUrl": "${authBaseUrl}",
      "baseUrl": "/realms/kong/account/",
      "surrogateAuthRequired": false,
      "enabled": true,
      "alwaysDisplayInConsole": false,
      "clientAuthenticatorType": "client-secret",
      "redirectUris": [
        "/realms/kong/account/*"
      ],
      "webOrigins": [],
      "notBefore": 0,
      "bearerOnly": false,
      "consentRequired": false,
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": false,
      "publicClient": true,
      "frontchannelLogout": false,
      "protocol": "openid-connect",
      "attributes": {
        "realm_client": "false",
        "post.logout.redirect.uris": "+",
        "pkce.code.challenge.method": "S256"
      },
      "authenticationFlowBindingOverrides": {},
      "fullScopeAllowed": false,
      "nodeReRegistrationTimeout": 0,
      "protocolMappers": [
        {
          "id": "6c22cd03-fe46-4d09-947b-f566ac402fd8",
          "name": "audience resolve",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-audience-resolve-mapper",
          "consentRequired": false,
          "config": {}
        }
      ],
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
      ],
      "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "microprofile-jwt"
      ]
    },
    {
      "id": "376cd5c0-db30-4166-8b73-788437120ce4",
      "clientId": "admin-cli",
      "name": "${client_admin-cli}",
      "surrogateAuthRequired": false,
      "enabled": true,
      "alwaysDisplayInConsole": false,
      "clientAuthenticatorType": "client-secret",
      "redirectUris": [],
      "webOrigins": [],
      "notBefore": 0,
      "bearerOnly": false,
      "consentRequired": false,
      "standardFlowEnabled": false,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": true,
      "serviceAccountsEnabled": false,
      "publicClient": true,
      "frontchannelLogout": false,
      "protocol": "openid-connect",
      "attributes": {
        "realm_client": "false",
        "client.use.lightweight.access.token.enabled": "true",
        "post.logout.redirect.uris": "+"
      },
      "authenticationFlowBindingOverrides": {},
      "fullScopeAllowed": true,
      "nodeReRegistrationTimeout": 0,
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
      ],
      "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "microprofile-jwt"
      ]
    },
    {
      "id": "dc1d2fac-1ad2-4498-8304-fbe33a217a2f",
      "clientId": "broker",
      "name": "${client_broker}",
      "surrogateAuthRequired": false,
      "enabled": true,
      "alwaysDisplayInConsole": false,
      "clientAuthenticatorType": "client-secret",
      "redirectUris": [],
      "webOrigins": [],
      "notBefore": 0,
      "bearerOnly": true,
      "consentRequired": false,
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": false,
      "publicClient": false,
      "frontchannelLogout": false,
      "protocol": "openid-connect",
      "attributes": {
        "realm_client": "true",
        "post.logout.redirect.uris": "+"
      },
      "authenticationFlowBindingOverrides": {},
      "fullScopeAllowed": false,
      "nodeReRegistrationTimeout": 0,
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
      ],
      "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "microprofile-jwt"
      ]
    },
    {
      "id": "a25f21b1-65e5-4047-96e8-e1ce5ba7ce4f",
      "clientId": "realm-management",
      "name": "${client_realm-management}",
      "surrogateAuthRequired": false,
      "enabled": true,
      "alwaysDisplayInConsole": false,
      "clientAuthenticatorType": "client-secret",
      "redirectUris": [],
      "webOrigins": [],
      "notBefore": 0,
      "bearerOnly": true,
      "consentRequired": false,
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": false,
      "publicClient": false,
      "frontchannelLogout": false,
      "protocol": "openid-connect",
      "attributes": {
        "realm_client": "true",
        "post.logout.redirect.uris": "+"
      },
      "authenticationFlowBindingOverrides": {},
      "fullScopeAllowed": false,
      "nodeReRegistrationTimeout": 0,
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
      ],
      "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "microprofile-jwt"
      ]
    },
    {
      "id": "a953b1fd-2730-4748-95eb-7d6423e4b4ac",
      "clientId": "security-admin-console",
      "name": "${client_security-admin-console}",
      "rootUrl": "${authAdminUrl}",
      "baseUrl": "/admin/kong/console/",
      "surrogateAuthRequired": false,
      "enabled": true,
      "alwaysDisplayInConsole": false,
      "clientAuthenticatorType": "client-secret",
      "redirectUris": [
        "/admin/kong/console/*"
      ],
      "webOrigins": [
        "+"
      ],
      "notBefore": 0,
      "bearerOnly": false,
      "consentRequired": false,
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": false,
      "publicClient": true,
      "frontchannelLogout": false,
      "protocol": "openid-connect",
      "attributes": {
        "realm_client": "false",
        "client.use.lightweight.access.token.enabled": "true",
        "post.logout.redirect.uris": "+",
        "pkce.code.challenge.method": "S256"
      },
      "authenticationFlowBindingOverrides": {},
      "fullScopeAllowed": true,
      "nodeReRegistrationTimeout": 0,
      "protocolMappers": [
        {
          "id": "ba7849fb-5780-4b6d-8f40-9ac5c7ec18ab",
          "name": "locale",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "locale",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "locale",
            "jsonType.label": "String"
          }
        }
      ],
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
      ],
      "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "microprofile-jwt"
      ]
    },
    {
      "id": "2b2593d8-679a-4604-8199-84edf159a151",
      "clientId": "source",
      "name": "",
      "description": "",
      "rootUrl": "",
      "adminUrl": "",
      "baseUrl": "",
      "surrogateAuthRequired": false,
      "enabled": true,
      "alwaysDisplayInConsole": false,
      "clientAuthenticatorType": "client-secret",
      "secret": "**********",
      "redirectUris": [
        "/*"
      ],
      "webOrigins": [
        "/*"
      ],
      "notBefore": 0,
      "bearerOnly": false,
      "consentRequired": false,
      "standardFlowEnabled": false,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": true,
      "publicClient": false,
      "frontchannelLogout": true,
      "protocol": "openid-connect",
      "attributes": {
        "logout.confirmation.enabled": "false",
        "client.secret.creation.time": "1776915413",
        "standard.token.exchange.enabled": "false",
        "oauth2.jwt.authorization.grant.enabled": "false",
        "frontchannel.logout.session.required": "true",
        "post.logout.redirect.uris": "+",
        "oauth2.device.authorization.grant.enabled": "false",
        "backchannel.logout.revoke.offline.tokens": "false",
        "realm_client": "false",
        "oidc.ciba.grant.enabled": "false",
        "backchannel.logout.session.required": "true",
        "display.on.consent.screen": "false",
        "dpop.bound.access.tokens": "false"
      },
      "authenticationFlowBindingOverrides": {},
      "fullScopeAllowed": true,
      "nodeReRegistrationTimeout": -1,
      "defaultClientScopes": [
        "service_account",
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
      ],
      "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "add-target-as-audience",
        "microprofile-jwt"
      ]
    },
    {
      "id": "181ef7ed-6a04-4238-bc1c-f1c6aaeff2d6",
      "clientId": "target",
      "name": "",
      "description": "",
      "rootUrl": "",
      "adminUrl": "",
      "baseUrl": "",
      "surrogateAuthRequired": false,
      "enabled": true,
      "alwaysDisplayInConsole": false,
      "clientAuthenticatorType": "client-secret",
      "secret": "**********",
      "redirectUris": [
        "http://localhost:8000/*"
      ],
      "webOrigins": [
        ""
      ],
      "notBefore": 0,
      "bearerOnly": false,
      "consentRequired": false,
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": false,
      "publicClient": false,
      "frontchannelLogout": true,
      "protocol": "openid-connect",
      "attributes": {
        "realm_client": "false",
        "oidc.ciba.grant.enabled": "false",
        "client.secret.creation.time": "1776915439",
        "backchannel.logout.session.required": "true",
        "standard.token.exchange.enabled": "true",
        "oauth2.jwt.authorization.grant.enabled": "false",
        "post.logout.redirect.uris": "+",
        "oauth2.device.authorization.grant.enabled": "false",
        "backchannel.logout.revoke.offline.tokens": "false",
        "dpop.bound.access.tokens": "false"
      },
      "authenticationFlowBindingOverrides": {},
      "fullScopeAllowed": true,
      "nodeReRegistrationTimeout": -1,
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
      ],
      "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "microprofile-jwt"
      ]
    }
  ],
  "clientScopes": [
    {
      "id": "ab57117d-022d-464b-9c06-915c6ab430af",
      "name": "offline_access",
      "description": "OpenID Connect built-in scope: offline_access",
      "protocol": "openid-connect",
      "attributes": {
        "consent.screen.text": "${offlineAccessScopeConsentText}",
        "display.on.consent.screen": "true"
      }
    },
    {
      "id": "2fcc25d2-918f-43d7-a791-d6a260a3ff70",
      "name": "address",
      "description": "OpenID Connect built-in scope: address",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "true",
        "consent.screen.text": "${addressScopeConsentText}",
        "display.on.consent.screen": "true"
      },
      "protocolMappers": [
        {
          "id": "b6306834-854d-4bae-812b-2f325ed7c271",
          "name": "address",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-address-mapper",
          "consentRequired": false,
          "config": {
            "user.attribute.formatted": "formatted",
            "user.attribute.country": "country",
            "introspection.token.claim": "true",
            "user.attribute.postal_code": "postal_code",
            "userinfo.token.claim": "true",
            "user.attribute.street": "street",
            "id.token.claim": "true",
            "user.attribute.region": "region",
            "access.token.claim": "true",
            "user.attribute.locality": "locality"
          }
        }
      ]
    },
    {
      "id": "35122cd6-bf62-4e7c-bd0e-7d90257c3681",
      "name": "phone",
      "description": "OpenID Connect built-in scope: phone",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "true",
        "consent.screen.text": "${phoneScopeConsentText}",
        "display.on.consent.screen": "true"
      },
      "protocolMappers": [
        {
          "id": "838fd270-39cf-4a92-8270-a83fea1bd158",
          "name": "phone number",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "phoneNumber",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "phone_number",
            "jsonType.label": "String"
          }
        },
        {
          "id": "5b5e63b1-84a1-403c-b6e3-23cb2ae8c83c",
          "name": "phone number verified",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "phoneNumberVerified",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "phone_number_verified",
            "jsonType.label": "boolean"
          }
        }
      ]
    },
    {
      "id": "2981f4e6-8a2a-46b1-ba14-e76682d17dcb",
      "name": "acr",
      "description": "OpenID Connect scope for add acr (authentication context class reference) to the token",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "false",
        "display.on.consent.screen": "false"
      },
      "protocolMappers": [
        {
          "id": "ce9261c2-6f0d-4d0b-8ebf-9f4389371556",
          "name": "acr loa level",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-acr-mapper",
          "consentRequired": false,
          "config": {
            "id.token.claim": "true",
            "introspection.token.claim": "true",
            "access.token.claim": "true",
            "userinfo.token.claim": "true"
          }
        }
      ]
    },
    {
      "id": "49afc2ee-b560-4d46-8437-338a3caa5d17",
      "name": "email",
      "description": "OpenID Connect built-in scope: email",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "true",
        "consent.screen.text": "${emailScopeConsentText}",
        "display.on.consent.screen": "true"
      },
      "protocolMappers": [
        {
          "id": "5b21d5e3-a063-4ab5-9469-34948d696ce1",
          "name": "email verified",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-property-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "emailVerified",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "email_verified",
            "jsonType.label": "boolean"
          }
        },
        {
          "id": "78ffbdec-4c08-4c84-bddf-391953a8b4c6",
          "name": "email",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "email",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "email",
            "jsonType.label": "String"
          }
        }
      ]
    },
    {
      "id": "08382a28-e780-4d9d-9542-6d6a50307fbf",
      "name": "service_account",
      "description": "Specific scope for a client enabled for service accounts",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "false",
        "display.on.consent.screen": "false"
      },
      "protocolMappers": [
        {
          "id": "a5de70ee-6717-4a3a-a978-8a11d18111ec",
          "name": "Client Host",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usersessionmodel-note-mapper",
          "consentRequired": false,
          "config": {
            "user.session.note": "clientHost",
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "clientHost",
            "jsonType.label": "String"
          }
        },
        {
          "id": "f1045282-1a73-4c39-9382-ec2ee8b94c09",
          "name": "Client ID",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usersessionmodel-note-mapper",
          "consentRequired": false,
          "config": {
            "user.session.note": "client_id",
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "client_id",
            "jsonType.label": "String"
          }
        },
        {
          "id": "4fef06d4-7945-431a-bcd5-bc20fa16cf4b",
          "name": "Client IP Address",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usersessionmodel-note-mapper",
          "consentRequired": false,
          "config": {
            "user.session.note": "clientAddress",
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "clientAddress",
            "jsonType.label": "String"
          }
        }
      ]
    },
    {
      "id": "1df188bd-bd8b-4795-9627-0e79568d3bbb",
      "name": "profile",
      "description": "OpenID Connect built-in scope: profile",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "true",
        "consent.screen.text": "${profileScopeConsentText}",
        "display.on.consent.screen": "true"
      },
      "protocolMappers": [
        {
          "id": "f7dc05c2-4bc0-4b6d-9eb2-9cec418d9639",
          "name": "picture",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "picture",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "picture",
            "jsonType.label": "String"
          }
        },
        {
          "id": "07f35d5b-ddbe-4b52-bf2a-ed15e2c3ca78",
          "name": "website",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "website",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "website",
            "jsonType.label": "String"
          }
        },
        {
          "id": "4afb43d4-1207-4251-b809-119ccb36c480",
          "name": "nickname",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "nickname",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "nickname",
            "jsonType.label": "String"
          }
        },
        {
          "id": "521ec0b1-82bd-420f-ab53-3c0df110d83a",
          "name": "profile",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "profile",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "profile",
            "jsonType.label": "String"
          }
        },
        {
          "id": "4db1e161-635e-4d41-a550-f3ed76d60f3b",
          "name": "given name",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "firstName",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "given_name",
            "jsonType.label": "String"
          }
        },
        {
          "id": "5db26300-f20b-43f2-968c-51b95a79dc12",
          "name": "updated at",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "updatedAt",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "updated_at",
            "jsonType.label": "long"
          }
        },
        {
          "id": "86db5bfb-da99-4501-b2f4-c9f2def9ceaf",
          "name": "birthdate",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "birthdate",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "birthdate",
            "jsonType.label": "String"
          }
        },
        {
          "id": "ab831c83-b941-405e-a1a9-23521fe85f4a",
          "name": "family name",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "lastName",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "family_name",
            "jsonType.label": "String"
          }
        },
        {
          "id": "c75399e5-1b76-4fae-9b93-2c85fc5795f6",
          "name": "username",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "username",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "preferred_username",
            "jsonType.label": "String"
          }
        },
        {
          "id": "011f3cbe-2e7c-4ff7-bd3b-9e4bdfb2645f",
          "name": "locale",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "locale",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "locale",
            "jsonType.label": "String"
          }
        },
        {
          "id": "cdd478c1-7c16-4841-a582-14415c036569",
          "name": "full name",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-full-name-mapper",
          "consentRequired": false,
          "config": {
            "id.token.claim": "true",
            "introspection.token.claim": "true",
            "access.token.claim": "true",
            "userinfo.token.claim": "true"
          }
        },
        {
          "id": "3288d41a-2e64-4140-b6c5-652b122e4992",
          "name": "zoneinfo",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "zoneinfo",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "zoneinfo",
            "jsonType.label": "String"
          }
        },
        {
          "id": "c58f01a9-073a-4027-9cd2-e75bf9409244",
          "name": "middle name",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "middleName",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "middle_name",
            "jsonType.label": "String"
          }
        },
        {
          "id": "38e7d7ae-ffd1-449e-bc68-4ee42019c178",
          "name": "gender",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "gender",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "gender",
            "jsonType.label": "String"
          }
        }
      ]
    },
    {
      "id": "25dccbcb-c1a1-420d-a883-b3c1433b3cee",
      "name": "role_list",
      "description": "SAML role list",
      "protocol": "saml",
      "attributes": {
        "consent.screen.text": "${samlRoleListScopeConsentText}",
        "display.on.consent.screen": "true"
      },
      "protocolMappers": [
        {
          "id": "6a14a208-45ac-450c-83d6-ca9afbabe494",
          "name": "role list",
          "protocol": "saml",
          "protocolMapper": "saml-role-list-mapper",
          "consentRequired": false,
          "config": {
            "single": "false",
            "attribute.nameformat": "Basic",
            "attribute.name": "Role"
          }
        }
      ]
    },
    {
      "id": "d0916d27-3edd-48c4-a8ff-9618ba93cfa5",
      "name": "saml_organization",
      "description": "Organization Membership",
      "protocol": "saml",
      "attributes": {
        "display.on.consent.screen": "false"
      },
      "protocolMappers": [
        {
          "id": "e293a36a-a9f1-484c-a1c2-4a16bfb82a75",
          "name": "organization",
          "protocol": "saml",
          "protocolMapper": "saml-organization-membership-mapper",
          "consentRequired": false,
          "config": {}
        }
      ]
    },
    {
      "id": "1eb05827-5e8f-46ae-9e92-f5f5d1a425d3",
      "name": "web-origins",
      "description": "OpenID Connect scope for add allowed web origins to the access token",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "false",
        "consent.screen.text": "",
        "display.on.consent.screen": "false"
      },
      "protocolMappers": [
        {
          "id": "738c6d88-0cbe-4a5b-858f-2d6b8120f496",
          "name": "allowed web origins",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-allowed-origins-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "access.token.claim": "true"
          }
        }
      ]
    },
    {
      "id": "37eef9fc-4eef-4457-99a8-090566e7a8b7",
      "name": "organization",
      "description": "Additional claims about the organization a subject belongs to",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "true",
        "consent.screen.text": "${organizationScopeConsentText}",
        "display.on.consent.screen": "true"
      },
      "protocolMappers": [
        {
          "id": "c97a45fd-7994-47fb-9be4-001a08f8634d",
          "name": "organization",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-organization-membership-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "multivalued": "true",
            "userinfo.token.claim": "true",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "organization",
            "jsonType.label": "String"
          }
        }
      ]
    },
    {
      "id": "6d8f9f4d-e54d-4e3e-9302-1371cc58b235",
      "name": "basic",
      "description": "OpenID Connect scope for add all basic claims to the token",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "false",
        "display.on.consent.screen": "false"
      },
      "protocolMappers": [
        {
          "id": "a14d400b-637c-4b6c-bc08-05ee0afbd955",
          "name": "auth_time",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usersessionmodel-note-mapper",
          "consentRequired": false,
          "config": {
            "user.session.note": "AUTH_TIME",
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "auth_time",
            "jsonType.label": "long"
          }
        },
        {
          "id": "34dc8fc5-5424-4485-9dd0-d913f7dc19c7",
          "name": "sub",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-sub-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "access.token.claim": "true"
          }
        }
      ]
    },
    {
      "id": "015aa1fe-d7ad-431f-a2eb-815a3d70d814",
      "name": "microprofile-jwt",
      "description": "Microprofile - JWT built-in scope",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "true",
        "display.on.consent.screen": "false"
      },
      "protocolMappers": [
        {
          "id": "1365e1f9-e2a2-474b-86e4-2aba305431c4",
          "name": "upn",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "username",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "upn",
            "jsonType.label": "String"
          }
        },
        {
          "id": "8e826a21-2c60-4052-8938-8b9bedb62b78",
          "name": "groups",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-realm-role-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "multivalued": "true",
            "userinfo.token.claim": "true",
            "user.attribute": "foo",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "groups",
            "jsonType.label": "String"
          }
        }
      ]
    },
    {
      "id": "29b58f71-ca3b-4266-90b7-70cb7b2ae6fc",
      "name": "roles",
      "description": "OpenID Connect scope for add user roles to the access token",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "false",
        "consent.screen.text": "${rolesScopeConsentText}",
        "display.on.consent.screen": "true"
      },
      "protocolMappers": [
        {
          "id": "1cd98ae0-dabf-4dee-bf14-f5fdaa30a5ab",
          "name": "audience resolve",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-audience-resolve-mapper",
          "consentRequired": false,
          "config": {
            "introspection.token.claim": "true",
            "access.token.claim": "true"
          }
        },
        {
          "id": "cd2f35ed-492d-4251-a3cc-c58e519562c8",
          "name": "client roles",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-client-role-mapper",
          "consentRequired": false,
          "config": {
            "user.attribute": "foo",
            "introspection.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "resource_access.${client_id}.roles",
            "jsonType.label": "String",
            "multivalued": "true"
          }
        },
        {
          "id": "1544eeff-b91a-4e52-b9c1-82df926d7094",
          "name": "realm roles",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-realm-role-mapper",
          "consentRequired": false,
          "config": {
            "user.attribute": "foo",
            "introspection.token.claim": "true",
            "access.token.claim": "true",
            "claim.name": "realm_access.roles",
            "jsonType.label": "String",
            "multivalued": "true"
          }
        }
      ]
    },
    {
      "id": "467c786f-80c8-4c6f-b9b4-461e74c7efcd",
      "name": "add-target-as-audience",
      "description": "",
      "protocol": "openid-connect",
      "attributes": {
        "include.in.token.scope": "false",
        "display.on.consent.screen": "true",
        "gui.order": "",
        "consent.screen.text": "",
        "include.in.openid.provider.metadata": "true"
      },
      "protocolMappers": [
        {
          "id": "b9b8927d-c721-491a-8937-d4c0a462ac51",
          "name": "add-target",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-audience-mapper",
          "consentRequired": false,
          "config": {
            "included.client.audience": "target",
            "id.token.claim": "false",
            "lightweight.claim": "false",
            "introspection.token.claim": "true",
            "access.token.claim": "true",
            "userinfo.token.claim": "false"
          }
        }
      ]
    }
  ],
  "defaultDefaultClientScopes": [
    "role_list",
    "saml_organization",
    "profile",
    "email",
    "roles",
    "web-origins",
    "acr",
    "basic"
  ],
  "defaultOptionalClientScopes": [
    "offline_access",
    "address",
    "phone",
    "microprofile-jwt",
    "organization",
    "add-target-as-audience"
  ],
  "browserSecurityHeaders": {
    "contentSecurityPolicyReportOnly": "",
    "xContentTypeOptions": "nosniff",
    "referrerPolicy": "no-referrer",
    "xRobotsTag": "none",
    "xFrameOptions": "SAMEORIGIN",
    "contentSecurityPolicy": "frame-src 'self'; frame-ancestors 'self'; object-src 'none';",
    "strictTransportSecurity": "max-age=31536000; includeSubDomains"
  },
  "smtpServer": {},
  "eventsEnabled": false,
  "eventsListeners": [
    "jboss-logging"
  ],
  "enabledEventTypes": [],
  "adminEventsEnabled": false,
  "adminEventsDetailsEnabled": false,
  "identityProviders": [],
  "identityProviderMappers": [],
  "components": {
    "org.keycloak.services.clientregistration.policy.ClientRegistrationPolicy": [
      {
        "id": "8c14ae38-69d7-49d9-8899-35bceaa3025d",
        "name": "Allowed Protocol Mapper Types",
        "providerId": "allowed-protocol-mappers",
        "subType": "authenticated",
        "subComponents": {},
        "config": {
          "allowed-protocol-mapper-types": [
            "saml-user-attribute-mapper",
            "saml-user-property-mapper",
            "oidc-usermodel-property-mapper",
            "saml-role-list-mapper",
            "oidc-usermodel-attribute-mapper",
            "oidc-address-mapper",
            "oidc-sha256-pairwise-sub-mapper",
            "oidc-full-name-mapper"
          ]
        }
      },
      {
        "id": "1d3f28aa-32a4-449c-af56-2fb9da7e8bc1",
        "name": "Trusted Hosts",
        "providerId": "trusted-hosts",
        "subType": "anonymous",
        "subComponents": {},
        "config": {
          "host-sending-registration-request-must-match": [
            "true"
          ],
          "client-uris-must-match": [
            "true"
          ]
        }
      },
      {
        "id": "09bd12a6-4971-41a1-8032-1aba342838d2",
        "name": "Allowed Client Scopes",
        "providerId": "allowed-client-templates",
        "subType": "anonymous",
        "subComponents": {},
        "config": {
          "allow-default-scopes": [
            "true"
          ]
        }
      },
      {
        "id": "1ed65f58-3c54-4d6c-b51d-45a7c747e051",
        "name": "Allowed Registration Web Origins",
        "providerId": "registration-web-origins",
        "subType": "authenticated",
        "subComponents": {},
        "config": {}
      },
      {
        "id": "198926a8-b270-45dd-bfa2-b042163d0ebe",
        "name": "Full Scope Disabled",
        "providerId": "scope",
        "subType": "anonymous",
        "subComponents": {},
        "config": {}
      },
      {
        "id": "881b378c-2e3a-48b0-8fbb-c30419e72ad3",
        "name": "Consent Required",
        "providerId": "consent-required",
        "subType": "anonymous",
        "subComponents": {},
        "config": {}
      },
      {
        "id": "de9d3068-aac7-4203-8ed5-a7bb5f81e31d",
        "name": "Allowed Protocol Mapper Types",
        "providerId": "allowed-protocol-mappers",
        "subType": "anonymous",
        "subComponents": {},
        "config": {
          "allowed-protocol-mapper-types": [
            "oidc-full-name-mapper",
            "oidc-address-mapper",
            "saml-user-property-mapper",
            "oidc-sha256-pairwise-sub-mapper",
            "saml-user-attribute-mapper",
            "saml-role-list-mapper",
            "oidc-usermodel-attribute-mapper",
            "oidc-usermodel-property-mapper"
          ]
        }
      },
      {
        "id": "1faf9a5c-5aa2-43da-b618-ec2568d58327",
        "name": "Allowed Registration Web Origins",
        "providerId": "registration-web-origins",
        "subType": "anonymous",
        "subComponents": {},
        "config": {}
      },
      {
        "id": "55bb41fd-2d88-4345-9471-0abdaad33e44",
        "name": "Allowed Client Scopes",
        "providerId": "allowed-client-templates",
        "subType": "authenticated",
        "subComponents": {},
        "config": {
          "allow-default-scopes": [
            "true"
          ]
        }
      },
      {
        "id": "bf8cb6b4-2484-4288-91ae-c1e8491f5892",
        "name": "Max Clients Limit",
        "providerId": "max-clients",
        "subType": "anonymous",
        "subComponents": {},
        "config": {
          "max-clients": [
            "200"
          ]
        }
      }
    ],
    "org.keycloak.keys.KeyProvider": [
      {
        "id": "bc08f2f0-3b32-428b-b640-0f1431fa5871",
        "name": "hmac-generated-hs512",
        "providerId": "hmac-generated",
        "subComponents": {},
        "config": {
          "priority": [
            "100"
          ],
          "algorithm": [
            "HS512"
          ]
        }
      },
      {
        "id": "3ac38eb5-ae7e-4592-9a04-534ee49e587c",
        "name": "rsa-enc-generated",
        "providerId": "rsa-enc-generated",
        "subComponents": {},
        "config": {
          "priority": [
            "100"
          ],
          "algorithm": [
            "RSA-OAEP"
          ]
        }
      },
      {
        "id": "e547e1e3-1ac1-43e0-9c57-7d141ff1095c",
        "name": "rsa-generated",
        "providerId": "rsa-generated",
        "subComponents": {},
        "config": {
          "priority": [
            "100"
          ]
        }
      },
      {
        "id": "81057097-9679-44d7-9a03-fa9c70375305",
        "name": "aes-generated",
        "providerId": "aes-generated",
        "subComponents": {},
        "config": {
          "priority": [
            "100"
          ]
        }
      }
    ]
  },
  "internationalizationEnabled": false,
  "authenticationFlows": [
    {
      "id": "b7de6d5c-aab5-47ae-9ed2-bce42227c3e1",
      "alias": "Account verification options",
      "description": "Method with which to verify the existing account",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "idp-email-verification",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "ALTERNATIVE",
          "priority": 20,
          "autheticatorFlow": true,
          "flowAlias": "Verify Existing Account by Re-authentication",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "682ca9e7-452b-4138-8d96-6d5b19db9fd9",
      "alias": "Browser - Conditional 2FA",
      "description": "Flow to determine if any 2FA is required for the authentication",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "conditional-user-configured",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorConfig": "browser-conditional-credential",
          "authenticator": "conditional-credential",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "auth-otp-form",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 30,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "webauthn-authenticator",
          "authenticatorFlow": false,
          "requirement": "DISABLED",
          "priority": 40,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "auth-recovery-authn-code-form",
          "authenticatorFlow": false,
          "requirement": "DISABLED",
          "priority": 50,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "e3896045-3f11-49b9-afea-f84913f1b672",
      "alias": "Browser - Conditional Organization",
      "description": "Flow to determine if the organization identity-first login is to be used",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "conditional-user-configured",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "organization",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "fbe6380c-43b1-4996-b162-0817ef6642d6",
      "alias": "Direct Grant - Conditional OTP",
      "description": "Flow to determine if the OTP is required for the authentication",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "conditional-user-configured",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "direct-grant-validate-otp",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "b59defb9-a04a-4d8c-98cb-831e10f18589",
      "alias": "First Broker Login - Conditional Organization",
      "description": "Flow to determine if the authenticator that adds organization members is to be used",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "conditional-user-configured",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "idp-add-organization-member",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "572c4113-3b4a-4708-90e1-61ba219d95f2",
      "alias": "First broker login - Conditional 2FA",
      "description": "Flow to determine if any 2FA is required for the authentication",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "conditional-user-configured",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorConfig": "first-broker-login-conditional-credential",
          "authenticator": "conditional-credential",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "auth-otp-form",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 30,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "webauthn-authenticator",
          "authenticatorFlow": false,
          "requirement": "DISABLED",
          "priority": 40,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "auth-recovery-authn-code-form",
          "authenticatorFlow": false,
          "requirement": "DISABLED",
          "priority": 50,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "ca67d28c-610d-4eed-bb61-b085dab3018b",
      "alias": "Handle Existing Account",
      "description": "Handle what to do if there is existing account with same email/username like authenticated identity provider",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "idp-confirm-link",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": true,
          "flowAlias": "Account verification options",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "2d1768fd-8371-4507-9be5-aa703ab38d7c",
      "alias": "Organization",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticatorFlow": true,
          "requirement": "CONDITIONAL",
          "priority": 10,
          "autheticatorFlow": true,
          "flowAlias": "Browser - Conditional Organization",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "f9272efe-039a-4510-9bf6-9de2d076ac2b",
      "alias": "Reset - Conditional OTP",
      "description": "Flow to determine if the OTP should be reset or not. Set to REQUIRED to force.",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "conditional-user-configured",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "reset-otp",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "21733875-d841-45f8-b0ae-91f699344070",
      "alias": "User creation or linking",
      "description": "Flow for the existing/non-existing user alternatives",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticatorConfig": "create unique user config",
          "authenticator": "idp-create-user-if-unique",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "ALTERNATIVE",
          "priority": 20,
          "autheticatorFlow": true,
          "flowAlias": "Handle Existing Account",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "e64a3a92-e5ac-4053-b943-72ef422f3e4e",
      "alias": "Verify Existing Account by Re-authentication",
      "description": "Reauthentication of existing account",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "idp-username-password-form",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "CONDITIONAL",
          "priority": 20,
          "autheticatorFlow": true,
          "flowAlias": "First broker login - Conditional 2FA",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "1297d7bc-c54e-466a-9b2d-faf0ef97ab50",
      "alias": "browser",
      "description": "Browser based authentication",
      "providerId": "basic-flow",
      "topLevel": true,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "auth-cookie",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "auth-spnego",
          "authenticatorFlow": false,
          "requirement": "DISABLED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "identity-provider-redirector",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 25,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "ALTERNATIVE",
          "priority": 26,
          "autheticatorFlow": true,
          "flowAlias": "Organization",
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "ALTERNATIVE",
          "priority": 30,
          "autheticatorFlow": true,
          "flowAlias": "forms",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "6e3560dc-ec8d-4eb8-bf12-3828c1a98425",
      "alias": "clients",
      "description": "Base authentication for clients",
      "providerId": "client-flow",
      "topLevel": true,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "client-secret",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "client-jwt",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "client-secret-jwt",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 30,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "client-x509",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 40,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "federated-jwt",
          "authenticatorFlow": false,
          "requirement": "ALTERNATIVE",
          "priority": 50,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "08a5869a-74e5-4872-9f94-0e98ba18d9a1",
      "alias": "direct grant",
      "description": "OpenID Connect Resource Owner Grant",
      "providerId": "basic-flow",
      "topLevel": true,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "direct-grant-validate-username",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "direct-grant-validate-password",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "CONDITIONAL",
          "priority": 30,
          "autheticatorFlow": true,
          "flowAlias": "Direct Grant - Conditional OTP",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "afff48cc-1dbe-40c1-9770-ef8313690bb6",
      "alias": "docker auth",
      "description": "Used by Docker clients to authenticate against the IDP",
      "providerId": "basic-flow",
      "topLevel": true,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "docker-http-basic-authenticator",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "0d02cfd0-7801-4d0e-9b58-190542052cff",
      "alias": "first broker login",
      "description": "Actions taken after first broker login with identity provider account, which is not yet linked to any Keycloak account",
      "providerId": "basic-flow",
      "topLevel": true,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticatorConfig": "review profile config",
          "authenticator": "idp-review-profile",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": true,
          "flowAlias": "User creation or linking",
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "CONDITIONAL",
          "priority": 60,
          "autheticatorFlow": true,
          "flowAlias": "First Broker Login - Conditional Organization",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "a087128b-7bd2-4f11-ad04-ccf150135087",
      "alias": "forms",
      "description": "Username, password, otp and other auth forms.",
      "providerId": "basic-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "auth-username-password-form",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "CONDITIONAL",
          "priority": 20,
          "autheticatorFlow": true,
          "flowAlias": "Browser - Conditional 2FA",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "d00465f3-23d2-44ca-a73c-371949d850e5",
      "alias": "registration",
      "description": "Registration flow",
      "providerId": "basic-flow",
      "topLevel": true,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "registration-page-form",
          "authenticatorFlow": true,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": true,
          "flowAlias": "registration form",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "05707ce5-68f7-4b48-8cdb-2adb17ac083b",
      "alias": "registration form",
      "description": "Registration form",
      "providerId": "form-flow",
      "topLevel": false,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "registration-user-creation",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "registration-password-action",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 50,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "registration-recaptcha-action",
          "authenticatorFlow": false,
          "requirement": "DISABLED",
          "priority": 60,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "registration-terms-and-conditions",
          "authenticatorFlow": false,
          "requirement": "DISABLED",
          "priority": 70,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "6d7f0354-1820-4e26-ac0e-1ea1e45cfd59",
      "alias": "reset credentials",
      "description": "Reset credentials for a user if they forgot their password or something",
      "providerId": "basic-flow",
      "topLevel": true,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "reset-credentials-choose-user",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "reset-credential-email",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 20,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticator": "reset-password",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 30,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        },
        {
          "authenticatorFlow": true,
          "requirement": "CONDITIONAL",
          "priority": 40,
          "autheticatorFlow": true,
          "flowAlias": "Reset - Conditional OTP",
          "userSetupAllowed": false
        }
      ]
    },
    {
      "id": "28f43219-ccb9-46c6-afa1-c38655a72c8d",
      "alias": "saml ecp",
      "description": "SAML ECP Profile Authentication Flow",
      "providerId": "basic-flow",
      "topLevel": true,
      "builtIn": true,
      "authenticationExecutions": [
        {
          "authenticator": "http-basic-authenticator",
          "authenticatorFlow": false,
          "requirement": "REQUIRED",
          "priority": 10,
          "autheticatorFlow": false,
          "userSetupAllowed": false
        }
      ]
    }
  ],
  "authenticatorConfig": [
    {
      "id": "150e9686-864b-484d-bfcf-b8cd5bc2f59a",
      "alias": "browser-conditional-credential",
      "config": {
        "credentials": "webauthn-passwordless"
      }
    },
    {
      "id": "87cef680-e2ed-42ca-a8fb-7f4f06ed2b25",
      "alias": "create unique user config",
      "config": {
        "require.password.update.after.registration": "false"
      }
    },
    {
      "id": "d40367f4-d64e-4462-903d-3a200805110c",
      "alias": "first-broker-login-conditional-credential",
      "config": {
        "credentials": "webauthn-passwordless"
      }
    },
    {
      "id": "04bf805a-8097-48c0-b860-d8fe73dd8d4d",
      "alias": "review profile config",
      "config": {
        "update.profile.on.first.login": "missing"
      }
    }
  ],
  "requiredActions": [
    {
      "alias": "CONFIGURE_TOTP",
      "name": "Configure OTP",
      "providerId": "CONFIGURE_TOTP",
      "enabled": true,
      "defaultAction": false,
      "priority": 10,
      "config": {}
    },
    {
      "alias": "TERMS_AND_CONDITIONS",
      "name": "Terms and Conditions",
      "providerId": "TERMS_AND_CONDITIONS",
      "enabled": false,
      "defaultAction": false,
      "priority": 20,
      "config": {}
    },
    {
      "alias": "UPDATE_PASSWORD",
      "name": "Update Password",
      "providerId": "UPDATE_PASSWORD",
      "enabled": true,
      "defaultAction": false,
      "priority": 30,
      "config": {}
    },
    {
      "alias": "UPDATE_PROFILE",
      "name": "Update Profile",
      "providerId": "UPDATE_PROFILE",
      "enabled": true,
      "defaultAction": false,
      "priority": 40,
      "config": {}
    },
    {
      "alias": "VERIFY_EMAIL",
      "name": "Verify Email",
      "providerId": "VERIFY_EMAIL",
      "enabled": true,
      "defaultAction": false,
      "priority": 50,
      "config": {}
    },
    {
      "alias": "delete_account",
      "name": "Delete Account",
      "providerId": "delete_account",
      "enabled": false,
      "defaultAction": false,
      "priority": 60,
      "config": {}
    },
    {
      "alias": "UPDATE_EMAIL",
      "name": "Update Email",
      "providerId": "UPDATE_EMAIL",
      "enabled": false,
      "defaultAction": false,
      "priority": 70,
      "config": {}
    },
    {
      "alias": "webauthn-register",
      "name": "Webauthn Register",
      "providerId": "webauthn-register",
      "enabled": true,
      "defaultAction": false,
      "priority": 80,
      "config": {}
    },
    {
      "alias": "webauthn-register-passwordless",
      "name": "Webauthn Register Passwordless",
      "providerId": "webauthn-register-passwordless",
      "enabled": true,
      "defaultAction": false,
      "priority": 90,
      "config": {}
    },
    {
      "alias": "VERIFY_PROFILE",
      "name": "Verify Profile",
      "providerId": "VERIFY_PROFILE",
      "enabled": true,
      "defaultAction": false,
      "priority": 100,
      "config": {}
    },
    {
      "alias": "delete_credential",
      "name": "Delete Credential",
      "providerId": "delete_credential",
      "enabled": true,
      "defaultAction": false,
      "priority": 110,
      "config": {}
    },
    {
      "alias": "idp_link",
      "name": "Linking Identity Provider",
      "providerId": "idp_link",
      "enabled": true,
      "defaultAction": false,
      "priority": 120,
      "config": {}
    },
    {
      "alias": "CONFIGURE_RECOVERY_AUTHN_CODES",
      "name": "Recovery Authentication Codes",
      "providerId": "CONFIGURE_RECOVERY_AUTHN_CODES",
      "enabled": true,
      "defaultAction": false,
      "priority": 130,
      "config": {}
    },
    {
      "alias": "update_user_locale",
      "name": "Update User Locale",
      "providerId": "update_user_locale",
      "enabled": true,
      "defaultAction": false,
      "priority": 1000,
      "config": {}
    }
  ],
  "browserFlow": "browser",
  "registrationFlow": "registration",
  "directGrantFlow": "direct grant",
  "resetCredentialsFlow": "reset credentials",
  "clientAuthenticationFlow": "clients",
  "dockerAuthenticationFlow": "docker auth",
  "firstBrokerLoginFlow": "first broker login",
  "attributes": {
    "cibaBackchannelTokenDeliveryMode": "poll",
    "cibaAuthRequestedUserHint": "login_hint",
    "clientOfflineSessionMaxLifespan": "0",
    "oauth2DevicePollingInterval": "5",
    "clientSessionIdleTimeout": "0",
    "clientOfflineSessionIdleTimeout": "0",
    "cibaInterval": "5",
    "realmReusableOtpCode": "false",
    "cibaExpiresIn": "120",
    "oauth2DeviceCodeLifespan": "600",
    "parRequestUriLifespan": "60",
    "clientSessionMaxLifespan": "0",
    "scimApiEnabled": "false"
  },
  "keycloakVersion": "26.6.1",
  "userManagedAccessAllowed": false,
  "organizationsEnabled": false,
  "verifiableCredentialsEnabled": false,
  "adminPermissionsEnabled": false,
  "scimApiEnabled": false,
  "clientProfiles": {
    "profiles": []
  },
  "clientPolicies": {
    "policies": []
  }
}
```

:::

とても長いのですが、重要な点は以下の通りです。

まずは、交換前のトークンを発行するためのクライアントを作成しています。`serviceAccountsEnabled`が`true`になっている通り、このクライアントはClient Credentials Grantを使用してアクセストークンを取得します。検証の都合上、secretは`source-secret`という固定値にしています。

```json
{
    "id": "2b2593d8-679a-4604-8199-84edf159a151",
    "clientId": "source",
    "name": "",
    "description": "",
    "rootUrl": "",
    "adminUrl": "",
    "baseUrl": "",
    "surrogateAuthRequired": false,
    "enabled": true,
    "alwaysDisplayInConsole": false,
    "clientAuthenticatorType": "client-secret",
    "secret": "source-secret",
    "redirectUris": [
    "/*"
    ],
    "webOrigins": [
    "/*"
    ],
    "notBefore": 0,
    "bearerOnly": false,
    "consentRequired": false,
    "standardFlowEnabled": false,
    "implicitFlowEnabled": false,
    "directAccessGrantsEnabled": false,
    "serviceAccountsEnabled": true,
    "publicClient": false,
    "frontchannelLogout": true,
    "protocol": "openid-connect",
    "attributes": {
    "logout.confirmation.enabled": "false",
    "client.secret.creation.time": "1776915413",
    "standard.token.exchange.enabled": "false",
    "oauth2.jwt.authorization.grant.enabled": "false",
    "frontchannel.logout.session.required": "true",
    "post.logout.redirect.uris": "+",
    "oauth2.device.authorization.grant.enabled": "false",
    "backchannel.logout.revoke.offline.tokens": "false",
    "realm_client": "false",
    "oidc.ciba.grant.enabled": "false",
    "backchannel.logout.session.required": "true",
    "display.on.consent.screen": "false",
    "dpop.bound.access.tokens": "false"
    },
    "authenticationFlowBindingOverrides": {},
    "fullScopeAllowed": true,
    "nodeReRegistrationTimeout": -1,
    "defaultClientScopes": [
    "service_account",
    "web-origins",
    "acr",
    "profile",
    "roles",
    "basic",
    "email"
    ],
    "optionalClientScopes": [
    "address",
    "phone",
    "organization",
    "offline_access",
    "add-target-as-audience",
    "microprofile-jwt"
    ]
}
```

続いて、交換先で利用するクライアントとして`target`を定めています。Token Exchangeを利用するために、クライアントの属性として `"standard.token.exchange.enabled": "true"` を設定しています。こちらも検証の都合上、secretは`target-secret`という固定値にしています。

```json
{
    "id": "181ef7ed-6a04-4238-bc1c-f1c6aaeff2d6",
    "clientId": "target",
    "name": "",
    "description": "",
    "rootUrl": "",
    "adminUrl": "",
    "baseUrl": "",
    "surrogateAuthRequired": false,
    "enabled": true,
    "alwaysDisplayInConsole": false,
    "clientAuthenticatorType": "client-secret",
    "secret": "target-secret",
    "redirectUris": [
        "http://localhost:8000/*"
    ],
    "webOrigins": [
        ""
    ],
    "notBefore": 0,
    "bearerOnly": false,
    "consentRequired": false,
    "standardFlowEnabled": true,
    "implicitFlowEnabled": false,
    "directAccessGrantsEnabled": false,
    "serviceAccountsEnabled": false,
    "publicClient": false,
    "frontchannelLogout": true,
    "protocol": "openid-connect",
    "attributes": {
        "realm_client": "false",
        "oidc.ciba.grant.enabled": "false",
        "client.secret.creation.time": "1776915439",
        "backchannel.logout.session.required": "true",
        "standard.token.exchange.enabled": "true",
        "oauth2.jwt.authorization.grant.enabled": "false",
        "post.logout.redirect.uris": "+",
        "oauth2.device.authorization.grant.enabled": "false",
        "backchannel.logout.revoke.offline.tokens": "false",
        "dpop.bound.access.tokens": "false"
    },
    "authenticationFlowBindingOverrides": {},
    "fullScopeAllowed": true,
    "nodeReRegistrationTimeout": -1,
    "defaultClientScopes": [
        "web-origins",
        "acr",
        "profile",
        "roles",
        "basic",
        "email"
    ],
    "optionalClientScopes": [
        "address",
        "phone",
        "organization",
        "offline_access",
        "microprofile-jwt"
    ]
}
```

また、`source`クライアントに`target`という`aud`を含めるためのクライアントスコープを作成しています。

```json
{
    "id": "467c786f-80c8-4c6f-b9b4-461e74c7efcd",
    "name": "add-target-as-audience",
    "description": "",
    "protocol": "openid-connect",
    "attributes": {
        "include.in.token.scope": "false",
        "display.on.consent.screen": "true",
        "gui.order": "",
        "consent.screen.text": "",
        "include.in.openid.provider.metadata": "true"
    },
    "protocolMappers": [
        {
            "id": "b9b8927d-c721-491a-8937-d4c0a462ac51",
            "name": "add-target",
            "protocol": "openid-connect",
            "protocolMapper": "oidc-audience-mapper",
            "consentRequired": false,
            "config": {
            "included.client.audience": "target",
            "id.token.claim": "false",
            "lightweight.claim": "false",
            "introspection.token.claim": "true",
            "access.token.claim": "true",
            "userinfo.token.claim": "false"
            }
        }
    ]
}
```

### 3. Kong Gatewayの設定

Kong Gatewayでは、Keycloak(source)から取得したアクセストークンを検証し、Token Exchangeを利用して新しいトークンと交換するための設定を行います。設定は、decKを利用し宣言的に定義していますが、GUIから設定しても同様です。

```yaml
_format_version: "3.0"

# http://localhost:8000/mockへアクセスされた時の転送先を指定する
services:
  - name: httpbin-service
    url: https://httpbin.org
    routes:
      - name: httpbin-route
        # 3.14からデフォルトのプロトコルがhttpsとなったので、従来通りhttpでもアクセスしたい場合は指定必須
        protocols:
          - http
          - https
        paths:
          - /mock
        strip_path: true

plugins:
  - name: openid-connect
    service: httpbin-service
    config:
      # 各メタデータ（token_endpoint, ...）のロードやトークンの検証時に利用
      issuer: http://localhost:8080/realms/kong
      # 本来、Discovery Endpointから自動的にロードされるので、設定は不要だが、Docker環境で実施しているため必要
      # 設定しない場合、jwks_endpoint, token_endpointはhttp://localhost:8080となり、Kongコンテナからは到達不可となる
      jwks_endpoint: http://keycloak:8080/realms/kong/protocol/openid-connect/certs
      token_endpoint: http://keycloak:8080/realms/kong/protocol/openid-connect/token
      # Token Exchangeで交換する先のClient ID/Secretを指定する
      client_id:
        - target
      client_secret:
        - target-secret
      auth_methods:
        - bearer
      # Token Exchange用のパラメータを指定する
      token_exchange:
        # 交換に利用するIssuerを指定する
        subject_token_issuers:
          - issuer: http://localhost:8080/realms/kong
            # トークンの交換を行う条件を指定する
            conditions:
              # audにtargetを有している場合、Token Exchangeが行われる
              has_audience:
                - target
              has_scopes:
              missing_audience:
              missing_scopes:
        # Token Exchangeで利用されるパラメータ
        request:
          scopes:
            - profile
  - name: opentelemetry
    config:
      traces_endpoint: http://otel-lgtm:4318/v1/traces
      logs_endpoint: http://otel-lgtm:4318/v1/logs
      access_logs_endpoint: http://otel-lgtm:4318/v1/logs
      metrics:
        endpoint: http://otel-lgtm:4318/v1/metrics
        push_interval: 5
        enable_bandwidth_metrics: true
        enable_consumer_attribute: true
        enable_latency_metrics: true
        enable_request_metrics: true
        enable_upstream_health_metrics: true
      resource_attributes:
        service.name: kong-gateway
      propagation:
        default_format: w3c
        extract:
          - w3c
        inject:
          - w3c
```

:::message

今回は検証用のため、Client Secretなどの情報をそのまま宣言ファイルに記載していますが、通常ではVaultから参照したり、`${{ env "DECK_CLIENT_SECRET" }}`のような形式で環境変数から参照させる方法を推奨します。

:::

#### 4. 実際に動作を見てみる

まずは、`source`クライアントからClient Credentials Grantを利用してアクセストークンを取得します。

```sh
curl --request POST \
  --url http://localhost:8080/realms/kong/protocol/openid-connect/token \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=client_credentials \
  --data client_id=source \
  --data client_secret=source-secret \
  --data scope=add-target-as-audience
```

実行結果：

```json
{"access_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJEbFh5TDIxWHhxalY2TFZsMkZodThGOGhpYmxYLVhJZ2dNaXhxa0hqcDlRIn0.eyJleHAiOjE3NzY5Mjc2OTcsImlhdCI6MTc3NjkyNzM5NywianRpIjoidHJydGNjOmI2ZjI4YmJhLTRlOGUtMWM2My05MDc5LTYxOWE0NmQyMmM5MCIsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC9yZWFsbXMva29uZyIsImF1ZCI6InRhcmdldCIsInN1YiI6ImI0NzY4ZTE5LTJkMDAtNDNhYy1iNjc1LTY0N2UzNzBjNmY3YyIsInR5cCI6IkJlYXJlciIsImF6cCI6InNvdXJjZSIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiLyoiXSwic2NvcGUiOiJlbWFpbCBwcm9maWxlIiwiY2xpZW50SG9zdCI6IjE3Mi4xOC4wLjEiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsInByZWZlcnJlZF91c2VybmFtZSI6InNlcnZpY2UtYWNjb3VudC1zb3VyY2UiLCJjbGllbnRBZGRyZXNzIjoiMTcyLjE4LjAuMSIsImNsaWVudF9pZCI6InNvdXJjZSJ9.iFEFdb2MUs06xpyM1X4cXgZXSJGXkwTRRgyFQdKGGnPIXbTTsKMT2dJCWOndTnsBDGLNe6gJWVRL2dNCA3ubQ6zXZ9sa_1Fz8Aia9NJCgE4KpsiJRaWlZBww2jBAdi8Fs24ngJRSIYFGdQfXcI2OIGvU6Fo_aog2PqJq6xO44xzc045PB9X0hcwwL7sJ-JcRitNe5jpCVxdThcYeHdAM3oLXiMyCj0SvTGMd8e2_THnro1XoEcJYmWQL0NMQqHIdxdNEcGfVRzmwWvk4xLd6uE7FyegyewKEFOD9UnkAIDUoNmVuk6NRvJ8v-a9EawPTVf9p_ywGMuCWM7251jenAQ","expires_in":300,"refresh_expires_in":0,"token_type":"Bearer","not-before-policy":0,"scope":"email profile"}
```

`$.access_token`をデコードしてみると以下のようになっています。

```sh
export AT=eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJEbFh5TDIxWHhxalY2TFZsMkZodThGOGhpYmxYLVhJZ2dNaXhxa0hqcDlRIn0.eyJleHAiOjE3NzY5Mjc2OTcsImlhdCI6MTc3NjkyNzM5NywianRpIjoidHJydGNjOmI2ZjI4YmJhLTRlOGUtMWM2My05MDc5LTYxOWE0NmQyMmM5MCIsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC9yZWFsbXMva29uZyIsImF1ZCI6InRhcmdldCIsInN1YiI6ImI0NzY4ZTE5LTJkMDAtNDNhYy1iNjc1LTY0N2UzNzBjNmY3YyIsInR5cCI6IkJlYXJlciIsImF6cCI6InNvdXJjZSIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiLyoiXSwic2NvcGUiOiJlbWFpbCBwcm9maWxlIiwiY2xpZW50SG9zdCI6IjE3Mi4xOC4wLjEiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsInByZWZlcnJlZF91c2VybmFtZSI6InNlcnZpY2UtYWNjb3VudC1zb3VyY2UiLCJjbGllbnRBZGRyZXNzIjoiMTcyLjE4LjAuMSIsImNsaWVudF9pZCI6InNvdXJjZSJ9.iFEFdb2MUs06xpyM1X4cXgZXSJGXkwTRRgyFQdKGGnPIXbTTsKMT2dJCWOndTnsBDGLNe6gJWVRL2dNCA3ubQ6zXZ9sa_1Fz8Aia9NJCgE4KpsiJRaWlZBww2jBAdi8Fs24ngJRSIYFGdQfXcI2OIGvU6Fo_aog2PqJq6xO44xzc045PB9X0hcwwL7sJ-JcRitNe5jpCVxdThcYeHdAM3oLXiMyCj0SvTGMd8e2_THnro1XoEcJYmWQL0NMQqHIdxdNEcGfVRzmwWvk4xLd6uE7FyegyewKEFOD9UnkAIDUoNmVuk6NRvJ8v-a9EawPTVf9p_ywGMuCWM7251jenAQ
jwt decode $AT
```

実行結果：

```sh
Token header
------------
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "DlXyL21XxqjV6LVl2Fhu8F8hiblX-XIggMixqkHjp9Q"
}

Token claims
------------
{
  "acr": "1",
  "allowed-origins": [
    "/*"
  ],
  "aud": "target",
  "azp": "source",
  "clientAddress": "172.18.0.1",
  "clientHost": "172.18.0.1",
  "client_id": "source",
  "email_verified": false,
  "exp": 1776927697,
  "iat": 1776927397,
  "iss": "http://localhost:8080/realms/kong",
  "jti": "trrtcc:b6f28bba-4e8e-1c63-9079-619a46d22c90",
  "preferred_username": "service-account-source",
  "scope": "email profile",
  "sub": "b4768e19-2d00-43ac-b675-647e370c6f7c",
  "typ": "Bearer"
}
```

`$.azp`をみると、このアクセストークンは、`source`クライアントで認可されたものだと確認できます。

続いて、このアクセストークンを元にKong Gatewayへリクエストを投げてみましょう。

```sh
curl --request GET \
  --url http://localhost:8000/mock/anything \
  --header "Authorization: Bearer $AT"
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
    "Authorization": "Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJEbFh5TDIxWHhxalY2TFZsMkZodThGOGhpYmxYLVhJZ2dNaXhxa0hqcDlRIn0.eyJleHAiOjE3NzY5MjgyNjIsImlhdCI6MTc3NjkyNzk2MiwianRpIjoidHJydHRlOjc2YmVmZjU5LTI4ODctNGQ2Yy1lYmViLTJmZTAxOTFjOTlhOSIsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC9yZWFsbXMva29uZyIsInN1YiI6ImI0NzY4ZTE5LTJkMDAtNDNhYy1iNjc1LTY0N2UzNzBjNmY3YyIsInR5cCI6IkJlYXJlciIsImF6cCI6InRhcmdldCIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiIl0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwicHJlZmVycmVkX3VzZXJuYW1lIjoic2VydmljZS1hY2NvdW50LXNvdXJjZSJ9.bdebEDiDkxB6O-i4Yy1dqAqjsVCFhGVcM_eXpWPEz8Es1XEj3cQsHB5pezkGCKmtubbPNkRnJsLgsJiIDYX7Gl9bVtAj30bI7xg4ayleKZrlfpdM6NpswiE7n7Yk_4senn5xd5ariqeG3bHrvEuIU4kgSqPjejKAd559BR1qXKvKVKuUHORgfz9W9ZgR3qaG4Zsboc6OW33fs1UxZXuc-5Ebq0dLN4OniXIRr8axL7q9L467Lu5Mw0jJ05ObnJBmQ1P0gn8YmPavl-qXSBZC6Uj2gGBQ0hxH91mb4DnIKHjgEqi62mhtW18T_j39UaAp1KencpxFXM7Na1h4qULQpQ", 
    "Host": "httpbin.org", 
    "Traceparent": "00-1d372f4326f245efe65c6e972b4cc124-4c7dcb61c86eb3cd-01", 
    "User-Agent": "curl/8.7.1", 
    "X-Amzn-Trace-Id": "Root=1-69e9c4db-5181119f7159cb605c463f3c", 
    "X-Forwarded-Host": "localhost", 
    "X-Forwarded-Path": "/mock/anything", 
    "X-Forwarded-Prefix": "/mock", 
    "X-Kong-Request-Id": "ae6b571a77743a3d9766a65419ab1616"
  }, 
  "json": null, 
  "method": "GET", 
  "origin": "172.66.0.243, 126.218.11.92", 
  "url": "https://localhost/anything"
}
```

ここで`Authorization`ヘッダーに含まれるBearerトークンの値を再度確認してみましょう。

```sh
export NEW_AT=eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJEbFh5TDIxWHhxalY2TFZsMkZodThGOGhpYmxYLVhJZ2dNaXhxa0hqcDlRIn0.eyJleHAiOjE3NzY5MjgyNjIsImlhdCI6MTc3NjkyNzk2MiwianRpIjoidHJydHRlOjc2YmVmZjU5LTI4ODctNGQ2Yy1lYmViLTJmZTAxOTFjOTlhOSIsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC9yZWFsbXMva29uZyIsInN1YiI6ImI0NzY4ZTE5LTJkMDAtNDNhYy1iNjc1LTY0N2UzNzBjNmY3YyIsInR5cCI6IkJlYXJlciIsImF6cCI6InRhcmdldCIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiIl0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwicHJlZmVycmVkX3VzZXJuYW1lIjoic2VydmljZS1hY2NvdW50LXNvdXJjZSJ9.bdebEDiDkxB6O-i4Yy1dqAqjsVCFhGVcM_eXpWPEz8Es1XEj3cQsHB5pezkGCKmtubbPNkRnJsLgsJiIDYX7Gl9bVtAj30bI7xg4ayleKZrlfpdM6NpswiE7n7Yk_4senn5xd5ariqeG3bHrvEuIU4kgSqPjejKAd559BR1qXKvKVKuUHORgfz9W9ZgR3qaG4Zsboc6OW33fs1UxZXuc-5Ebq0dLN4OniXIRr8axL7q9L467Lu5Mw0jJ05ObnJBmQ1P0gn8YmPavl-qXSBZC6Uj2gGBQ0hxH91mb4DnIKHjgEqi62mhtW18T_j39UaAp1KencpxFXM7Na1h4qULQpQ
jwt decode $NEW_AT
```

実行結果：

```sh
Token header
------------
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "DlXyL21XxqjV6LVl2Fhu8F8hiblX-XIggMixqkHjp9Q"
}

Token claims
------------
{
  "acr": "1",
  "allowed-origins": [
    ""
  ],
  "azp": "target",
  "email_verified": false,
  "exp": 1776928262,
  "iat": 1776927962,
  "iss": "http://localhost:8080/realms/kong",
  "jti": "trrtte:76beff59-2887-4d6c-ebeb-2fe0191c99a9",
  "preferred_username": "service-account-source",
  "scope": "email profile",
  "sub": "b4768e19-2d00-43ac-b675-647e370c6f7c",
  "typ": "Bearer"
}
```

`$.azp`を確認すると、このアクセストークンは`target`によって認可されたものだと確認することができます。

## 終わりに

今回は、Kong Gateway 3.14でOpenID Connectプラグインに追加されたToken Exchangeを試してみました。従来は、Request Calloutプラグインやカスタムプラグインを活用し、この手のパターンを設定していましたが、ついにプラグインとしてサポートするようになったので、従来よりも簡単に設定できるようになったと思います。
