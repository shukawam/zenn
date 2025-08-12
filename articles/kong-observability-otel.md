---
title: "Kong Gateway のオブザーバビリティを OpenTelemetry で高める"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong", "opentelemetry"]
published: true
---

## はじめに

こんにちは 🖐️ 以前こんな記事を書きました。

https://zenn.dev/shukawam/articles/kong-observability-with-oss-stack

ざっくりまとめると、Kubernetes 上で稼働している Kong Gateway のオブザーバビリティをメジャーな OSS を使って高めるという内容です。今回は、よりオブザーバビリティ標準（OpenTelemetry）に即した形で実践していきたいと思います！

## できあがる構成

![architecture](/images/kong-observability-otel/architecture.png)

上記のような構成が最終的にできあがります。使っているプラグインの意図を解説します。

### Prometheus Plugin

[Prometheus Plugin](https://developer.konghq.com/plugins/prometheus/) は、Kong Gateway とプロキシしているアップストリームのサービスに対するメトリクスを Prometheus 形式で公開するためのプラグインです。このプラグインを利用しなくても Admin API もしくは Status API からある程度のメトリクスは取得できるのですが、有効化することで以下のようなリクエスト関係のメトリクスを追加で取得することができます。

- Status code metrics
- Latency metrics
- Bandwidth metrics
- Upstream health metrics
- AI LLM metrics

また、設定項目に `per_consumer` という項目が存在する[^1]のですが、こちらを有効にすることで `http_requests_total`, `stream_sessions_total` のメトリクスをコンシューマ単位で取得することが可能になります。極端にコンシューマーが多い場合は、いわゆるカーディナリティエクスプロージョンが発生する可能性がある可能性があるため、有効化する場合は事前に机上での検討および実機検証することを推奨します。

[^1]: https://developer.konghq.com/plugins/prometheus/reference/#schema--config-per-consumer

### File Log Plugin

[File Log Plugin](https://developer.konghq.com/plugins/file-log/) は、Kong Gateway に対するリクエスト、レスポンスのログを JSON 形式でファイル出力するためのプラグインです。Kubernetes 環境の場合は、コンテナの標準出力に出力されたログはコンテナランタイムによってホスト側の所定の場所（`/var/log/pods/*/*.log` など）に出力され、それをログコレクター[^2]で収集し、適切なバックエンドに送信することが一般的かと思いますが、本プラグインで出力先を `/dev/stdout` や `/dev/stderr` に指定することでその営みを活用することが可能です。

[^2]: Promtail, Vector, Fluent Bit など

ただし、ドキュメントでも言及されている通り、File Log Plugin は I/O 処理が同期で行われるためホストのディスクが低速な場合は、パフォーマンスに関する問題が発生する可能性があります。また、Linux カーネルのパイプバッファサイズ（`PIPE_BUF`）の上限（4KB）を超えたログを一度に標準出力に書き込むとログのインターリービング[^3]が発生する可能性があるため、自身の書き込むログのサイズ（ヘッダーやボディサイズに多くは依存）と相談しながら検証すると良いでしょう。

[^3]: 複数のログが混じってしまうこと

### OpenTelemetry Plugin

[OpenTelemetry Plugin](https://developer.konghq.com/plugins/opentelemetry/) は、Kong Gateway に関わるトレースと Kong Gateway のログ（Control/Data Plane の起動ログなど）を出力するためのプラグインです。Kong Gateway 自体は、すでに OTLP 準拠の形でトレースデータを取得・出力できるような実装が組み込まれているので、設定（`kong.conf`）を有効化することでこれが可能になります。自動的に挿入されるスパンとしては以下の種類が存在します。

| スパン                 | 概要                                                       |
| ---------------------- | ---------------------------------------------------------- |
| `request`              | リクエストレベルの計装を有効化する                         |
| `db_query`             | データベースクエリの計装を有効化する                       |
| `dns_query`            | DNS クエリの計装を有効化する                               |
| `router`               | ルーター実行の計装を有効化する                             |
| `http_client`          | OpenResty HTTP クライアントの計装を有効化する              |
| `balancer`             | バランサーの計装を有効化する                               |
| `plugin_rewrite`       | 書き換えフェーズのプラグイン実行の計装を有効化する         |
| `plugin_access`        | アクセスフェーズのプラグイン実行の計装を有効化する         |
| `plugin_header_filter` | `header_filter` フェーズでプラグイン実行の計装を有効化する |
| `all`                  | すべての計装を有効化する                                   |
| `off`                  | すべての計装を無効化する（デフォルト）                     |

個人的には、すべての計装を有効(`all`)にし不要な部分があれば削っていくみたいな方針で良いんじゃないかな？と思います。また、プラグイン側の設定でシグナル送信に関するいくつかの設定（バッチやキューなど）が設定できますが、ここではあまり複雑なことは行わずに OpenTelemetry Collector 側に移譲する方が設計としてはシンプルになると思います。

## 実際の設定例や注意点について

### decK

Kubernetes 環境はすでに存在していること、Kong Gateway は Hybrid Mode(Konnect でも可)でクラスタに構築されていること、OpenTelemetry Collector などは適切に構築されていることを前提とします。また、設定は decK によって宣言的に Kong Gateway に適用します。

まずは、Prometheus Plugin は以下のように設定します。

```yaml
_format_version: "3.0"

plugins:
  - name: prometheus
    config:
      per_consumer: true
      status_code_metrics: true
      ai_metrics: true
      latency_metrics: true
      bandwidth_metrics: true
      upstream_health_metrics: true
      wasm_metrics: false
```

`wasm_metrics` は、自分の利用用途だと使わないので無効化しています。（というか、コードベースを見ても Wasm 関係のメトリクスを収集している様子もなさそうでしたのでどこで使われているのやら...）

次に、File Log Plugin です。

```yaml
_format_version: "3.0"

plugins:
  - name: file-log
    config:
      path: /dev/stdout
```

アクセスログは、標準出力（`/dev/stdout`）に出力しておくことで、コンテナランタイムがホスト上の所定のディレクトリにログを出力してくれます。一応、Fluent Bit のような軽量なログコレクターをサイドカーのような形でデプロイして、アクセスログを収集する方法もあります（下図参照）が、同一クラスタに OpenTelemetry Collector をデプロイする場合は、File Log Plugin を用いるのが最も簡単に実現できるでしょう。

![http-log-plugin](/images/kong-observability-otel/http-log-plugin.png)

最後に、OpenTelemetry Plugin です。

```yaml
_format_version: "3.0"

plugins:
  - name: opentelemetry
    config:
      # 環境に合わせて設定してください
      traces_endpoint: http://opentelemetry-collector.observability.svc.cluster.local:4318/v1/traces
      logs_endpoint: http://opentelemetry-collector.observability.svc.cluster.local:4318/v1/logs
      resource_attributes:
        # デフォルトは、"kong"がサービス名として利用されるので書き換えたい場合は設定する
        service.name: kong-gateway
```

基本は上記で十分だと思います。他にもシグナルをバッチ送信するためのパラメータやキューに関するパラメータ、サンプリングに関するパラメータが存在しますが OpenTelemetry Collector 側に制御を移譲した方が全体としてシンプルな作りになるはずです。

### メトリクスについて

なるべく OpenTelemetry に集約するって言ってるじゃん！嘘つき！と言われそうですが、メトリクスだけは Prometheus から直接 pull した方が素直に実現できます。[OpenTelemetry Collector Kubernetes Distro](https://github.com/open-telemetry/opentelemetry-collector-releases/tree/main/distributions/otelcol-k8s) にも Prometheus Receiver が含まれている[^4]ので、これを用いれば良さそうに見えますが、Kong Gateway でメトリクスを取得する上での推奨である Status API(読み取り専用のステータス情報などを提供する API)は、サービスなどに公開するように作られていません[^5]。

[^4]: https://github.com/open-telemetry/opentelemetry-collector-releases/blob/main/distributions/otelcol-k8s/manifest.yaml#L70
[^5]: https://github.com/Kong/charts/blob/main/charts/kong/values.yaml#L215-L223

じゃあ、どうするか？というと代わりに ServiceMonitor リソースを作成します。例えば、kube-prometheus-stack を利用している場合は、Prometheus Operator から作成した ServiceMonitor を正しく認識させるために以下のようなリリースラベルをつけて作成します。

```yaml
serviceMonitor:
  # Specifies whether ServiceMonitor for Prometheus operator should be created
  # If you wish to gather metrics from a Kong instance with the proxy disabled (such as a hybrid control plane), see:
  # https://github.com/Kong/charts/blob/main/charts/kong/README.md#prometheus-operator-integration
  enabled: true
  trustCRDsExist: false
  interval: 30s
  # Specifies namespace, where ServiceMonitor should be installed
  namespace: observability
  labels:
    release: kube-prometheus-stack
```

ちなみに、リリースラベルについてはクラスタに存在する Prometheus リソースの `serviceMonitorSelector` を確認して、一致するように記述すれば良いです。

```sh
kubectl -n observability get prometheus -o yaml \
| yq .items[].spec.serviceMonitorSelector
```

実行結果

```sh
matchLabels:
  release: kube-prometheus-stack
```

## 終わりに

今回は、Kong Gateway の監視をなるべく OpenTelemetry に寄せる形で実現してみました。OpenTelemetry Collector の設定などは要件に依存するところなので特に触れていませんが、設定例などは下記リポジトリをご参照ください。

https://github.com/shukawam/kong-cluster
