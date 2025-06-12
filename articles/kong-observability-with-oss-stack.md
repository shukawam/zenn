---
title: "Kong Gateway の監視基盤を OSS(Prometheus/Loki/Tempo + Grafana) で整備する"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong", "kubernetes", "observability"]
published: false
---

## はじめに

こんにちは 🖐️ この記事では、Kong Gateway をより安全に運用する上で必要になる監視基盤をいくつかの OSS(Prometheus/Grafana Loki/Grafana Tempo + Grafana)を使って整備します。具体的には以下のような構成を Kubernetes 上で作成することがゴールとなっています。

![architecture](/images/kong-observability-with-oss-stack/architecture.png)

## 使用するプラグインについて

Kong Gateway は、コア機能に加えて数多くのプラグインを提供しています[^1]。今回の関心ごとである監視関連プラグインは以下にまとまっていますので、興味があれば参照してみてください。今回はこのうち、[OpenTelemetry](https://docs.konghq.com/hub/kong-inc/opentelemetry/) と [Prometheus](https://docs.konghq.com/hub/kong-inc/prometheus/) を利用するので、以降で簡単に解説します。

https://docs.konghq.com/hub/?category=analytics-monitoring%2Clogging

[^1]: 利用可能なプラグインは、Kong Plugin Hub を参照ください: [https://docs.konghq.com/hub/](https://docs.konghq.com/hub/)

### OpenTelemetry plugin

OpenTelemetry plugin は、OTLP[^2]互換のバックエンドに対してテレメトリー・データを出力するためのプラグインです。ざっとドキュメントをみると以下のテレメトリー・データに対応しているようです。

#### トレース

Kong Gateway 自体は、OpenTelemetry で計装されている[^3]ため、Gateway の呼び出し前後を含んでトレースを伝搬させたり、Gateway 内部のパフォーマンス分析をより詳細にするために用います。

#### ログ

リクエストスコープ（アクセスログなど）のログと非リクエストスコープ（起動ログなど）を OTLP 互換のバックエンドに出力することができます。

#### メトリクス

Kong Gateway 自体のメトリクスというよりは、取得したスパンからメトリクスが生成可能な [Span Metrics Connector](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/spanmetricsconnector) を使ってメトリクスを生成するみたいです。Kong Gateway 自体のメトリクスは、後述する Prometheus plugin を使って取得すると良いでしょう。

[^2]: OpenTelemetry Protocol
[^3]: 興味がある方は、この辺りを読んでみるとどんな感じに計装されているか雰囲気が掴めます: [https://github.com/Kong/kong/tree/master/kong/observability](https://github.com/Kong/kong/tree/master/kong/observability)

### Prometheus plugin

Prometheus plugin は、Kong Gateway やプロキシしている上流サービスに関連するメトリクスを Prometheus フォーマットで公開するためのプラグインです。[Kong 公式の Grafana ダッシュボード](https://grafana.com/grafana/dashboards/7424-kong-official/)も存在するのである程度の可視化であれば、すぐに行うことができます。

## 環境を作っていく

再掲ですが、以下の環境を Kubernetes(EKS)上に作成していきます。

![architecture](/images/kong-observability-with-oss-stack/architecture.png)

特に Prometheus は構築にあたってさまざまな選択肢がありますが、今回は以下を使いました。

- Prometheus + Grafana: [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- Grafana Loki: [loki-stack](https://github.com/grafana/helm-charts/tree/main/charts/loki-stack)
- Grafana Tempo: [tempo](https://github.com/grafana/helm-charts/tree/main/charts/tempo)
- OpenTelemetry Collector: [OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator)
- Kong Gateway: [Kong Ingress Controller(KIC)](https://github.com/Kong/kubernetes-ingress-controller)

### 前準備：Helm リポジトリの追加と Namespace の作成

この後の手順で利用する Helm リポジトリを追加しておきます。

```sh
# for kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# for Grafana Loki/Tempo
helm repo add grafana https://grafana.github.io/helm-charts
# for OpenTelemetry Operator
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
# for Kong
helm repo add kong https://charts.konghq.com

# update
helm repo update
```

監視基盤と Kong Gateway をデプロイするための Namespace を作っておきます。

```sh
kubectl create namespace observability
kubectl create namespace kong
```

### Prometheus + Grafana

```yaml
# values.yaml
prometheus:
  prometheusSpec:
    scrapeInterval: 10s
    evaluationInterval: 30s
grafana:
  persistence:
    enabled: true # enable persistence using Persistent Volumes
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: "default" # Configure a dashboard provider file to
          orgId: 1 # put Kong dashboard into.
          folder: ""
          type: file
          disableDeletion: false
          editable: true
    options:
      path: /var/lib/grafana/dashboards/default
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://kube-prometheus-stack-prometheus:9090
          isDefault: true
        - name: Loki
          type: loki
          url: http://loki-stack:3100
        - name: Tempo
          type: tempo
          url: http://tempo:3100
  dashboards:
    default:
      kong-dashboard:
        gnetId: 7424 # Install the following Grafana dashboard in the
        revision: 11 # instance: https://grafana.com/dashboards/7424
        datasource: Prometheus
      kic-dashboard:
        gnetId: 15662
        datasource: Prometheus
```

この後、構築する Grafana Loki/Tempo を Datasource としてあらかじめ登録したり、Kong 社提供のダッシュボードを含めておくことがポイントです。以下を実行して、クラスタ上にデプロイしましょう。

```sh
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
    -n observability \
    --values values.yaml
```

### Grafana Loki

```yaml
loki:
  image:
    # see: https://github.com/grafana/loki/issues/11557#issuecomment-2014825845
    tag: 2.9.3
```

コメントの参照先に書いてある通り、`kube-prometheus-stack-73.1.0`, `loki-stack-2.10.2` でも、Grafana の Datasource として Loki が参照できない Issue が再現したので、回避策として Loki のバージョンを Issue 内に記載があった `2.9.3` まで上げています。同様に、以下を実行して、クラスタ上にデプロイしましょう。

```sh
helm install loki-stack grafana/loki-stack \
    -n observability \
    --values values.yaml
```

### Grafana Tempo

Tempo は、特に何も変更を加えていないのでそのままデプロイしてください。

```sh
helm install tempo grafana/tempo \
    -n observability \
    --values values.yaml
```

### OpenTelemetry Collector

```yaml
manager:
  collectorImage:
    repository: otel/opentelemetry-collector-k8s
admissionWebhooks:
  certManager:
    enabled: false
  autoGenerateCert:
    enabled: true
```

コレクターのイメージだけ指定するのを忘れずに。あとは、自身の環境に合わせて設定してください。

```sh
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
    -n observability
```

続いて、インストールしたオペレーターを用いて、各テレメトリー・データを収集、処理、送信するための OpenTelemetry コレクターを作成します。

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: opentelemetry
spec:
  mode: daemonset
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15
      batch:
        send_batch_size: 10000
        timeout: 10s
    exporters:
      otlp/tempo:
        endpoint: tempo:4317
        tls:
          insecure: true
      otlphttp/loki:
        endpoint: http://loki-stack:3100/otlp
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp/tempo]
        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlphttp/loki]
```

Kubernetes クラスタ上にデプロイします。

```sh
kubectl apply -f opentelemetry-collector.yaml
```

### Kong Gateway

続いて、Kong Gateway のデプロイと今までに作成した監視基盤と連携するための設定をしていきます。

https://zenn.dev/shukawam/articles/kong-ee-with-eks

では、Hybrid mode で構築しましたが、今回は Kong Ingress Controller(KIC)を用いて DB-less モードで構築してみます。まずは、Gateway API の CRD をデプロイします。

```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

続いて、`GatewayClass`, `Gateway` をデプロイします。

```yaml
kind: GatewayClass
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: kong
  annotations:
    konghq.com/gatewayclass-unmanaged: "true"
spec:
  controllerName: konghq.com/kic-gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kong
spec:
  gatewayClassName: kong
  listeners:
    - name: proxy
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
```

提供されている Helm Chart を用いて、KIC をインストールします。

```yaml
gateway:
  serviceMonitor:
    enabled: true
    labels:
      release: kube-prometheus-stack
  env:
    tracing_instrumentations: all
    tracing_sampling_rate: 1.0
```

ここでは、デフォルトの設定に加えて、Prometheus からメトリクスエンドポイントを叩くために `ServiceMonitor` の作成と OpenTelemetry プラグインを用いてトレースを取得するための設定を加えています。これを以下のコマンドでデプロイします。

```sh
helm install kong kong/ingress \
    --values values.yaml \
    -n kong
```

:::message
今回の内容は、全て OSS の Kong Gateway でも実施できる内容ですが Enterprise Edition を使いたい場合は、以下ドキュメントに従って `KongLicense` を作成するか、環境変数(`gateway.env`)に `LICENSE_DATA` を含めてください。

https://docs.konghq.com/kubernetes-ingress-controller/latest/license/
:::

次に、OpenTelemetry, Prometheus プラグインをグローバルレベルで有効化します。

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongClusterPlugin
metadata:
  name: opentelemetry-plugin
  annotations:
    kubernetes.io/ingress.class: kong
  labels:
    global: "true"
plugin: opentelemetry
config:
  resource_arrtibutes:
    service.name: kong
  # gRPC endpoint is not supported by the OpenTelemetry plugin, so we use HTTP endpoints
  traces_endpoint: http://opentelemetry-collector.observability.svc.cluster.local:4318/v1/traces
  logs_endpoint: http://opentelemetry-collector.observability.svc.cluster.local:4318/v1/logs
---
apiVersion: configuration.konghq.com/v1
kind: KongClusterPlugin
metadata:
  name: prometheus-plugin
  annotations:
    kubernetes.io/ingress.class: kong
  labels:
    global: "true"
plugin: prometheus
config:
  status_code_metrics: true
  bandwidth_metrics: true
  upstream_health_metrics: true
  latency_metrics: true
  per_consumer: false
```

これをデプロイします。

```sh
kubectl apply -f kong-plugins.yaml
```

### サンプルアプリケーションのデプロイ

ここまでできたら、各テレメトリー・データを確認するためにサンプルアプリケーションをデプロイしましょう。今回は、KIC のドキュメントに掲載されていたサンプルアプリケーションを用います。

```sh
kubectl apply -f https://docs.konghq.com/assets/kubernetes-ingress-controller/examples/echo-service.yaml -n kong
```

続いて、Gateway API を用いて Kong Gateway の設定を行います。

```yaml
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: echo
  annotations:
    konghq.com/strip-path: "true"
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: kong
      namespace: kong
      sectionName: proxy
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /echo
      backendRefs:
        - name: echo
          port: 1027
```

これをデプロイします。

```sh
kubectl apply -f echo-route.yaml -n kong
```

何回かエンドポイントを叩き、テレメトリー・データを発生させてみましょう。

```sh
for ((i=0; i<20; i++))
do
    # PROXY_IPは、GatewayのADDRESSを確認し、設定してください
    curl -i $PROXY_IP/echo
done
```

## 監視基盤の確認

### メトリクス

Grafana のダッシュボードに Kong (Official)というものがあるため、それを確認してみましょう。

![example-of-metrics](/images/kong-observability-with-oss-stack/example-of-metrics.png)

上図では、RPS が一例として掲載されていますが、それ以外にもレイテンシーや帯域、メモリ使用率、Upstream のステータス、Nginx のいくつかのメトリクスが可視化されています。これでも運用上必要なメトリクスが不足している場合は、以下ドキュメントに出力されるメトリクスの一覧が記載されているので、ご活用ください。

https://docs.konghq.com/kubernetes-ingress-controller/latest/production/observability/prometheus/

### トレース

トレースは、Explore から Datasource を Tempo にすることで確認可能です。今回のサンプルアプリケーションは、OpenTelemetry で計装されているわけではないため、確認できるのは Gateway 内部の処理でどの工程でどれくらいの時間がかかっているのか？ということだけですが...。

![example-of-trace](/images/kong-observability-with-oss-stack/example-of-trace.png)

### ログ

ログは、Explore から Datasource を Loki にすることで確認可能です。

![example-of-log](/images/kong-observability-with-oss-stack/example-of-log.png)

## おわりに

今回は、Kong Gateway の監視をするための OSS 基盤の構築とそこにテレメトリー・データを送信するための設定を OpenTelemetry, Prometheus プラグインを使って行ってみました。Kong ではこれ以外にも Datadog や Dynatrace など商用オブザーバビリティ・サービスと連携するためのプラグインも提供しているので、機会があれば試してみたいと思います。
