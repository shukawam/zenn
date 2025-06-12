---
title: "Kong Gateway Operator(KGO) を用いた Kong Ingress Controller(KIC) の構築"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong", "kubernetes", "aws"]
published: false
---

## はじめに

こんにちは 🖐️ 今回は、Kong Ingress Controller(以下、KIC)を構築してみます。Helm でインストールする方法や Kong Gateway Operator(以下、KGO)でインストールする方法が存在しますが、今回は KGO を使ってみます。

## Kong Ingress Controller

Kubernetes クラスタ外から内へのトラフィックを処理するリソースとしては、`Ingress` や `HTTPRoute` などがよく知られています。KIC は、Kubernetes 標準のリソースとして宣言した `Ingress` や `HTTPRoute` の設定項目を Kong Gateway の設定に変換することで、クラスタ内へのトラフィックを処理します。標準的な Kubernetes リソースとして定義できるので、Kong Gateway 自体の学習コストが抑えられるところや Kubernetes の特徴である宣言的なオペレーションに則れるところも良いポイントだと感じました。全体像としては、公式ドキュメントに掲載されている以下の図がとてもイメージしやすいかと思います。

![kic-gateway-arch](https://docs.konghq.com/assets/images/products/kubernetes-ingress-controller/kic-gateway-arch.png)

引用：[https://docs.konghq.com/kubernetes-ingress-controller/3.4.x/](https://docs.konghq.com/kubernetes-ingress-controller/3.4.x/)

また、Ingress のレイヤーで Kong が持つプラグインが活用できるのも嬉しいポイントだと思います。[^1]

[^1]: プラグインの一例：[https://docs.konghq.com/kubernetes-ingress-controller/3.4.x/plugins/authentication/](https://docs.konghq.com/kubernetes-ingress-controller/3.4.x/plugins/authentication/), [https://docs.konghq.com/kubernetes-ingress-controller/3.4.x/plugins/rate-limiting/](https://docs.konghq.com/kubernetes-ingress-controller/3.4.x/plugins/rate-limiting/), [https://docs.konghq.com/kubernetes-ingress-controller/3.4.x/plugins/mtls/](https://docs.konghq.com/kubernetes-ingress-controller/3.4.x/plugins/mtls/)

## Kong Gateway Operator
