---
title: "Prometheus/OpenTelemetryプラグインのメトリクス差分について"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: false
---

## はじめに

こんにちは 🦍 こちらの記事は、PrometheusプラグインとOpenTelemetryプラグインが出力するメトリクスの情報差分についてまとめたものです。Kong Gatewayでは、3.13以降ログ・メトリクス・トレースをすべてOpenTelemetryプラグインで扱えるようになり、オブザーバビリティの設定が随分とすっきりしました。しかし、従来より使われていたPrometheusプラグインと出力される内容に差分が存在するため、少し注意が必要です。それでは解説していきます！

## 差分のサマリー

:::message

明確にドキュメンテーションされているわけではなく、実機を使って検証した結果に基づくため、もしかしたら漏れているような差分が存在するかもしれません。

:::

検証した上でわかっている差分は以下の通りです。

- AI関連のメトリクスは、OpenTelemetryプラグインでは未対応（少なくとも3.13では）
- メトリクス名が変更されている
- レイテンシーヒストグラムについては単位がミリ秒から秒へ変更されている

それぞれ補足します。

### AI関連のメトリクスは、OpenTelemetryプラグインでは未対応

Prometheusプラグインでは、AI関連のメトリクスを取得するために以下のような設定を書きます。

```yaml
plugins:
  - name: prometheus
    config:
      ai_metrics: true
```

しかし、OpenTelemetryプラグインで使用可能なメトリクスの一覧を参照すると、AI関連のメトリクスは存在しないため、AI関連のメトリクスを活用したい場合は、引き続きPrometheusプラグインを使用する必要があります。

https://developer.konghq.com/plugins/opentelemetry/#metrics

### メトリクス名の変更

次に、メトリクス名の変更についてです。調査した限りでは以下のような差分が存在します。

| Metrics                         | Prometheus                                      | OpenTelemetry                         |
| ------------------------------- | ----------------------------------------------- | ------------------------------------- |
| HTTPリクエスト数                | `kong_http_requests_total`                      | `http_server_request_count_total`     |
| Kong内部のレイテンシ            | `kong_kong_latency_ms`                          | `kong_latency_internal_seconds`       |
| レイテンシ                      | `kong_request_latency_ms`                       | `kong_latency_total_seconds`          |
| アップストリームのレイテンシ    | `kong_upstream_latency_ms`                      | `kong_latency_upstream_seconds`       |
| 帯域幅（Ingress）               | `kong_bandwidth_bytes`                          | `http_server_request_size_bytes`      |
| 帯域幅（Egress）                | `kong_bandwidth_bytes`                          | `http_server_response_size_bytes`     |
| 接続ステータス(CP)              | `kong_control_plane_connected`                  | `kong_cp_connection_status_ratio`     |
| 接続ステータス(DP)              | `kong_datastore_reachable`                      | `kong_db_connection_status_ratio`     |
| 証明書の有効期限                | `kong_data_plane_cluster_cert_expiry_timestamp` | `kong_dp_cluster_cert_expiry_seconds` |
| メモリ使用量 - Shared Dictional | `kong_memory_lua_shared_dict_bytes`             | `kong_shared_dict_usage_bytes`        |
| メモリ総容量 - Shared Dictional | `kong_memory_lua_shared_dict_total_bytes`       | `kong_shared_dict_size_bytes`         |
| Nginx接続メトリクス             | `kong_nginx_connections_total`                  | `kong_nginx_connection_count`         |
| Nginxタイマー                   | `kong_nginx_timers`                             | `kong_nginx_timer_count`              |
| ノード/ターゲット情報           | `kong_node_info`                                | `target_info`                         |

### レイテンシーヒストグラムの単位

Prometheusプラグインでは、レイテンシヒストグラムが`kong_request_latency_ms`で提供されています。OpenTelemetryプラグインの項目を見ると、`kong_latency_total_seconds`のようになっており、単位がミリ秒から秒へ変更されているので、今までPrometheusプラグインを使っていた方は若干注意が必要です。ただし、バケット自体は以下に示す通りほぼ同等のものが提供されているため、ヒストグラム自体はほぼ同じものを得ることができます。

- Prometheus
  - 内部レイテンシ: `1, 2, 5, 7, 10, 15, 20, 30, 50, 75, 100, 200, 500, 750, 1000, 3000, 6000, +Inf`
  - アップストリームのレイテンシ: `700, 1000, 2000, 5000, 10000, 30000, 60000, +Inf`
  - 総レイテンシ: `1000, 2000, 5000, 10000, 30000, 60000, +Inf`
- OpenTelemetry
  - 内部レイテンシ: `0.001, 0.002, 0.005, 0.007, 0.01, 0.015, 0.02, 0.03, 0.05, 0.075, 0.1, 0.2, 0.5, 0.75, 1, 3, 6, +Inf`
  - アップストリームのレイテンシ: `0.025, 0.05, 0.08, 0.1, 0.25, 0.4, 0.7, 1, 2, 5, 10, 30, 60, +Inf`
  - 総レイテンシ: `0.025, 0.05, 0.08, 0.1, 0.25, 0.4, 0.7, 1, 2, 5, 10, 30, 60, +Inf`

## まとめ

今回は、PrometheusプラグインとOpenTelemetryプラグインから得られる情報差分をまとめてみました。プラグインの移行を検討される方や最新版を使う際にどちらを使うべきか悩んでいる方はぜひ参考にしていただければ幸いです。
