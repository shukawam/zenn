---
title: "Kong Operator の KonnectAPIAuthConfiguration を Secret から参照して作成する"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: true
published_at: 2025-12-22 00:00
---

## はじめに

こんにちは 🖐️ こちらの記事は、[Kong Advent Calendar 2025](https://qiita.com/advent-calendar/2025/kong) の Day22 として書かれています。今回は、タイトルにある通り Kong Gateway の構築や設定を Kubernetes-way で実現するための Kong Operator に関する内容です。具体定には、Kong Operator を利用して、Kong Konnect 上に Control Plane（以下、CP）を作成するために Konnect の API を実行するための資格情報を CR(Custom Resource)として作成する必要があるのですが、その時にハマった内容についてです。

## 何で困っていたか？

https://developer.konghq.com/operator/dataplanes/get-started/hybrid/deploy-dataplane/#create-a-konnectapiauthconfiguration-resource

上記のドキュメントに記されている通り、Konnect のリソースを操作するためには `KonnectAPIAuthConfiguration` という資格情報を CR として以下のように作成する必要があります。

```yaml
kind: KonnectAPIAuthConfiguration
apiVersion: konnect.konghq.com/v1alpha1
metadata:
  name: konnect-api-auth
  namespace: kong
spec:
  type: token
  token: "'$KONNECT_TOKEN'"
  serverURL: us.api.konghq.com
```

ここで、`token` フィールドに Konnect の API アクセスに必要な PAT(Personal Access Token)やシステムアカウントのトークンを設定する必要があります。しかし、こちらのマニフェストファイルをそのまま GitHub などで管理してしまうと、トークンが記載されたままになってしまいます。これを避けるために、`KonnectAPIAuthConfiguration` では、Secret リソースから参照するためのフィールド（`secretRef`）が用意されています。

```sh
$ kubectl explain konnectapiauthconfiguration.spec
GROUP:      konnect.konghq.com
KIND:       KonnectAPIAuthConfiguration
VERSION:    v1alpha1

FIELD: spec <Object>


DESCRIPTION:
    Spec is the specification of the KonnectAPIAuthConfiguration resource.

FIELDS:
  secretRef     <Object>
    SecretRef is a reference to a Kubernetes Secret containing the Konnect
    token.
    This secret is required to have the konghq.com/credential label set to
    "konnect".

  serverURL     <string> -required-
    ServerURL is the URL of the Konnect server.
    It can be either a full URL with an HTTPs scheme or just a hostname.
    Please refer to https://docs.konghq.com/konnect/network/ for the list of
    supported hostnames.

  token <string>
    Token is the Konnect token used to authenticate with the Konnect API.

  type  <string> -required-
  enum: token, secretRef
    KonnectAPIAuthType is the type of authentication used to authenticate with
    the Konnect API.
```

`secretRef` フィールドの説明を参照してみると、読み込む Secret リソースとしては `konghq.com/credential` ラベルが `konnect` に設定されている必要があることがわかります。これはドキュメントを参照しても同様のことが記載されています。

https://developer.konghq.com/operator/reference/custom-resources/#konnect-konghq-com-v1alpha1-types-konnectapiauthconfigurationspec

これに則って、以下のように Secret リソースを作成してみました。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: konnect-token
  namespace: kong
  labels:
    konghq.com/credential: konnect
data:
  token: <base64 encoded token>
```

これを参照するように、 `KonnectAPIAuthConfiguration` を以下のように作成しました。

```yaml
kind: KonnectAPIAuthConfiguration
apiVersion: konnect.konghq.com/v1alpha1
metadata:
  name: konnect-api-auth
  namespace: kong
spec:
  type: secretRef
  secretRef:
    name: konnect-token
    namespace: kong
  serverURL: us.api.konghq.com
```

きちんと、`KonnectAPIAuthConfiguration` が作成できているかを確認してみます。

```sh
$ kubectl get konnectapiauthconfiguration -n kong
NAME               VALID   ORGID   SERVERURL
konnect-api-auth   False
```

有効性が `False` になっています。Kong Operator のログも参照してみましょう。

```sh
$ kubectl logs -f kong-operator-kong-operator-controller-manager-6c5467f874-76wmg -n kong-system
# ... omit ...
{"level":"error","ts":"2025-12-17T08:17:12Z","msg":"Reconciler error","controller":"KonnectAPIAuthConfiguration","controllerGroup":"konnect.konghq.com","controllerKind":"KonnectAPIAuthConfiguration","KonnectAPIAuthConfiguration":{"name":"konnect-api-auth","namespace":"kong"},"namespace":"kong","name":"konnect-api-auth","reconcileID":"7b89b5d2-9131-4efe-be08-8a1e54fc3ee8","error":"failed to get Secret kong/konnect-token: Secret \"konnect-token\" not found","stacktrace":"sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).reconcileHandler\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.21.0/pkg/internal/controller/controller.go:353\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).processNextWorkItem\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.21.0/pkg/internal/controller/controller.go:300\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller[...]).Start.func2.1\n\t/home/runner/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.21.0/pkg/internal/controller/controller.go:202"}
```

Operator が Secret リソースを見つけられていないそうです。

## じゃあ、どうすれば？

Secret リソースには、追加で以下のようなラベルを追加する必要があります。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: konnect-token
  namespace: kong
  labels:
    konghq.com/secret: "true" # これを追加する
    konghq.com/credential: konnect
data:
  token: <base64 encoded token>
```

実際に、`KonnectAPIAuthConfiguration` を確認してみると、有効性が `True` になっていることがわかります。

```sh
$ kubectl get konnectapiauthconfiguration -n kong
NAME               VALID   ORGID                                  SERVERURL
konnect-api-auth   True    c6a336ca-19ae-44b0-b422-050cb20399f8   https://us.api.konghq.com
```

## おまけ

Controller が Secret リソースに要求するラベルはこちらに記載があります。

https://github.com/Kong/kong-operator/blob/main/controller/konnect/reconciler_konnectapiauth.go#L282-L284

ここで記載のある `SecretCredentialLabel`, `SecretCredentialLabelValueKonnect` は、それぞれ以下のように定義されており、きちんとドキュメントの説明と一致しています。

https://github.com/Kong/kong-operator/blob/main/controller/konnect/reconciler_konnectapiauth.go#L45-L54

では、`konghq.com/secret: "true"` のラベルについてはどこで記載されているかというと、ValidatingWebhook に記載があります。

https://github.com/Kong/kong-operator/blob/main/charts/kong-operator/templates/validating-webhook.yaml#L114-L143

## おわりに

今回は、Kong Operator で Secret リソースを参照して KonnectAPIAuthConfiguration を作成する際にハマったことについての共有でした。いつかドキュメント側が修正されることを願いますが、それまではこちらの記事が参考になれば幸いです。
