---
title: "Azure Container Apps に Kong Gateway(Konnect) をデプロイしてみた"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "kong", "terraform"]
published: true
---

## はじめに

こんにちは 🖐️ 今日は、Azure Container Apps に Kong Gateway(Konnect)をAzure CLIを用いてデプロイしてみます。同じ構成で取り組まれる方の参考になれば幸いです。それではやっていきましょう 🦍

## Azure Container Apps

Kubernetes や Dapr, KEDA, Envoy などを利用して作られたサーバレスのコンテナ実行基盤です。ですが、Kubernetes API へのアクセスは禁止されているようで、ユーザーとしては裏側が Kubernetes であることを意識することなく利用することができます。また、KEDA を用いたイベントドリブンによる動的スケールや Dapr を用いたマイクロサービス開発などもサポートされているようです。

## Kong Konnect

Kong Gateway は本番環境で利用される場合、基本的には以下のような Control Plane と Data Plane が分離された Hybrid 構成をとります。

![kong-hybrid](/images/kong-konnect-with-ms-azure/kong-hybrid.png)

Kong Konnect は状態を持っている Control Plane の管理をプロバイダー側に任せることで管理工数を削減することを目的とした SaaS となります[^1]。

[^1]: より正確には、Data Plane もプロバイダーの管理下に置くことが可能で、[Dedicated Cloud Gateway](https://developer.konghq.com/dedicated-cloud-gateways/) と呼ばれています。

![konnect-overview](/images/kong-konnect-with-ms-azure/konnect-overview.png)

引用: [https://docs.jp.konghq.com/konnect/getting-started/](https://docs.jp.konghq.com/konnect/getting-started/)

## 手順

冒頭で述べた通り、Kong Gateway(Konnect)を Azure Container Apps にデプロイしてみます。CLI や Terraform Provider に比べて UI の方が変化し得ると思ったため、本記事では Azure CLI を用いた方法を紹介します。また、Azure CLI のインストールや初期設定などは本記事のスコープ外とさせていただきます。

### 事前準備

まずは、Konnect 側で Control Plane を作成します。Gateway Manager の画面から **+ New Gateway** をクリックします。

![prepare-step-1](/images/kong-konnect-with-ms-azure/prepare-step-1.png)

続いて、作成する Gateway の種別や各種情報を入力します。今回は、クラウド（Azure）環境に Kong Gateway(Data Plane)をデプロイし、自分で管理するため Self-Managed Hybrid を選択し、情報に関しては以下のように入力します。

| 入力項目    | 必須？ | 入力内容                                     |
| ----------- | ------ | -------------------------------------------- |
| Name        | Y      | azure-container-apps-gateway                 |
| Description | N      | Kong Gateway running on Azure Container Apps |
| Labels      | N      | env: azure                                   |

![prepare-step-2](/images/kong-konnect-with-ms-azure/prepare-step-2.png)

その次に、Gateway のバージョンや稼働させるプラットフォームを選択します。今回は、2025/07 現在の最新版である **3.10** を選択します。プラットフォームに関しては、自身の環境に合わせたものが最初に選択されていると思いますが、Azure Container Apps にデプロイするため、**Linux(Docker)**を選択してください。また、セットアップスクリプトが表示されると思いますが、後の手順で利用するのでタブなどを閉じずにこのままにしておいてください。

![prepare-step-3](/images/kong-konnect-with-ms-azure/prepare-step-3.png)

これで下準備は整ったので、実際にデプロイしていきましょう。

### Azure CLI

まずは、環境を作成します。

```sh
az containerapp env create \
    --name azure-container-apps-gateway \
    --resource-group <your-resource-group-name> \
    --location <your-location e.g. japaneast>
```

続いて、Azure Container Apps に Kong Gateway をデプロイするための構成を作成します。Konnect に表示されているセットアップスクリプトを参照しながら、以下の YAML ファイルを作成してください。

```yaml
# kong-gateway.yaml
properties:
  environmentId: /subscriptions/<your-subscription-id>/resourceGroups/<your-resource-group-name>/providers/Microsoft.App/managedEnvironments/azure-container-apps-gateway
  configuration:
    activeRevisionsMode: Single
    ingress:
      allowInsecure: true
      external: true
      targetPort: 8000
      traffic:
        - latestRevision: true
          weight: 100
      transport: Auto
  template:
    containers:
      - env:
          - name: KONG_ROLE
            value: "data_plane"
          - name: KONG_DATABASE
            value: "off"
          - name: KONG_VITALS
            value: "off"
          - name: KONG_CLUSTER_MTLS
            value: "pki"
          - name: KONG_CLUSTER_CONTROL_PLANE
            value: "d189f3cad2.us.cp0.konghq.com:443" # Konnectに表示されている値に差し替え
          - name: KONG_CLUSTER_SERVER_NAME
            value: "d189f3cad2.us.cp0.konghq.com" # Konnectに表示されている値に差し替え
          - name: KONG_CLUSTER_TELEMETRY_ENDPOINT
            value: "d189f3cad2.us.tp0.konghq.com:443" # Konnectに表示されている値に差し替え
          - name: KONG_CLUSTER_TELEMETRY_SERVER_NAME
            value: "d189f3cad2.us.tp0.konghq.com" # Konnectに表示されている値に差し替え
          - name: KONG_CLUSTER_CERT
            value: | # Konnectに表示されている値に差し替え
              -----BEGIN CERTIFICATE-----
              MIICWjCCAgCgAwIBAgIBATAKBggqhkjOPQQDBDBeMVwwCQYDVQQGEwJVUzBPBgNV
              BAMeSABrAG8AbgBuAGUAYwB0AC0AYQB6AHUAcgBlAC0AYwBvAG4AdABhAGkAbgBl
              AHIALQBhAHAAcABzAC0AZwBhAHQAZQB3AGEAeTAeFw0yNTA3MDgwNDIxMDhaFw0z
              NTA3MDgwNDIxMDhaMF4xXDAJBgNVBAYTAlVTME8GA1UEAx5IAGsAbwBuAG4AZQBj
              AHQALQBhAHoAdQByAGUALQBjAG8AbgB0AGEAaQBuAGUAcgAtAGEAcABwAHMALQBn
              AGEAdABlAHcAYQB5MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEgIgNg+oIr/MA
              UEmnfY03Gc8SuD84UOOcYka6t5cezgpNHX7YrFE5LbzYpINFJ5qmLJyaAe6KdDCJ
              mUvAiHmAw6OBrjCBqzAMBgNVHRMBAf8EAjAAMAsGA1UdDwQEAwIABjAdBgNVHSUE
              FjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwFwYJKwYBBAGCNxQCBAoMCGNlcnRUeXBl
              MCMGCSsGAQQBgjcVAgQWBBQBAQEBAQEBAQEBAQEBAQEBAQEBATAcBgkrBgEEAYI3
              FQcEDzANBgUpAQEBAQIBCgIBFDATBgkrBgEEAYI3FQEEBgIEABQACjAKBggqhkjO
              PQQDBANIADBFAiBih+XnxIK+A07ISNwdiuKu8xgwNJIxqfgss1ENtnb4EwIhAN3x
              2HL0w/8z6pDzsxVcjjLCd6sYgqZuIoSoHPYerrdD
              -----END CERTIFICATE-----
          - name: KONG_CLUSTER_CERT_KEY
            value: | # Konnectに表示されている値に差し替え
              -----BEGIN PRIVATE KEY-----
              MIGTAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBHkwdwIBAQQgwusqCdgRGb4DYtf4
              6Y0NvGk7s+cgFF2orTgZhoaWpO2gCgYIKoZIzj0DAQehRANCAASAiA2D6giv8wBQ
              Sad9jTcZzxK4PzhQ45xiRrq3lx7OCk0dftisUTktvNikg0UnmqYsnJoB7op0MImZ
              S8CIeYDD
              -----END PRIVATE KEY-----
          - name: KONG_LUA_SSL_TRUSTED_CERTIFICATE
            value: system
          - name: KONG_KONNECT_MODE
            value: on
          - name: KONG_CLUSTER_DP_LABELS
            value: type:docker-linuxdockerOS
          - name: KONG_ROUTER_FLAVOR
            value: expressions
        image: kong/kong-gateway:3.10
        name: kong-gateway
        resources:
          cpu: 0.5
          memory: 1Gi
```

デプロイします。

```sh
az containerapp create \
    --name kong-gateway \
    --resource-group <your-resource-group> \
    --yaml kong-gateway.yaml
```

無事デプロイが完了し、Control Plane との疎通確認が取れると、Konnect 側で以下のように表示されます。

![prepare-step-4](/images/kong-konnect-with-ms-azure/prepare-step-4.png)

最後に簡単なサービスを登録し、動作確認を行います。以下の構成ファイルを、Kong Gateway を設定するための CLI である decK を使って読み込みます。

```yaml
# kong.yaml
_format_version: "3.0"

services:
  - name: httpbin-service
    url: http://httpbin.org/anything
    routes:
      - name: http
        paths:
          - /mock
        strip_path: true
```

適用します。

```sh
deck gateway sync kong.yaml \
    --konnect-token <your-konnect-pat> \
    --konnect-control-plane-name azure-container-apps-gateway
```

リクエストを送信して動作確認します。

```sh
curl https://kong-gateway.gentleground-bfcffaab.japaneast.azurecontainerapps.io/mock
```

以下のような結果が返って来れば疎通確認は完了です。

```json
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "User-Agent": "curl/8.7.1",
    "X-Amzn-Trace-Id": "Root=1-686cb93b-1620bb3e6f7f6a286cbad150",
    "X-Arr-Ssl": "true",
    "X-Envoy-Expected-Rq-Timeout-Ms": "1800000",
    "X-Envoy-External-Address": "126.218.11.92",
    "X-Forwarded-Host": "kong-gateway.gentleground-bfcffaab.japaneast.azurecontainerapps.io",
    "X-Forwarded-Path": "/mock",
    "X-Forwarded-Prefix": "/mock",
    "X-K8Se-App-Kind": "web",
    "X-K8Se-App-Name": "kong-gateway--ccdquo8",
    "X-K8Se-App-Namespace": "k8se-apps",
    "X-K8Se-Protocol": "http1",
    "X-Kong-Request-Id": "83e5dc27b0adfc765927be062590165c",
    "X-Ms-Containerapp-Name": "kong-gateway",
    "X-Ms-Containerapp-Revision-Name": "kong-gateway--ccdquo8"
  },
  "json": null,
  "method": "GET",
  "origin": "126.218.11.92, 100.100.0.48, 48.218.158.81",
  "url": "http://kong-gateway.gentleground-bfcffaab.japaneast.azurecontainerapps.io/anything"
}
```

## 終わりに

Azure Container Apps に Kong Gateway(Konnect)をデプロイしてみました。Data Plane のみのデプロイのため、データベースなどは考慮する必要がなく比較的簡単にデプロイできました。需要があれば、Azure Container Apps + Azure Database for PostgreSQL で Control Plane も Data Plane も自分で管理する構成に関しても取り上げてみたいと思います。

ちなみに、本記事では Azure CLI を使いましたが Terraform を用いてより宣言的にデプロイしたい方はこちらが参考になると思います。

https://github.com/shukawam/kong-terraform/tree/main/azure
