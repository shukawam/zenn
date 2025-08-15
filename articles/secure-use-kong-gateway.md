---
title: "Kong Gateway を安全に運用するための認証・認可"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: true
---

## はじめに

こんにちは 🖐️ 今回は、API Gateway の機能や使い方を解説するというよりは、**Kong Gateway 自体**を安全に運用する観点で記事を書いていきたいと思います。オペレーションミスで設定を消してしまった！全世界に Kong Admin API や Kong Manager を公開していて誰でも更新できる状態になってしまっていた！などということがないように、きちんと保護してセキュアに運用していきましょう。安全に運用するという観点だともっとさまざまな観点[^1]が考えられそうですが、本記事では認証と認可に焦点を当てて解説していきたいと思います。

## Kong Gateway の設定

Kong Gateway の設定方法は以下の通りいくつか存在します。

- Kong Manager という専用の UI を用いて行う
- decK と呼ばれる CLI を用いて宣言的に行う
- Admin API を実行することで設定を行う

いずれも最終的には Admin API が実行されるので、本記事では Kong Gateway を安全に運用する = Admin API を安全に運用する（+ Kong Manager 自体の認証をきちんと行う）と読み替えてもらっても差し支えないです。簡易的なイメージを以下に記します。

![admin-api-overview](/images/secure-use-kong-gateway/admin-api-ovreview.png)
_Kong Gateway の設定イメージ_

Kong Manager から操作を行う際は、裏側で Admin API が呼び出されて設定を行いますし、decK を用いた場合には状態ファイルを元に呼び出す Admin API を決定し自動的に呼び出されますし、そのまま Admin API を呼び出すこともできる、と言ったイメージです。

## Kong Gateway を安全に運用するための設定例

Kong Gateway は、エンタープライズエディションを利用していることを前提としています。また、紹介する設定例は Docker Compose を用いており、IdP としては Auth0 を利用しています。以降では、Kong Manager の保護と Admin API の保護という観点でみていきます。

:::message
記事自体がコード全量を載せるととても長くなりそうなので、要点の解説に絞っています。全量を見たい、自分で動かしてみて試したい方は以下のリポジトリをご参照ください。

[https://github.com/shukawam/kong-security](https://github.com/shukawam/kong-security)
:::

### Kong Manager の保護

Kong Manager の保護は、**誰が Kong Manager にアクセス可能なのか？** という認証によって行われます。アクセス可能な上で、どんな操作が可能なのか？という、いわゆる認可部分に関しては、Admin API の認可制御で実現されているので、[Admin API の保護](#admin-api-の保護)で詳しく解説します。

まず、Kong Manager の認証方式は以下がサポートされています。

- Basic 認証
- OpenID Connect
- LDAP 認証

本記事では、この中でもよく用いられている Basic 認証と OpenID Connect を取り扱います。

#### Basic 認証

Kong Gateway で保管されている ID/パスワードに基づいて本人性を検証します。3 つの認証方式の中では最も簡単に導入できますが、認証処理に関するカスタマイズ性（パスワードの複雑性）や認証強度は最も低いです。設定を有効にするためには、環境変数に以下の値を設定し、Kong Gateway を起動します。

```yaml:compose.yaml
environment:
    # ... 省略 ...
    KONG_ADMIN_GUI_HOST: http://localhost:8002
    KONG_LICENSE_DATA: ${KONG_LICENSE_DATA:-}
    # Kong Manager Authentication Settings
    KONG_ENFORCE_RBAC: on
    # Enable Basic Auth for Kong Manager
    KONG_ADMIN_GUI_AUTH: basic-auth
    KONG_PASSWORD: kong
    KONG_ADMIN_GUI_SESSION_CONF: |
      { "secret": "kong" }
```

この状態で Kong Manager (`http://localhost:8002`)にアクセスすると、ID/パスワードによる認証が求められ `kong_admin/kong` を入力することで Kong Manager にアクセスすることができるようになります。

![basic-authn](/images/secure-use-kong-gateway/basic-authn.png)
_Kong Manager で ID/パスワードによる認証が求められる様子_

これに加えて、セキュリティ観点でよく設定する項目としては以下があげられます。（Docker で指定する時は、Prefix に `KONG_` をつければ良いです）

- admin_gui_auth_password_complexity: パスワードの複雑性を決定するためのパラメータ
- admin_gui_auth_login_attempts: ログインの試行回数上限（0 に設定した場合は、無限回の施行が許可される）
- admin_gui_auth_login_attempts_ttl: ログイン施行履歴の保持期間

その他の項目は以下をご参照ください。

https://developer.konghq.com/gateway/configuration/#kong-manager-section

#### OpenID Connect

外部の Identity Provider(以下、IdP)に認証処理を移譲する方式です。IdP が認証した結果を ID トークンという形で Kong Gateway に渡し、Kong Gateway がそれを検証する[^2]ことで認証連携を行います。Basic 認証に比べると、導入には若干の手間がかかりますが IdP 側が持っているさまざまな機能（e.g. パスワードのローテーション、パスワードレス認証、etc.）が活用できる点がメリットです。簡易的な動作イメージは以下の通りです。

![openid-connect](/images/secure-use-kong-gateway/openid-connect.png)
_OpenID Connect を用いた Kong Manager の保護イメージ_

設定を有効にするには、環境変数に以下の値を設定し、Kong Gateway を起動します。

```yaml
environment:
  # ... 省略 ...
  KONG_ADMIN_GUI_HOST: http://localhost:8002
  KONG_LICENSE_DATA: ${KONG_LICENSE_DATA:-}
  # Kong Manager Authentication Settings
  KONG_ENFORCE_RBAC: on
  # Enable OpenID Connect for Kong Manager
  # see: https://developer.konghq.com/plugins/openid-connect/reference/
  KONG_ADMIN_GUI_AUTH: openid-connect
  KONG_ADMIN_GUI_AUTH_CONF: |
    { 
      "issuer": "${ISSUER}",
      "client_id": ["${CLIENT_ID}"],
      "client_secret": ["${CLIENT_SECRET}"],
      "authenticated_groups_claim": ["${AUTHENTICATED_GROUPS_CLAIM}"],
      "redirect_uri": ["http://localhost:8001/auth"],
      "login_redirect_uri": ["http://localhost:8002"],
      "logout_redirect_uri": ["http://localhost:8002"],
      "scopes": ["openid", "profile", "email"],
      "admin_auto_create_rbac_token_disabled": false
    }
```

OpenID Connect による認証連携は、Kong の OpenID Connect プラグインが利用されているため、`KONG_ADMIN_GUI_AUTH_CONF` の設定項目は、以下のドキュメントを参照すると良いです。

https://developer.konghq.com/plugins/openid-connect/reference/

上記の設定例(`KONG_ADMIN_GUI_AUTH_CONF`)を簡単に解説します。

- `issuer`: Issuer(誰が認証したのか？)を指定します。Kong の OIDC プラグインは、Issuer の項目に Discovery Endpoint を指定すると公開されているメタデータを読み取って適切に設定してくれるので、Discovery Endpoint が公開されている場合はこちらを指定すると良いでしょう。
- `client_id`: Client ID を指定します。（OP/RP 間の認証で利用されます）
- `client_secret`: Client Secret を指定します。（OP/RP 間の認証で利用されます）
- `authenticated_groups_claim`: ID トークン内の任意のクレームを Kong 側のグループ情報にマッピングするために使用します。認証用途では用いないですが、認可処理で必要なパラメータのため指定しています。
- `redirect_uri`: OP での認証完了後のリダイレクト先を指定します。パスは `/auth` を指定する必要があるため、ホストしている環境によってホスト名を適宜変えてください。
- `login_redirect_uri`: ID トークンの検証後、リダイレクトする先を指定します。
- `logout_redirect_uri`: ログアウト後に、リダイレクトする先を指定します。
- `scopes`: OpenID Connect として必要なスコープを指定します。上記例では、`profile`, `email` も指定していますが最低限、`openid` が必要です。
- `admin_auto_create_rbac_token_disabled`: RBAC トークンと呼ばれる Admin API を実行する際の資格情報を Kong の Admin ユーザーに対して自動的に生成するかどうかを決定します。

この状態で Kong Manager (`http://localhost:8002`)にアクセスすると、OpenID Connect による認証が開始し、私の例だと Auth0 に認証処理が移譲されます。適切な認証情報を入力することで Kong Manager にアクセスすることができるようになります。

![openid-connect-flow-1](/images/secure-use-kong-gateway/openid-connect-flow-1.png)
_OpenID Connect による認証連携が有効になっている様子_

![openid-connect-flow-2](/images/secure-use-kong-gateway/openid-connect-flow-2.png)
_IdP(Auth0)に認証処理が移譲されている様子_

Kong Manager の保護に関しては以上です。

### Admin API の保護

Admin API の保護は、**誰が Admin API にアクセス可能なのか？** という認証に加え、**どんな操作が許可されているのか？** という認可によって行われます。まず、Kong Gateway では以下の通り 2 種類のユーザーが存在します。

- Admin ユーザー: Kong Manager に加え、Admin API もアクセス可能なユーザー
- RBAC ユーザー: Admin API のみアクセス可能なユーザー

Admin ユーザー、RBAC ユーザーともに RBAC トークンを用いて Admin API にアクセス可能かを判別します。Admin ユーザーは、何らかのロールが必ずアタッチされていますが、RBAC ユーザーは 1 個以上のロールをアタッチすることで、そのロールに基づいた認可制御（RBAC | Role Based Access Control）を行うことができます。簡易的な動作イメージは以下の通りです。

![rbac-overview](/images/secure-use-kong-gateway/rbac-overview.png)
_Admin API の保護イメージ_

ロールは、Kong Gateway 側でデフォルトで用意されているロール（下表参照）を利用することもできますし、自分で API リソースとそれに対する操作（CRUD）を選択してカスタムロールを作成することも可能です。

| ロール        | 概要                                                             |
| ------------- | ---------------------------------------------------------------- |
| `super-admin` | 全ての API リソースと全ての操作が可能（RBAC 系のリソースを含む） |
| `admin`       | 全ての API リソースと全ての操作が可能（RBAC 系のリソースを除く） |
| `read-only`   | 全ての API リソースに対して参照操作が可能                        |

動作をイメージしやすいように簡易的な検証を行ってみます。まずは、`platform-viewer` という RBAC ユーザーを作成します。このユーザーには、`read-only` というデフォルトで用意されている参照のみが可能なロールをアタッチします。

![create-rbac-user](/images/secure-use-kong-gateway/create-rbac-user.png)
_platform-viewer という RBAC ユーザーを作成する様子_

まずは、このユーザーで RBAC トークンなしで Admin API をアクセスしてみます。

```sh
curl -i http://localhost:8001/services
```

Admin API を実行するための RBAC トークンが存在しないため、以下のような認証エラーが返却されます。

```sh
HTTP/1.1 401 Unauthorized
Date: Fri, 15 Aug 2025 07:16:52 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
X-Kong-Admin-Request-ID: 49cd615d41b176337534346f9a93a003
Access-Control-Allow-Origin: *
Content-Length: 69
X-Kong-Admin-Latency: 6
Server: kong/3.11.0.0-enterprise-edition

{"message":"Invalid credentials. Token or User credentials required"}
```

適切な RBAC トークンを含めて同様の API を実行してみます。

```sh
curl http://localhost:8001/services \
    -i -H "Kong-Admin-Token:password"
```

実行すると以下のような結果が返却されます。サービスは作成していないので、結果は 0 件ですがステータスコードが 200 で返却されることが確認できます。

```sh
HTTP/1.1 200 OK
Date: Fri, 15 Aug 2025 07:16:17 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
X-Kong-Admin-Request-ID: b3cd175b2230e141b778ae43f5ae6260
Access-Control-Allow-Origin: *
Content-Length: 23
X-Kong-Admin-Latency: 12
Server: kong/3.11.0.0-enterprise-edition

{"next":null,"data":[]}
```

`platform-viewer` は、`read-only` ロールが付与されているため、参照操作は可能ですが更新系の操作は失敗します。例えば、サービスの作成を同じ RBAC トークンを用いて実行してみます。

```sh
curl -X POST http://localhost:8001/services \
    -i -H Kong-Admin-Token:password \
    -d '{
        "name": "httpbin-service",
        "url": "https://httpbin.org"
    }'
```

以下のような結果が返却され、更新系の操作が権限不足により失敗することが確認できます。

```sh
HTTP/1.1 403 Forbidden
Date: Fri, 15 Aug 2025 07:17:50 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
X-Kong-Admin-Request-ID: ffc95e7cf5423b7d28c8a3118b81a938
Access-Control-Allow-Origin: *
Content-Length: 82
X-Kong-Admin-Latency: 48
Server: kong/3.11.0.0-enterprise-edition

{"message":"platform-viewer, you do not have permissions to create this resource"}
```

## おわりに

今回は、Kong Gateway を安全に運用するための方法を認証・認可の観点から解説してみました。次は、実際にプロキシしている API に対する保護観点（認証と認可）を解説したいと思いますので、お楽しみに 😎

[^1]: 例えば、ネットワークセキュリティによって到達可能な IP を制限することや TLS による暗号化通信など
[^2]: ID トークンに含まれる署名を検証することで通信経路でトークンが改ざんされていないこと、誰が？(`iss`)・誰を？(`sub`)・誰のために？(`aud`)・いつ？(`iat`)・どんな認証方式で？(`acr`)などの情報を検証することで第三者による認証結果を信頼する
