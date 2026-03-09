---
title: "AWS Lambda + Kong GatewayでMCPサーバーを構築する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong", "aws", "mcp"]
published: true
---

## はじめに

こちらの記事では、AWS Lambdaで実装したAPIをKong Gatewayを使ってMCPサーバーとして公開する方法を紹介します。類似の機能として、[Amazon Bedrock AgentCore Gateway](https://aws.amazon.com/blogs/machine-learning/introducing-amazon-bedrock-agentcore-gateway-transforming-enterprise-ai-agent-tool-development/)もありますが、マルチクラウドに展開されている場合やMCP以外にLLMのAPIも統合管理したい、付加機能を追加したい場合などは、Kong Gatewayを利用するのも一つの選択肢となります。

## 作ってみる環境

以下のような環境を作成してみます。

![architecture](/images/mcp-server-with-aws-lambda/architecture.png)
_環境の概要図_

AWS Lambdaで実装されたAPIをKong Gatewayを使ってMCPサーバーとして公開します。Kong GatewayからAWS Lambdaの実行は、[AWS Lambdaプラグイン](https://developer.konghq.com/plugins/aws-lambda/)を利用し、MCP/REST変換については、[AI MCP Proxyプラグイン](https://developer.konghq.com/plugins/ai-mcp-proxy/)を利用することで実現します。

## 設定例

まずは、以下のような関数を書き、AWS Lambdaへデプロイします。

```js
const catalogue = [
  {id: 1, name: 'Cool T-shirt', description: 'このTシャツを着れば、あなたのクールさが120%増加することを保証します（効果には個人差があります）。洗濯するとクールさが少し減りますが、乾燥機で取り戻せます。'},
  {id: 2, name: 'Magic Pants', description: '魔法のパントス。このパントスを履けば、あなたは魔法使いになれます。ただし、魔法は1日1回しか使えないです。'},
  {id: 3, name: 'Invisible Shoes', description: '見えない靴。この靴を履けば、あなたは透明になります。ただし、走ると「ゴソゴソ」音がするので、隠れられないことがあります。'},
  {id: 4, name: 'Flying Hat', description: '飛行帽。この帽子をかぶれば、あなたは空を飛べます。ただし、風が強いと落ちてきます。'}
]

export const handler = async (event) => {
  const response = {
    statusCode: 200,
    body: {data: catalogue}
  };
  return response;
};
```

次に、Kong Gatewayの設定を行います。設定は、decKを利用して宣言的に行いますが、慣れていない方はGUIから設定しても大丈夫です。なお、Kong Gatewayは、Amazon EC2などに構築しておきEC2からLambdaを呼び出すためのIAMロールを作成し、EC2にIAMロールを割り当てておくと良いでしょう[^1]。

```yaml
_format_version: "3.0"

services:
  - name: httpbin-service
    url: http://httpbin.org
    routes:
      - name: httpbin-route
        paths:
          - /catalogue
        strip_path: true
  - name: mcp-service
    url: http://localhost:8000
    routes:
      - name: mcp-route
        paths:
          - /mcp
        strip_path: true

plugins:
  - name: aws-lambda
    route: httpbin-route
    config:
      aws_region: ap-northeast-1
      function_name: shukawam-catalogue
      invocation_type: RequestResponse
  - name: ai-mcp-proxy
    route: httpbin-route
    config:
      mode: conversion-listener
      tools:
        - description: Get catalogue
          method: GET
          path: /catalogue
          annotations:
            title: Get catalogue
```

設定内容を簡単に解説します。まずは、以下の部分についてです。

```yaml
services:
  # ... omit ...
  - name: httpbin-service
    url: http://httpbin.org
    routes:
      - name: httpbin-route
        paths:
          - /catalogue
        strip_path: true

plugins:
  # ... omit ...
  - name: aws-lambda
    route: httpbin-route
    config:
      aws_region: ap-northeast-1
      function_name: shukawam-catalogue
      invocation_type: RequestResponse
```

ServiceとしてRouteのパスにアクセスがあった際の転送先を指定していますが、実はこの設定はたいした意味を持っていません。AWS Lambdaプラグインは、今回の`httpbin-route`に対してフォワードプロキシとして振る舞うため、本来の転送先である `https://httpbin.org` は利用されず、AWS Lambdaが呼び出されます。また、AWS Lambdaプラグインの設定では、AWSリージョンや関数名を指定します。

次に以下の部分についてです。

```yaml
services:
  # ... omit ...
  - name: mcp-service
    url: http://localhost:8000
    routes:
      - name: mcp-route
        paths:
          - /mcp
        strip_path: true

plugins:
  # ... omit ...
  - name: ai-mcp-proxy
    route: mcp-route
    config:
      mode: conversion-listener
      tools:
        - description: Get catalogue
          method: GET
          path: /catalogue
          annotations:
            title: Get catalogue
```

この部分では、AWS LambdaをMCPサーバーとして公開するための設定を行なっています。ポイントは、転送先設定であるServiceにKong Gateway自身の公開ポートを指定している点です。これにより、AWS Lambdaプラグインで実行するAPIをMCPサーバーとして公開可能になります。AI MCP Proxyプラグインでは、いくつかの動作モードがありますが今回は`conversion-listener`というモードを利用しています。これにより、Kong GatewayがMCP/REST変換のリスナーとして振る舞うようになります。toolsの部分では、MCPサーバーとして公開するAPIの情報を記載します。

AI MCP Proxyプラグインの動作モードについては以下の通りです。

| 動作モード             | 説明                                                                                                                                                                    |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `passthrough-listener` | MCPリクエストを受信し、Gatewayサービスの`upstream_url`に転送する（REST/MCP変換は行なわれず、既存で運用しているMCPサーバーがあることが前提）                             |
| `conversion-listener`  | REST APIのパスをMCPツールに変換する（既存のREST APIをMCPサーバーとして公開するために利用する）                                                                          |
| `conversion-only`      | REST APIのパスをMCPツールに変換するが、MCPクライアントからのリクエストは受け付けない（後述する`listener`モードを組み合わせて、複数MCPサーバーをまとめるために利用する） |
| `listener`             | `conversion-only`と組み合わせて、複数のMCPサーバーを統合するために利用する                                                                                              |

[^1]: Kong GatewayからAWS Lambdaを呼び出すための準備については、こちらが参考になりました: [Kong KonnectでKong GatewayをEC2にデプロイしLambdaを実行してみる](https://dev.classmethod.jp/articles/kong-konnect-ec2-lambda/)

## 試してみる

Claude Codeを使って試してみます。Claude Codeの設定を以下のように行います（必要箇所のみ抜粋しています）。

```json
"mcpServers": {
    "catalogue": {
        "type": "http",
        "url": "http://<GATEWAY_IP>:8000/mcp"
    }
}
```

これで、Claude Codeから`catalogue`というMCPサーバーに対してリクエストを送ると、AWS Lambdaで実装したAPIが呼び出されるようになります。

```sh
❯ 管理しているカタログの一覧を教えて

⏺ catalogue - Get catalogue (MCP)
  ⎿  {
       "body": {
         "data": [
     … +24 lines (ctrl+o to expand)

⏺ 管理しているカタログの一覧です：

  ┌─────┬─────────────────┬───────────────────────────────────────────────────────────┐
  │ ID  │     商品名      │                           説明                            │
  ├─────┼─────────────────┼───────────────────────────────────────────────────────────┤
  │ 1   │ Cool T-shirt    │ クールさが120%増加するTシャツ（洗濯で減少、乾燥機で回復） │
  ├─────┼─────────────────┼───────────────────────────────────────────────────────────┤
  │ 2   │ Magic Pants     │ 1日1回魔法が使える魔法のパンツ                            │
  ├─────┼─────────────────┼───────────────────────────────────────────────────────────┤
  │ 3   │ Invisible Shoes │ 透明になれる靴（走ると音がする）                          │
  ├─────┼─────────────────┼───────────────────────────────────────────────────────────┤
  │ 4   │ Flying Hat      │ 空を飛べる帽子（強風時は落下注意）                        │
  └─────┴─────────────────┴───────────────────────────────────────────────────────────┘

  合計4件の商品が登録されています。
```

## おわりに

今回は、AWS Lambdaで作成したAPIをKong Gatewayを使ってMCPサーバーとして公開する方法を紹介しました。AWS LambdaプラグインとAI MCP Proxyプラグインを組み合わせることで、簡単に実現できることがわかります。AWS Lambda以外にも、Kong Gatewayは様々なバックエンドと連携可能なので、マルチクラウド環境や既存のREST APIをMCPサーバーとして公開したい場合などに活用できると思います。ぜひ試してみてください！
