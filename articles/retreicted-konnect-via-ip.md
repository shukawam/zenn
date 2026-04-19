---
title: "Kong KonnectにIP制限をかける"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: true
---

## はじめに

Aloha🤙 今日は、Kong KonnectのGUIやAPIに対してIP制限をかける方法を解説します。と言ってもやることや設定自体はとても簡単なので、解説記事などは正直不要なレベルなのですが、「そもそもこんな機能あったのか！」という方達に少しでも知ってもらうために書いています。

## やりたいこと

今回は、以下のように自宅の固定IPアドレスからのみKong KonnectのGUIやAPIにアクセスできるようにしたいと思います。また、IPを変えてアクセスができないことを簡易的に確認するために、スマホのテザリングを利用した場合にアクセスができないことを確認します。

![overview](/images/restricted-konnect-via-ip/overview.png)
_IP制限のイメージ_

## 設定方法

Kong KonnectのサイドメニューからOrganization > Settings > Security > IP allow listを選択します。

![konnect-ip-allow-list](/images/restricted-konnect-via-ip/konnect-ip-allow-list.png)
_Kong KonnectのIP allow list設定_

IP allow listの画面で、Add IP addressをクリックして、アクセスを許可したいIPアドレスを入力します。今回は自宅の固定IPアドレスを入力します。
自分の端末のグローバルIPを確認するには、以下のコマンドを実行します。

```sh
curl https://api.ipify.org; echo
```

出てきたIPアドレスをIP rangeに入力して、Createをクリックします。

![konnect-add-ip](/images/restricted-konnect-via-ip/konnect-add-ip.png)
_IPアドレスの追加_

これで設定はすべて完了です。

## 動作確認

一応動作確認を行います。まずは、設定したIPアドレスからKong KonnectのGUIにアクセスしてみます。アクセスが成功すれば、IP制限の設定は正しく行われていることになります。

![konnect-access-success](/images/restricted-konnect-via-ip/konnect-access-success.png)
_Kong Konnectへのアクセスが成功した様子_

APIアクセスも試してみます。以下のコマンドを実行し、Control Planeの一覧を取得してみましょう。

```sh
curl -s -D - -o /dev/null -X GET  https://us.api.konghq.com/v2/control-planes -H "Authorization: Bearer kpat_..."
```

実行結果：

```sh
HTTP/2 200
content-type: application/json; charset=utf-8
ratelimit-reset: 49
x-ratelimit-remaining-minute: 5999
x-ratelimit-limit-minute: 6000
ratelimit-limit: 6000
ratelimit-remaining: 5999
content-security-policy: default-src 'self';base-uri 'self';block-all-mixed-content;font-src 'self' https: data:;form-action 'self';frame-ancestors 'self';img-src 'self' data:;object-src 'none';script-src 'self';script-src-attr 'none';style-src 'self' https: 'unsafe-inline';upgrade-insecure-requests
cross-origin-embedder-policy: require-corp
cross-origin-opener-policy: same-origin
cross-origin-resource-policy: same-origin
x-dns-prefetch-control: off
expect-ct: max-age=0
x-frame-options: SAMEORIGIN
strict-transport-security: max-age=15552000; includeSubDomains
x-download-options: noopen
x-content-type-options: nosniff
origin-agent-cluster: ?1
x-permitted-cross-domain-policies: none
referrer-policy: no-referrer
x-xss-protection: 0
x-datadog-trace-id: 2570007720680280156
x-datadog-parent-id: 5637382832734763252
x-b3-traceid: 000000000000000023aa8067cd28d000
x-b3-spanid: 4e3c01d1780a7800
x-b3-sampled: 1
etag: W/"1530-RsmfRIu7SauxKHhPwzA6nss1D5o"
date: Fri, 17 Apr 2026 01:23:11 GMT
x-envoy-upstream-service-time: 20
vary: Origin
access-control-allow-credentials: true
access-control-expose-headers: Date,x-datadog-trace-id,Konnect-Acting-As,Konnect-Feature-Set,x-intercom-identity-hash
via: kong-enterprise-edition
x-kong-upstream-latency: 22
x-kong-proxy-latency: 78
```

続いて、スマートフォンのテザリングでグローバルIPを変えた状態で同様の操作を試してみます。アクセスが拒否されれば、IP制限の設定が正しく行われていることになります。

![konnect-access-denied](/images/restricted-konnect-via-ip/konnect-access-denied.png)
_Kong Konnectへのアクセスが失敗した様子_

再度APIアクセスも試してみます。以下のコマンドを実行し、Control Planeの一覧を取得してみましょう。

```sh
curl -s -D - -o /dev/null -X GET  https://us.api.konghq.com/v2/control-planes -H "Authorization: Bearer kpat_..."
```

実行結果：

```sh
HTTP/2 403
date: Fri, 17 Apr 2026 01:25:24 GMT
content-length: 179
vary: Origin
access-control-allow-credentials: true
access-control-expose-headers: Date,x-datadog-trace-id,Konnect-Acting-As,Konnect-Feature-Set,x-intercom-identity-hash
via: kong-enterprise-edition
x-kong-response-latency: 72
```

設定したIPアドレス外ではアクセスが拒否されていることが確認できました。

## おわりに

今回は、Kong KonnectのGUIやAPIに対してIP制限をかける方法を解説しました。設定自体はとても簡単ですが、会社の拠点IPレンジからしか接続を許可したくないケースなどに対応するために有効な機能なので、ぜひ活用してみてくださいね！それでは、Mahalo!🌺
