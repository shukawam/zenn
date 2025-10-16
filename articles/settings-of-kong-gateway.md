---
title: "Kong Gateway の設定反映について"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: false
---

## はじめに

Kong Gateway のクラスタリング（特に、耐障害性）に関するお話です。本番環境では、Control Plane（以下、CP） と Data Plane（以下、DP） を分離して構築するハイブリッド・モードと呼ばれるデプロイ方式が推奨されています。このとき、実際に API トラフィックをプロキシする DP に対する設定反映は、CP から公開されている Admin API を実行することで行われます。ざっくり図解すると以下のようになります。

![overview-of-sync-config](/images/availability-of-kong-gateway/overview-of-sync-config.png)

ここで、CP に障害が発生した場合、DP はどう振る舞うのでしょうか。API トラフィックはプロキシされなくなる？それともそのままプロキシされる？などなど Kong のクラスタリングに興味が湧いてしまったので、ソースコードを見ながらドキュメントに記述されている内容を紐解いて行きたいと思います。

:::message
ソースコードは、GitHub 上で Tag が打たれている最新バージョンである 3.9.1 に基づきます。ブログ中にコードを引用する場合は、見やすさや理解の助けのためにコメント行などで適宜追記を加えています。

https://github.com/Kong/kong/tree/3.9.1
:::

## まずはドキュメントを読んでみる

実は冒頭の疑問に対する答えはドキュメントに記載があります。

https://developer.konghq.com/gateway/cp-dp-communication/#faqs

> **What happens if the Control Plane and Data Plane nodes disconnect?**
> If a Data Plane node becomes disconnected from its Control Plane, configuration can’t travel between them. In that situation, the Data Plane node continues to use cached configuration until it reconnects to the Control Plane and receives new configuration.
> Whenever a connection is re-established with the Control Plane, it pushes the latest configuration to the Data Plane node. It doesn’t queue up or try to apply older changes.
> If your Control Plane is a Kong Mesh global Control Plane, see Kong Mesh failure modes for connectivity issues.

ということで、CP に障害が発生し DP との通信ができなくなった場合は DP がキャッシュしている設定を使い続け、API トラフィックをプロキシし続けるということがわかります。大まかな動作が想像できたところで実際にコードを読み理解を深めていきます。

## コードを読み理解を深める

Kong Gateway のクラスタリングに関するコードは以下にまとまっています。今回はこのあたりを中心に読んでいきます。

https://github.com/Kong/kong/tree/3.9.1/kong/clustering

クラスタリングに関するエントリーポイントは、`kong/clustering/init.lua` になります。

```lua:kong/clustering/init.lua
function _M:init_worker()
  -- ... 省略(プラグインやフィルターに関するコード) ...

  -- kong.confのroleを読み取る
  local role = self.conf.role

  -- roleがcontrol_planeだった場合の初期化処理
  if role == "control_plane" then
    self:init_cp_worker(basic_info)
    return
  end

  -- roleがdata_planeだった場合の初期化処理
  if role == "data_plane" then
    self:init_dp_worker(basic_info)
  end
end
```

まずは、CP としてノードが起動された時の動きを見ていきます。`init_cp_worker()` の中身は以下のようになっており、`kong/clustering/events.lua` の `init()` と `kong/clustering/control_plane.lua` の `init_worker()` が呼ばれていることがわかります。

```lua:kong/clustering/init.lua
function _M:init_cp_worker(basic_info)

  events.init()

  self.instance = require("kong.clustering.control_plane").new(self)
  self.instance:init_worker(basic_info)
end
```

`kong/clustering/events.lua` の `init()` は以下のようになっており、`clustering:push_config` というイベントをサブスクライブし、イベントが検知されたら `handle_clustering_push_config_event` を実行するイベントハンドラと `dao:crud` というイベントを受け取り、`handle_dao_crud_event` を実行するイベントハンドラを登録しています。

```lua:kong/clustering/events.lua
local function init()
  cluster_events = assert(kong.cluster_events)
  worker_events  = assert(kong.worker_events)

  -- ... 省略 ...
  cluster_events:subscribe("clustering:push_config", handle_clustering_push_config_event)

  -- ... 省略 ...
  worker_events.register(handle_dao_crud_event, "dao:crud")
end
```
