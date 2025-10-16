---
title: "KNEP(Kong Native Event Proxy)を試してみた"
emoji: "✍️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong", "kafka"]
published: false
---

## はじめに

こんにちは 🖐️ 今回は、ベータ版の機能である Kong Native Event Proxy

## KNEP(Kong Native Event Proxy)

## 試してみる

以下のような構成を作ってみます。Kafka Client と HTTP クライアント（API 呼び出し）を用いて Apache Kafka と Pub/Sub するような構成です。

![architecture](/images/get-started-with-knep/architecture.png)

また、本記事は以下のことを前提とさせていただきます。

- Konnect のアカウントを有していること
  - API 操作するための資格情報（PAT, Access Token）の発行が済んでいること
- Kafka クライアントのインストールなどが済んでいること
  - e.g. [kafkactl](https://github.com/deviceinsight/kafkactl)

### KNEP 用の Control Plane を作成する

まずは、KNEP 用の Control Plane を作成します。

```sh
KONNECT_TOKEN=kpat_...
KONNECT_CONTROL_PLANE_ID=$( curl -X POST "https://us.api.konghq.com/v2/control-planes" \
     -H "Authorization: Bearer $KONNECT_TOKEN" \
     --json '{
       "name": "knep-gateway",
       "cluster_type": "CLUSTER_TYPE_KAFKA_NATIVE_EVENT_PROXY"
     }' | jq -r '.id')
```

作成できると、Konnect UI 上の"Event Gateway"から確認ができます。

![event-gateway](/images/get-started-with-knep/event-gateway.png)

### Apache Kafka のクラスタを作成する

## 終わりに

## 参考情報

https://developer.konghq.com/event-gateway/

https://developer.konghq.com/event-gateway/get-started/
