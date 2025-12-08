---
title: "S3 を利用した Kong Gateway Data Plane の復旧シナリオ"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: true
published_at: 2025-12-17 00:00
---

## はじめに

こんにちは 🖐️ この記事は、[Kong Advent Calendar 2025](https://qiita.com/advent-calendar/2025/kong) の Day17 として書かれています。今回は、タイトルにある通り Kong Gateway の Control Plane（以下、CP） がダウンした時に Data Plane（以下、DP） を復旧させるシナリオとして S3 などのオブジェクトストレージを利用する方法を紹介します。

## 障害発生時の動き

まずは Kong Gateway を構成する各要素で障害が発生した場合の動きを説明します。尚、ここでは CP と DP が分離されているハイブリッドモードを前提とします。

![kong-hybrid-mode](/images/kong-cp-recovery-s3-backup/kong-hybrid-mode.png)
_Kong Gateway ハイブリッドモードの概要_

CP は、開発者から Kong Gateway の設定を API 経由で受け取り、その設定情報をデータベースへ保存します。その後、現在の設定と差分がある場合は、DP へ設定情報の配信を行います。DP が実際の API トラフィックを受け取る役割を担うため、基本的に CP がダウンしても DP は API トラフィックを処理し続けます。

![cp-down-scenario](/images/kong-cp-recovery-s3-backup/cp-down-scenario.png)
_CP ダウン時にも DP は API トラフィックを処理し続ける_

しかし、CP がダウンしている場合は、新しい設定を DP へ配信することができません。そのため、CP が復旧するまでの間は、DP 上にキャッシュされている設定情報を元に API トラフィックを処理し続けます。当然ですが、DP がダウンしている場合は、実際の API トラフィックを処理できなくなるため、冗長化構成を組んだり、回復力のあるインフラとその仕組み（例えば、Kubernetes など）を利用することが重要です。

ここで、CP がダウンしている場合に DP がダウンしてしまい、新しい DP が起動したシナリオを考えてみましょう。図示すると以下のようになります。

![dp-self-healing](/images/kong-cp-recovery-s3-backup/dp-self-healing.png)
_CP ダウン時に DP が回復もしくは、スケールアウトした場合_

この場合、DP は CP との疎通が取れないため、プロビジョニングを完了することができず、API トラフィックを処理できない状態となってしまいます。このような状態に備えて、Kong Gateway では S3 などのオブジェクトストレージから設定ファイルを読み込み、DP を起動することが可能な機能が提供されています。

## S3 を利用した耐障害性シナリオ

:::message
ドキュメントにも明記されている通り、この機能は厳格な SLA を遵守する必要があるユーザーに向けて提供されている機能です。遵守する必要がある SLA やメンテナンス負荷などを鑑みて、利用を検討してください。

https://developer.konghq.com/gateway/cp-outage/
:::

動作のイメージ図は以下の通りです。

![s3-backup-overview](/images/kong-cp-recovery-s3-backup/s3-backup-overview.png)
_S3 を利用した耐障害性シナリオの概要_

見てもらえるとわかる通り、1 つ以上のバックアップノードと呼ばれる DP が S3 へ Kong Gateway の設定情報を自動的にプッシュします。仮に新しい DP が起動時に CP との疎通が取れない場合は、S3 から設定情報を読み込み、取得した設定ファイルを用いて、API トラフィックの処理を開始します。

いくつか、ドキュメントに注意事項が記載されているので整理します。

1. ストレージ内の設定データの暗号化はユーザーの責任であること
2. バックアップノードは、実際のトラフィックを受ける用途としては推奨されないこと
   - S3 へのバックアップ処理を担当するため、実際の API トラフィックを処理するとアタックサーフェスを拡大してしまう可能性があること
   - バックアップ処理が実際の API トラフィックの遅延時間の増加要因となること

### 実際に試してみる

実際に、S3 を使用した耐障害性シナリオを試してみます。ここでは、以下の条件を前提とします。

- Kong Gateway は、ローカルの Docker Compose で動作させています
- Kong Gateway の Enterprise Edition を使用していること
- 読み書き可能な S3 バケットが存在すること

まずは、実現するための `compose.yaml` を以下のように用意します。

```yaml:compose.yaml
x-default: &default
  networks:
    - kong-network
  restart: on-failure
x-kong-env: &kong-env
  KONG_DATABASE: postgres
  KONG_PG_HOST: database
  KONG_PG_PASSWORD: kong
  KONG_PASSWORD: kong
x-aws-env: &aws-env
  AWS_REGION: ap-northeast-1
  AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID:-}
  AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY:-}
  KONG_CLUSTER_FALLBACK_CONFIG_STORAGE: s3://${AWS_S3_BUCKET_NAME}/kong

networks:
  kong-network:

services:
  database:
    <<: *default
    image: postgres:17
    container_name: database
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kong
    ports:
      - 5432:5432
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kong"]
      interval: 30s
      timeout: 30s
      retries: 3
  kong-bootstrap:
    <<: *default
    image: &kong-gateway-image kong/kong-gateway:3.12
    container_name: kong-bootstrap
    depends_on:
      database:
        condition: service_healthy
    command: kong migrations bootstrap
    environment:
      <<: *kong-env
  kong-cp:
    <<: *default
    image: *kong-gateway-image
    container_name: kong-cp
    depends_on:
      kong-bootstrap:
        condition: service_completed_successfully
    ports:
      - 8001:8001
      - 8002:8002
      - 8100:8100
    environment:
      <<: *kong-env
      KONG_ROLE: control_plane
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
      KONG_ADMIN_GUI_LISTEN: 0.0.0.0:8002
      KONG_CLUSTER_LISTEN: 0.0.0.0:8005
      KONG_TELEMETRY_LISTEN: 0.0.0.0:8006
      KONG_STATUS_LISTEN: 0.0.0.0:8100
      KONG_ADMIN_GUI_HOST: http://localhost:8002
      KONG_LICENSE_DATA: ${KONG_LICENSE_DATA:-}
      KONG_TRACING_INSTRUMENTATIONS: all
      KONG_TRACING_SAMPLING_RATE: 1.0
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_CLUSTER_CERT: /etc/kong/certs/tls.crt
      KONG_CLUSTER_CERT_KEY: /etc/kong/certs/tls.key
    volumes:
      - ./certs:/etc/kong/certs
  kong-dp-exporter:
    <<: *default
    image: *kong-gateway-image
    container_name: kong-dp-exporter
    depends_on:
      kong-bootstrap:
        condition: service_completed_successfully
    environment:
      KONG_ROLE: data_plane
      KONG_DATABASE: off
      KONG_CLUSTER_SERVER_NAME: kong-cluster
      KONG_CLUSTER_CONTROL_PLANE: kong-cp:8005
      KONG_CLUSTER_TELEMETRY_ENDPOINT: kong-cp:8006
      KONG_CLUSTER_TELEMETRY_SERVER_NAME: kong-cp
      KONG_CLUSTER_MTLS: shared
      KONG_LICENSE_DATA: ${KONG_LICENSE_DATA:-}
      KONG_TRACING_INSTRUMENTATIONS: all
      KONG_TRACING_SAMPLING_RATE: 1.0
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_CLUSTER_CERT: /etc/kong/certs/tls.crt
      KONG_CLUSTER_CERT_KEY: /etc/kong/certs/tls.key
      # ... 1
      <<: *aws-env
      KONG_CLUSTER_FALLBACK_CONFIG_EXPORT: "on"
    volumes:
      - ./certs:/etc/kong/certs
  kong-dp-importer:
    <<: *default
    image: *kong-gateway-image
    container_name: kong-dp-importer
    depends_on:
      kong-bootstrap:
        condition: service_completed_successfully
    ports:
      - 8000:8000
    environment:
      KONG_ROLE: data_plane
      KONG_DATABASE: off
      KONG_CLUSTER_CONTROL_PLANE: kong-cp:8005
      KONG_CLUSTER_MTLS: shared
      KONG_LICENSE_DATA: ${KONG_LICENSE_DATA:-}
      KONG_TRACING_INSTRUMENTATIONS: all
      KONG_TRACING_SAMPLING_RATE: 1.0
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_CLUSTER_CERT: /etc/kong/certs/tls.crt
      KONG_CLUSTER_CERT_KEY: /etc/kong/certs/tls.key
      # ... 1
      <<: *aws-env
      KONG_CLUSTER_FALLBACK_CONFIG_IMPORT: "on"
    volumes:
      - ./certs:/etc/kong/certs

  deck:
    <<: *default
    image: kong/deck:v1.54.0
    working_dir: /files
    container_name: deck
    command:
      ["gateway", "sync", "--kong-addr", "http://kong-cp:8001", "kong.yaml"]
    volumes:
      - ./kong.yaml:/files/kong.yaml
    depends_on:
      - kong-cp
```

少し長いですが、重要なポイントは以下の部分だけです。

```yaml:
x-aws-env: &aws-env
  AWS_REGION: ${AWS_REGION:-ap-northeast-1}
  AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID:-}
  AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY:-}
  KONG_CLUSTER_FALLBACK_CONFIG_STORAGE: s3://${AWS_S3_BUCKET_NAME}/kong

services:
  # ... omit ...
  kong-dp-exporter:
    # ... omit ...
    environment:
      # ... 1
      <<: *aws-env
      KONG_CLUSTER_FALLBACK_CONFIG_EXPORT: "on"
  kong-dp-importer:
    # ... omit ...
    ports:
      - 8000:8000
    environment:
      # ... 2
      <<: *aws-env
      KONG_CLUSTER_FALLBACK_CONFIG_IMPORT: "on"
```

上記の `kong-dp-exporter` サービスは Kong Gateway の設定情報を S3 へ保存するためのバックアップノードの役割を持ちます。`KONG_CLUSTER_FALLBACK_CONFIG_EXPORT` 環境変数を `on` に設定することで、Kong Gateway の設定情報が S3 へエクスポートされるようになります。その他、S3 バケットへ書き込むための AWS の認証情報が必要なため、`x-aws-env` エイリアスでまとめています。次に、`kong-dp-importer` サービスは、コンテナの起動時に CP と疎通が取れない場合に、S3 から設定情報をインポートするための設定(`KONG_CLUSTER_FALLBACK_CONFIG_IMPORT`) が含まれています。

ここで、実際の動きを確認するための以下のような手順で実験を行います。

1. `kong-dp-importer` サービス以外のコンテナを起動する
2. S3 バケットに設定情報が保存されていることを確認する
3. `kong-dp-importer` サービスを以下の条件で起動する
   - CP との疎通が取れない状態で起動する（`kong-cp` コンテナを停止しておく）
   - `KONG_CLUSTER_FALLBACK_CONFIG_IMPORT: "off"` の設定を有効化する
4. `kong-dp-importer` サービスを以下の条件で起動する
   - CP との疎通が取れない状態で起動する（`kong-cp` コンテナを停止しておく）
   - `KONG_CLUSTER_FALLBACK_CONFIG_IMPORT: "on"` の設定を有効化する

期待値としては、3 の手順では `kong-dp-importer` サービスは起動はするがルーティングに関する設定情報を持たないため、API トラフィックが処理できない状態となります。一方で、4 の手順では CP と疎通が取れない場合のフォールバックとして S3 から設定をインポートできるので、API トラフィックが処理できる状態となります。

尚、環境全体を起動時に decK を利用して以下のような設定が Kong Gateway に適用されます。

```yaml:kong.yaml
_format_version: "3.0"

services:
  - name: httpbin-service
    url: https://httpbin.org
    routes:
      - name: httpbin-route
        paths:
          - /mock
        strip_path: true
```

よって、正しくアクセスできた場合は以下のような結果が得られます。

```sh
curl -i http://localhost:8000/mock/status/200
```

実行結果：

```sh
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 0
Connection: keep-alive
Date: Mon, 08 Dec 2025 08:24:01 GMT
Server: gunicorn/19.9.0
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Kong-Upstream-Latency: 1795
X-Kong-Proxy-Latency: 2
Via: 1.1 kong/3.12.0.0-enterprise-edition
X-Kong-Request-Id: e79d5d280d8e6e2cfb22ae15b57f6e7d
```

まずは、実験手順通り `kong-dp-importer` サービス以外のコンテナを起動します。

```sh
docker compose up -d \
    database \
    kong-bootstrap \
    kong-cp \
    kong-dp-exporter \
    deck
```

起動後に、S3 バケットの中を確認すると、以下のように設定情報が保存されていることがわかります。

```sh
aws s3 ls s3://$AWS_S3_BUCKET_NAME/kong/3.12.0.0/
                           PRE election/
2025-12-08 17:39:04      16132 config.json
```

`config.json` の中身を確認すると、Kong Gateway の設定情報が保存されていることがわかります。

```json:config.json
{
    "_transform": false,
    "services": [
        {
            "write_timeout": 60000,
            "protocol": "https",
            "id": "c2dd77ef-12de-4010-9c95-f7a4f9d79aaf",
            "enabled": true,
            "ws_id": "ef7dde63-bb22-4212-a5d1-ce6fe910d15e",
            "path": null,
            "tls_verify": null,
            "retries": 5,
            "tls_verify_depth": null,
            "ca_certificates": null,
            "client_certificate": null,
            "updated_at": 1765189923,
            "connect_timeout": 60000,
            "port": 443,
            "read_timeout": 60000,
            "tags": null,
            "created_at": 1765189923,
            "tls_sans": null,
            "host": "httpbin.org",
            "name": "httpbin-service"
        }
    ],
    "routes": [
        {
            "sources": null,
            "destinations": null,
            "https_redirect_status_code": 426,
            "response_buffering": true,
            "created_at": 1765189923,
            "ws_id": "ef7dde63-bb22-4212-a5d1-ce6fe910d15e",
            "tags": null,
            "strip_path": true,
            "protocols": [
                "http",
                "https"
            ],
            "headers": null,
            "preserve_host": false,
            "updated_at": 1765189923,
            "id": "a8a484d6-7b18-4c4a-badb-61b58f379722",
            "name": "httpbin-route",
            "request_buffering": true,
            "snis": null,
            "service": "c2dd77ef-12de-4010-9c95-f7a4f9d79aaf",
            "regex_priority": 0,
            "path_handling": "v0",
            "methods": null,
            "paths": [
                "/mock"
            ],
            "hosts": null
        }
    ],
    "_format_version": "3.0",
    ... omitted ...
}
```

続いて、`kong-dp-importer` サービスを CP との疎通が取れない状態で起動します。このとき、`KONG_CLUSTER_FALLBACK_CONFIG_IMPORT` 環境変数を `off` に設定します。

```sh
# kong-cp コンテナを停止する
docker compose stop kong-cp
# KONG_CLUSTER_FALLBACK_CONFIG_IMPORT を off に設定して起動する
docker compose up -d kong-dp-importer
```

起動中のログを確認すると、以下のように CP との疎通が取れないことが確認できます。

```sh
# ... omit ...
kong-dp-importer  | 2025/12/08 09:30:25 [error] 2680#0: *2260 [lua] manager.lua:647: connect(): [rpc] unable to connect to peer: failed to connect: [cosocket] DNS resolution failed: dns server error: 3 name error. Tried: ["(short)kong-cp:(na) - cache-miss","kong-cp:33 - cache-hit/dns server error: 3 name error","kong-cp:1 - cache-hit/dns server error: 3 name error","kong-cp:5 - cache-hit/dns server error: 3 name error"], context: ngx.timer
kong-dp-importer  | 2025/12/08 09:30:26 [warn] 2686#0: *330 [lua] data_plane.lua:251: communicate(): [clustering] connection to control plane wss://kong-cp:8005/v1/outlet?node_id=7db998b1-a64f-488f-8e16-f30342d4c558&node_hostname=5f1f477506f7&node_version=3.12.0.0 broken: failed to connect: [cosocket] DNS resolution failed: dns server error: 3 name error. Tried: ["(short)kong-cp:(na) - cache-miss","kong-cp:33 - cache-hit/dns server error: 3 name error","kong-cp:1 - cache-hit/dns server error: 3 name error","kong-cp:5 - cache-hit/dns server error: 3 name error"] (retrying after 5 seconds) [kong-cp:8005], context: ngx.timer
```

この状態でリクエストを送ってみると、Route が存在しないため 404 エラーが返ってくることが確認できます。

```sh
curl -i http://localhost:8000/mock/status/200
```

実行結果：

```sh
HTTP/1.1 404 Not Found
Date: Mon, 08 Dec 2025 09:33:20 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 103
X-Kong-Response-Latency: 1
Server: kong/3.12.0.0-enterprise-edition
X-Kong-Request-Id: 173e17561ca26b01c9e4b40eefc18672

{
  "message":"no Route matched with those values",
  "request_id":"173e17561ca26b01c9e4b40eefc18672"
}
```

最後に、`KONG_CLUSTER_FALLBACK_CONFIG_IMPORT` 環境変数を `on` に設定して `kong-dp-importer` サービスを起動してみます。

```sh
docker compose up -d kong-dp-importer
```

この状態でリクエストを送ってみます。

```sh
curl -i http://localhost:8000/mock/status/200
```

実行結果：

```sh
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 0
Connection: keep-alive
Date: Mon, 08 Dec 2025 10:55:41 GMT
Server: gunicorn/19.9.0
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Kong-Upstream-Latency: 54075
X-Kong-Proxy-Latency: 60560
Via: 1.1 kong/3.12.0.0-enterprise-edition
X-Kong-Request-Id: 0e53fb39ed61f05d2a5b714bd0661a6d
```

CP への疎通が取れない状況でも S3 から設定情報をインポートすることで、期待通り API トラフィックが処理されることが確認できました。

## 終わりに

今回は、Kong Gateway の CP がダウンしている際に、DP を S3 に保存してあるバックアップを用いて復旧（もしくはスケール）させるシナリオを試してみました。記事内部でも記載しましたが、別途バックアップ用の Kong Gateway を用意する必要があるため、運用負荷が増加する点にはご注意ください。ただし、SLA を遵守する場合は有効な手段となるはずです。また、今回の構成を簡単に再現するためのリポジトリも共有します。

https://github.com/shukawam/kong-dp-dr
