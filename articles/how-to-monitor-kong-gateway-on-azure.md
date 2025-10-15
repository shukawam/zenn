---
title: "Azure Monitor を利用した Kong Gateway の監視方法"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong", "azure"]
published: true
---

## はじめに

以前、Azure Container App(以下、ACA)で Kong Gateway をデプロイしてみました。

https://zenn.dev/shukawam/articles/kong-konnect-with-ms-azure

今回は、もう少しまともに使える環境を！ということで、Azure Monitor を使ってこの Kong Gateway を監視するための設定を仕込みたいと思います。

## Azure Monitor について

Azure Monitor は、Azure が提供するオブザーバビリティ関連のコンポーネントの集合です。公式ドキュメントに記載されているこちらの図が個人的にわかりやすかったです。

![azure-monitor-overview](https://learn.microsoft.com/ja-jp/azure/azure-monitor/fundamentals/media/overview/overview-simple-20230707-opt.svg)

https://learn.microsoft.com/ja-jp/azure/azure-monitor/fundamentals/overview

## 作りたい構成

前回同様、Kong Gateway は ACA で動作させます。その上で Azure Monitor と連携するために以下のような構成を採用します。

![architecture](/images/how-to-monitor-kong-gateway/architecture.png)

いくつか別の選択肢もあったので合わせて紹介します。

まず、OpenTelemetry Collector を Sidecar コンテナでデプロイしていますが、ACA には OpenTelemetry エージェントが存在しそれを活用することで自分で OpenTelemetry Collector をデプロイする必要がなくなります。

https://learn.microsoft.com/ja-jp/azure/container-apps/opentelemetry-agents?tabs=arm%2Carm-example

ですが、上記ドキュメントに記載の通り、Azure App Insights へテレメトリー・シグナルを送信する際はメトリクスがサポートされていません。Kong では、Kong Gateway 自身のメトリクスに加えて API レベルのメトリクスを一元的に収集し、Prometheus 形式で公開する機能が存在しますが、OpenTelemetry エージェントを使う構成では、これを活用できないのでこの選択肢はなくなりました。また、テレメトリー・パイプラインの細かな制御（任意の属性を付与したり、サンプリングの制御）も ACA の OpenTelemetry エージェントでは実現ができなさそうだったのも不採用の理由の一つです。

次に、ログの扱いです。Kong Gateway の File Log プラグインを用いて任意の場所にファイルとしてログ出力を行い、Filelog レシーバーを使うことで OpenTelemetry Collector に全てのテレメトリー・シグナルを統一することもできたのですが、OpenTelemetry Collector が落ちてしまった際にログのローテーションができなくなり、EmptyDir のサイズ上限である 50GB に抵触する可能性があったため、標準出力に出力したログを ACA の機能で Log Analytics に転送することにしました。上記の懸念事項は OpenTelemetry Collector 自体を監視し、適切な対処（再起動など）をすることで未然に防ぐことができますが、連携の手軽さの観点も相まってこのような選択をとっています。

## Terraform での構成例

https://github.com/shukawam/kong-terraform/tree/main/azure

こちらのリポジトリに含まれている構成で `terraform apply` してもらうと上記の環境を”**ほぼ**”作成することができます。ただし、ACA と Log Analytics の連携は、以下のように記述しても適切にログ連携されなかったので、別途 UI から ACA の設定画面で新規に Log Analytics のワークスペースを作成することで解決しました。（解決方法わかる方いれば教えていただけると助かります 🙏）

```tf:container_apps.tf
resource "azurerm_container_app_environment" "shukawam_container_app_environment" {
  name                       = "shukawam-container-app-environment"
  location                   = var.location
  resource_group_name        = azurerm_resource_group.shukawam_resource_group.name
  logs_destination           = "log-analytics"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.shukawam_log_analytics_workspace.id
}
```

以降で実現するためのポイントを解説します。

```tf:container_apps.tf
resource "azurerm_container_app" "shukawam-kong-gateway" {
  name                         = "shukawam-kong-gateway"
  container_app_environment_id = azurerm_container_app_environment.shukawam_container_app_environment.id
  resource_group_name          = azurerm_resource_group.shukawam_resource_group.name
  revision_mode                = "Single"
  template {
    init_container {
      name   = "load-otel-collector-config"
      image  = "busybox:latest"
      cpu    = "0.25"
      memory = "0.5Gi"
      command = [
        "wget",
        "https://raw.githubusercontent.com/shukawam/kong-terraform/refs/heads/main/azure/config/otel-collector-config.yaml",
        "-O",
        "/tmp/config.yaml"
      ]
      volume_mounts {
        name = "otel-config"
        path = "/tmp"
      }
    }
    container {
      name   = "opentelemetry-collector"
      image  = "otel/opentelemetry-collector-contrib:0.137.0"
      cpu    = "0.25"
      memory = "0.5Gi"
      args   = ["--config=/etc/otelcol-contrib/config.yaml"]
      env {
        name  = "CONNECTION_STRING"
        value = azurerm_application_insights.shukawam_application_insights.connection_string
      }
      volume_mounts {
        name = "otel-config"
        path = "/etc/otelcol-contrib"
      }
    }
    container {
      name   = "kong-gateway"
      image  = "kong/kong-gateway:3.12"
      cpu    = "0.5"
      memory = "1Gi"
      # ... 省略 ...
      env {
        name  = "KONG_CLUSTER_CERT"
        value = var.kong_cluster_cert
      }
      env {
        name  = "KONG_CLUSTER_CERT_KEY"
        value = var.kong_cluster_cert_key
      }
      env {
        name  = "KONG_LUA_SSL_TRUSTED_CERTIFICATE"
        value = "system"
      }
      env {
        name  = "KONG_KONNECT_MODE"
        value = "on"
      }
      env {
        name  = "KONG_CLUSTER_DP_LABELS"
        value = "type:docker-linuxdockerOS"
      }
      env {
        name  = "KONG_ROUTER_FLAVOR"
        value = "expressions"
      }
      env {
        name  = "KONG_TRACING_INSTRUMENTATIONS"
        value = "all"
      }
      env {
        name  = "KONG_TRACING_SAMPLING_RATE"
        value = "1.0"
      }
      env {
        name  = "KONG_STATUS_LISTEN"
        value = "0.0.0.0:8100"
      }
    }

    # Always keep 1 replica to avoid cold start
    min_replicas = 1
    max_replicas = 1
    volume {
      name         = "otel-config"
      storage_type = "EmptyDir"
    }
  }

  ingress {
    allow_insecure_connections = true
    external_enabled           = true
    target_port                = "8000"
    transport                  = "auto"
    traffic_weight {
      latest_revision = true
      percentage      = 100
    }
  }
}
```

まず、OpenTelemetry Collector を Sidecar で起動するために設定ファイルを渡してあげる必要があります。設定ファイルを含める形で OpenTelemetry Collector のイメージを再ビルドするか外部から設定ファイルをマウントする選択肢がありますが、今回は手軽さを優先して initContainers で設定ファイルを外部から取得し、EmptyDir 経由で OpenTelemetry Collector のコンテナにマウントすることにしました。また、OpenTelemetry Collector は、contrib を使用していますが、イメージ自体を軽量にするのとアタックサーフェースを減らすという意味で OCB(OpenTelemetry Collector Builder)を使い、必要なコンポーネントのみを含む形で稼働させるのが望ましいでしょう。加えて、トレース・メトリクスを取得するための設定を Kong Gateway 自体に含んでいるのもポイントです。

```tf:container_apps.tf
env {
    name  = "KONG_TRACING_INSTRUMENTATIONS"
    value = "all"
}
env {
    name  = "KONG_TRACING_SAMPLING_RATE"
    value = "1.0"
}
env {
    name  = "KONG_STATUS_LISTEN"
    value = "0.0.0.0:8100"
}
```

また、アクセスログは File Log プラグインを用いて標準出力に出力することで ACA の機能を使って Log Analytics へ転送することが可能です。

```yaml:kong.yaml
plugins:
  - name: file-log
    config:
      path: /dev/stdout
      custom_fields_by_lua:
        traceid: |
          local h = kong.request.get_header('traceparent')
          return h:match("%-([a-f0-9]+)%-[a-f0-9]+%-")
        spanid: |
          local h = kong.request.get_header('traceparent')
          return h:match("%-[a-f0-9]+%-([a-f0-9]+)%-")
```

続いて、OpenTelemetry Collector の設定は以下のようにしています。特に目立ったポイントはないですが、ACA に構築した Kong Gateway が 8100 で Status API をリッスンしているのでそれをスクレイプするように設定しています。また、[Azure Monitor の exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/azuremonitorexporter) が存在するので、これを使うことで適切なテレメトリー・シグナルを送信することが可能です。

```yaml:otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: kong-dp
          scrape_interval: 10s
          static_configs:
            - targets:
                - localhost:8100

processors:
  batch:

exporters:
  azuremonitor:
    connection_string: ${env:CONNECTION_STRING}
  debug:
    verbosity: detailed
    sampling_initial: 5
    sampling_thereafter: 200

service:
  pipelines:
    traces:
      receivers:
        - otlp
      processors:
        - batch
      exporters:
        - azuremonitor
    metrics:
      receivers:
        - prometheus
      processors:
        - batch
      exporters:
        - azuremonitor
```

## おわりに

今回は、ACA で稼働している Kong Gateway を Azure のオブザーバビリティ関連サービスを使って監視するための構成を検討してみました。他にも色々な選択肢があると思いますが、一例として参考になれば幸いです。

## 参考

https://learn.microsoft.com/ja-jp/azure/container-apps/log-monitoring?tabs=bash

https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/azuremonitorexporter
