---
title: "Kong Gateway(EE) を EKS 上に構築してみた"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong", "aws", "kubernetes"]
published: false
---

## はじめに

API Gateway として知られている [Kong Gateway](https://docs.konghq.com/gateway/latest/) を Kubernetes(EKS)上に構築してみます。Kubernetes 上に展開する手段は、以下の通りいくつか存在します。

- DB-less なパターン
- Kong Konnect[^1] を使うパターン
- Kubernetes クラスタ内で Control Plane と Data Plane を分けるパターン(Hybrid mode)
- 各ノードが PostgreSQL に接続されているパターン（Traditional mode）

[^1]: Kong 社が提供する SaaS 型の API マネジメントプラットフォームのこと

今回は、この中でも Hybrid mode を試してみようと思います。

### 構築手順

基本手順は、以下のドキュメントに従いますが今後のメンテナンス性やクラスタの状態が確認しやすくなることなどを鑑みて [Argo CD](https://argoproj.github.io/cd/) を用いた GitOps で構築します。尚、Argo CD 自体の構築手順やプロジェクトの作成などは本記事のスコープ外とさせていただきます。

https://argoproj.github.io/cd/

ざっと手順を眺めてみると以下のような流れになっていることがわかります。

1. `charts.konghq.com` の Helm レポジトリを追加する（Argo CD で構築するなら省略可）
2. Enterprise License を Kubernetes Secret として登録しておく
3. Control/Data Plane 間の通信を相互 TLS で保護するための証明書を Kubernetes Secret として登録しておく
4. Control Plane をインストールする
5. Data Plane をインストールする
6. Kong Gateway に Service/Route を作成し、疎通確認を行う

順番に見ていきます。

#### Enterprise License を Kubernetes Secret として登録しておく

ドキュメント通りに進めていきます。

```sh
kubectl create namespace kong
kubectl create secret generic kong-enterprise-license --from-file=license=license.json -n kong
```

一応確認します。

```sh
kubectl get secret -n kong | grep -i kong-enterprise-license
kong-enterprise-license   Opaque              1      23h
```

#### Control/Data Plane 間の通信を相互 TLS で保護するための証明書を Kubernetes Secret として登録しておく

X509 証明書を作成します。

```sh
openssl req \
    -new -x509 -nodes \
    -newkey ec:<(openssl ecparam -name secp384r1) \
    -keyout ./tls.key \
    -out ./tls.crt \
    -days 1095 \
    -subj "/CN=kong_clustering"
```

作成された証明書と秘密鍵を Kubernetes の Secret として登録します。

```sh
kubectl create secret tls kong-cluster-cert \
    --cert=./tls.crt \
    --key=./tls.key \
    -n kong
```

一応確認します。

```sh
kubectl get secret kong-cluster-cert -n kong | grep -i kong-cluster-cert
kong-cluster-cert   kubernetes.io/tls   2      23h
```

#### Control Plane をインストールする

Kong Gateway における Control Plane は、実際の API トラフィックに対するリバースプロキシを担当する Data Plane に対して、設定を配布したりそのための設定を受け付けたりする役割を持ちます。Hybrid mode で構築すると以下のような全体像となります。

![kong-ee-with-eks](/images/kong-ee-with-eks/hybrid-mode-architecture.png)

引用：[https://docs.konghq.com/gateway/3.10.x/production/deployment-topologies/hybrid-mode/](https://docs.konghq.com/gateway/3.10.x/production/deployment-topologies/hybrid-mode/)

まずは、ドキュメントに記載がある `values-cp.yaml` を見てみましょう。

```yaml
# Do not use Kong Ingress Controller
ingressController:
  enabled: false

image:
  repository: kong/kong-gateway
  tag: "3.10.0.2"

# Mount the secret created earlier
secretVolumes:
  - kong-cluster-cert

env:
  # This is a control_plane node
  role: control_plane
  # These certificates are used for control plane / data plane communication
  cluster_cert: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_cert_key: /etc/secrets/kong-cluster-cert/tls.key

  # Database
  # CHANGE THESE VALUES
  database: postgres
  pg_database: kong
  pg_user: kong
  pg_password: demo123
  pg_host: kong-cp-postgresql.kong.svc.cluster.local
  pg_ssl: "on"

  # Kong Manager password
  password: kong_admin_password

# Enterprise functionality
enterprise:
  enabled: true
  license_secret: kong-enterprise-license

# The control plane serves the Admin API
admin:
  enabled: true
  http:
    enabled: true

# Clustering endpoints are required in hybrid mode
cluster:
  enabled: true
  tls:
    enabled: true

clustertelemetry:
  enabled: true
  tls:
    enabled: true

# Optional features
manager:
  enabled: false

# These roles will be served by different Helm releases
proxy:
  enabled: false
```

何点かポイントを解説します。まずは、Ingress Controller についてです。Kong Gateway のインストール時に Kong Ingress Controller(以下、KIC)を同時にインストールさせるかのフラグですが、個人的に KIC は KIC として管理したいので、ここはそのままとしておきます。

```yaml
# Do not use Kong Ingress Controller
ingressController:
  enabled: false
```

続いて、Hybrid mode としてクラスタを組むために必要な設定がこちらになります。ここでは、作成するノードの役割を `Control Plane` として指定しています。

```yaml
env:
  # This is a control_plane node
  role: control_plane
  # These certificates are used for control plane / data plane communication
  cluster_cert: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_cert_key: /etc/secrets/kong-cluster-cert/tls.key
```

余談ですが、`role` として設定できる項目は [https://github.com/Kong/kong/blob/master/kong.conf.default#L267-L283](https://github.com/Kong/kong/blob/master/kong.conf.default#L267-L283) に記載がある通り、`traditional`, `control_plane`, `data_plane` が指定可能です。

```conf
#------------------------------------------------------------------------------
# HYBRID MODE
#------------------------------------------------------------------------------

#role = traditional              # Use this setting to enable hybrid mode,
                                 # This allows running some Kong nodes in a
                                 # control plane role with a database and
                                 # have them deliver configuration updates
                                 # to other nodes running to DB-less running in
                                 # a data plane role.
                                 #
                                 # Valid values for this setting are:
                                 #
                                 # - `traditional`: do not use hybrid mode.
                                 # - `control_plane`: this node runs in a
                                 #   control plane role. It can use a database
                                 #   and will deliver configuration updates
                                 #   to data plane nodes.
                                 # - `data_plane`: this is a data plane node.
                                 #   It runs DB-less and receives configuration
                                 #   updates from a control plane node.
```

また、`role` として読み込んだ値によって追加の設定項目（相互 TLS 用の証明書と秘密鍵）を読み込むようになっているみたいです。[^2]

[^2]: [https://github.com/Kong/kong/blob/master/kong/conf_loader/parse.lua#L696-L722](https://github.com/Kong/kong/blob/master/kong/conf_loader/parse.lua#L696-L722)

```lua
  if conf.role == "control_plane" or conf.role == "data_plane" then
    local cluster_cert = conf.cluster_cert
    local cluster_cert_key = conf.cluster_cert_key
    local cluster_ca_cert = conf.cluster_ca_cert

    if not cluster_cert or not cluster_cert_key then
      errors[#errors + 1] = "cluster certificate and key must be provided to use Hybrid mode"

    else
      if not exists(cluster_cert) then
        cluster_cert = try_decode_base64(cluster_cert)
        conf.cluster_cert = cluster_cert
        local _, err = openssl_x509.new(cluster_cert)
        if err then
          errors[#errors + 1] = "cluster_cert: failed loading certificate from " .. cluster_cert
        end
      end

      if not exists(cluster_cert_key) then
        cluster_cert_key = try_decode_base64(cluster_cert_key)
        conf.cluster_cert_key = cluster_cert_key
        local _, err = openssl_pkey.new(cluster_cert_key)
        if err then
          errors[#errors + 1] = "cluster_cert_key: failed loading key from " .. cluster_cert_key
        end
      end
    end
```

Kong Manager という GUI ツールを有効にしたい場合は、以下の設定値を変更してください。

```yaml
# Optional features
manager:
  enabled: false # 有効にしたいなら、trueにする
```

前述した通り、Data Plane は API に対するリバースプロキシをするわけではないので、以下のような設定になっています。

```yaml
# These roles will be served by different Helm releases
proxy:
  enabled: false
```

脱線しましたが、Argo CD のアプリケーションとする場合は、以下のように書けば OK です。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kong-cp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: kong
  source:
    repoURL: https://charts.konghq.com
    chart: kong
    targetRevision: 2.48.0
    helm:
      values: |
        # Do not use Kong Ingress Controller
        ingressController:
          enabled: false
        image:
          repository: kong/kong-gateway
          tag: "3.10.0.2"
        # Mount the secret created earlier
        secretVolumes:
          - kong-cluster-cert
        env:
          # This is a control_plane node
          role: control_plane
          # These certificates are used for control plane / data plane communication
          cluster_cert: /etc/secrets/kong-cluster-cert/tls.crt
          cluster_cert_key: /etc/secrets/kong-cluster-cert/tls.key
          # Database
          # CHANGE THESE VALUES
          database: postgres
          pg_database: kong
          pg_user: kong
          pg_password: demo123
          pg_host: kong-cp-postgresql.kong.svc.cluster.local
          pg_ssl: "on"
          # Kong Manager password
          password: kong_admin_password
        # Enterprise functionality
        enterprise:
          enabled: true
          license_secret: kong-enterprise-license
        # The control plane serves the Admin API
        admin:
          enabled: true
          http:
            enabled: true
        # Clustering endpoints are required in hybrid mode
        cluster:
          enabled: true
          tls:
            enabled: true
        clustertelemetry:
          enabled: true
          tls:
            enabled: true
        # Optional features
        manager:
          enabled: false
        # These roles will be served by different Helm releases
        proxy:
          enabled: false
        # This is for testing purposes only
        # DO NOT DO THIS IN PRODUCTION
        # Your cluster needs a way to create PersistentVolumeClaims
        # if this option is enabled
        postgresql:
          enabled: true
          auth:
            password: demo123
  destination:
    server: https://kubernetes.default.svc
    namespace: kong
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

これをクラスタ上に展開します。

```yaml
kubectl apply -f kong-cp.yaml
```

GitHub などのリモートリポジトリと同期を取るためのアプリケーションなどを定義したい場合は、以下を参考にしてください。

https://github.com/shukawam/kong-cluster

以下のような状態になっていれば、Control Plane のインストールは完了です。

```sh
kubectl get pods -n kong | grep -i kong-cp
kong-cp-kong-756d948cdb-4bst9                1/1     Running     0          4h53m
kong-cp-kong-init-migrations-28cvc           0/1     Completed   0          4h45m
kong-cp-kong-post-upgrade-migrations-rlgmh   0/1     Completed   0          4h45m
kong-cp-kong-pre-upgrade-migrations-4sj68    0/1     Completed   0          4h45m
kong-cp-postgresql-0                         1/1     Running     0          4h53m
```

#### Data Plane をインストールする

続いて、Data Plane をインストールします。ドキュメントに記載がある `values-dp.yaml` を見てみましょう。

```yaml
# Do not use Kong Ingress Controller
ingressController:
  enabled: false

image:
  repository: kong/kong-gateway
  tag: "3.10.0.2"

# Mount the secret created earlier
secretVolumes:
  - kong-cluster-cert

env:
  # data_plane nodes do not have a database
  role: data_plane
  database: "off"

  # Tell the data plane how to connect to the control plane
  cluster_control_plane: kong-cp-kong-cluster.kong.svc.cluster.local:8005
  cluster_telemetry_endpoint: kong-cp-kong-clustertelemetry.kong.svc.cluster.local:8006

  # Configure control plane / data plane authentication
  lua_ssl_trusted_certificate: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_cert: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_cert_key: /etc/secrets/kong-cluster-cert/tls.key

# Enterprise functionality
enterprise:
  enabled: true
  license_secret: kong-enterprise-license

# The data plane handles proxy traffic only
proxy:
  enabled: true

# These roles are served by the kong-cp deployment
admin:
  enabled: false

manager:
  enabled: false
```

こちらもポイントを見ていきます。Control Plane 同様に、KIC の管理は KIC として行いたいので、無効化のまま進めていきます。

```yaml
# Do not use Kong Ingress Controller
ingressController:
  enabled: false
```

また、このノードは Data Plane として稼働させたいため、以下のように設定されています。Gateway の設定値を保存する Database と通信するのは、Hybrid mode の場合は Control Plane のみなので `database` のフラグも `off` にしてあります。Control Plane と相互 TLS で通信するための証明書や秘密鍵も同様に設定してあります。

```yaml
env:
  # data_plane nodes do not have a database
  role: data_plane
  database: "off"
  # ... 省略 ...
  # Configure control plane / data plane authentication
  lua_ssl_trusted_certificate: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_cert: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_cert_key: /etc/secrets/kong-cluster-cert/tls.key
```

また、Data Plane は実際の API トラフィックのリバースプロキシとして動作するので、`proxy` に関する設定が有効化されています。

```yaml
# The data plane handles proxy traffic only
proxy:
  enabled: true
```

Argo CD のアプリケーションとする場合は、以下のようにすれば OK です。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kong-dp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: kong
  source:
    repoURL: https://charts.konghq.com
    chart: kong
    targetRevision: 2.48.0
    helm:
      values: |
        ingressController:
          enabled: false
        image:
          repository: kong/kong-gateway
          tag: "3.10.0.2"
        secretVolumes:
          - kong-cluster-cert
        env:
          role: data_plane
          database: "off"
          cluster_control_plane: kong-cp-kong-cluster.kong.svc.cluster.local:8005
          cluster_telemetry_endpoint: kong-cp-kong-clustertelemetry.kong.svc.cluster.local:8006
          lua_ssl_trusted_certificate: /etc/secrets/kong-cluster-cert/tls.crt
          cluster_cert: /etc/secrets/kong-cluster-cert/tls.crt
          cluster_cert_key: /etc/secrets/kong-cluster-cert/tls.key
          enterprise:
            enabled: true
            license_secret: kong-enterprise-license
          proxy:
            enabled: true
          admin:
            enabled: false
          manager:
            enabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: kong
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

これをクラスタ上に展開します。

```sh
kubectl apply -f kong-cp.yaml
```

GitHub などのリモートリポジトリと同期を取るためのアプリケーションなどを定義したい場合は、以下を参考にしてください。

https://github.com/shukawam/kong-cluster

以下のような状態になっていれば、Data Plane のインストールは完了です。

```sh
kubectl get pods -n kong | grep -i kong-dp
kong-dp-kong-694b8cf69f-scsxl                1/1     Running     0          4h55m
```

#### Kong Gateway に Service/Route を作成し、疎通確認を行う

まずは、何も設定されていないので架空の Service/Route に対してリクエストを送ります。

```sh
# Load BalancerのIPを取得
PROXY_IP=$(kubectl get service --namespace kong kong-dp-kong-proxy -o jsonpath='{range .status.loadBalancer.ingress[0]}{@.ip}{@.hostname}{end}')
# 架空のService/Routeに対してリクエストを送る
curl $PROXY_IP/mock/anything
```

以下のような結果が返ってくれば OK です。

```json
{
  "message": "no Route matched with those values",
  "request_id": "35970b105df5193fe1f0287a36a21f4e"
}
```

続いて、Admin API を用いて Service/Route の登録を行います。

```sh
# KongのAdmin APIをポートフォワードしておく
kubectl port-forward -n kong service/kong-cp-kong-admin 8001
# Admin APIを用いてServiceの作成
curl localhost:8001/services -d name=mock  -d url="https://httpbin.konghq.com"
# Admin APIを用いてRouteの作成
curl localhost:8001/services/mock/routes -d "paths=/mock"
```

再度、リクエストを送って以下のような結果が返ってくれば OK です。

```sh
curl $PROXY_IP/mock/anything
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "httpbin.konghq.com",
    "User-Agent": "curl/8.7.1",
    "X-Forwarded-Host": "a4a5383339fbc4665b9faa90b6f9e0da-204037230.us-east-1.elb.amazonaws.com",
    "X-Forwarded-Path": "/mock/anything",
    "X-Forwarded-Prefix": "/mock",
    "X-Kong-Request-Id": "7315d7594038b74133b459730b414e5b"
  },
  "json": null,
  "method": "GET",
  "origin": "192.168.59.205",
  "url": "http://a4a5383339fbc4665b9faa90b6f9e0da-204037230.us-east-1.elb.amazonaws.com/anything"
}
```

## 終わりに

今回は、Kubernetes(EKS)上に Kong Gateway を Hybrid mode で構築してみました。これからは、この構築した Gateway に対してオブザーバビリティ関連プラグインや認証・認可関連のプラグインを設定してみたりして色々遊んでみたいと思います。また、Kong Ingress Controller(KIC)や Kong Mesh なども同様に試していければと思います 🖐️

また、Helm を用いてデプロイする場合はさまざまな設定例がリポジトリで公開されているので、構築したい例に合わせて参照すると良いでしょう。

https://github.com/Kong/charts/tree/main/charts/kong/example-values
