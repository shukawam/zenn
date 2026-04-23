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

@[gist](https://gist.github.com/shukawam/f8ad84fb6e8e23a88064cb21c2cf6741)

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
