---
title: "EKS クラスタの構築メモ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "kubernetes"]
published: false
---

## はじめに

こんにちは 🖐️ 色々な検証作業をするために Kubernetes クラスタが必要になりました。EKS(Amazon Elastic Kubernetes Service) を構築するのは初めてなので自分の作業メモがてら記録しておきます。誰かの参考になれば幸いです。

### eksctl

EKS には、[eksctl](https://eksctl.io/) という公式の CLI が存在し、クラスタの構築やアドオン管理などはこの CLI を使うのが通例みたいです。

```sh
eksctl create cluster
```

というコマンド一発で EKS クラスタが配置される VPC やノードグループ、ロールなどが自動的に作成され、クラスタに接続するための資格情報である `kubeconfig` まで追記されます。振る舞いを変えたい場合は、いくつかパラメータが用意されているので、それを使うと良いでしょう。

例：EKS クラスタの名前を自分で決めて、リージョンを指定（`us-east-1`）する

```sh
eksctl create cluster --name <EKS クラスタ名> --region us-east-1
```

1 shot で実行する場合は、上記で良いと思いますが複数回の構築＆削除が想定される場合は、設定ファイル[^1]に残しておくと何かと便利だと思います。

[^1]: 設定ファイルのスキーマは、[公式ドキュメント](https://eksctl.io/usage/schema/)に記載があります。

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: <EKS クラスタ名>
  region: us-east-1
```

今回は、以下のような設定にしてみました。

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: shukawam-cluster
  region: us-east-1
  version: "1.32"
  tags:
    owner: shukawam
    environment: development
nodeGroups:
  - name: shukawam-nodegroup
    instanceType: m5.large
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    iam:
      withOIDC: true
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        albIngress: true
```

作成するときは、こんな感じで実行すれば OK です。

```sh
eksctl create cluster -f cluster.yaml
```

### StorageClass

PostgreSQL を構築しようと思ったときに、Pod が上がってこないな...と思っていたら PVC に StorageClass が設定されておらず、Pending となっていました。検証用途のクラスタでストレージの性能要件などは考えなくても良いはずなので、デフォルトで入っていた `gp2` を使わせます。今回は、上記の通り EKS Auto Mode で作成したわけではないので、[CSI Driver for Amazon EBS](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/) を入れていきます。

こちらに従って、進めていきます。

https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/ebs-csi.html

まずは、`eksctl` を用いて IAM ロールの作成およびポリシーのアタッチを行います。

```sh
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <EKS クラスタ名> \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

CSI Driver for Amazon EBS をアドオンとしてクラスタに追加します。

```sh
eksctl create addon \
    --name aws-ebs-csi-driver \
    --cluster shukawam-cluster \
    --region us-east-1 \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --force
```

## 終わりに

これで最低限の環境は整ったと思います。あとは、このクラスタで Kong Gateway や KIC(Kong Ingress Controller)などを検証していければと考えています。また、EKS Nativeに使うさまざまな方法（Pod Identity、etc.）などは全くキャッチアップできていないので、おいおいやっていければと思っています 💪
