---
title: "Amazon Bedrock AgentCore RuntimeをKong Gatewayを通して実行する"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "bedrock", "kong"]
published: true
---

## はじめに

こんにちは。今回は、Amazon Bedrock AgentCore RuntimeをKong Gateway経由で実行してみたいと思います。Kong Gatewayは、今まで提供してきたAPI Gateway機能に加え、LLM/MCP/A2A/Eventを管理するためのGatewayを提供しています。AgentCore Runtimeのような便利な実行基盤に加え、エージェントを実行するための非機能要件（LLMのAPI呼び出しにおけるFallback、レート制限、オブザーバビリティ、認証・認可、などなど）をKong Gatewayが巻き取るような構成も十分考えうるのではないかと思います。

![agentcore-with-kong](/images/agentcore-with-kong-gateway/agentcore-with-kong.png)
_Amazon Bedrock AgentCoreとKong Gateway_

## 構成

今回は上記の構成のうち、クライアントからKong Gatewayを通してAgentCore Runtimeにデプロイしたエージェントを呼ぶ構成を試します。まず、Amazon Bedrock AgentCore Runtimeにデプロイしたエージェントを実行するためには、インバウンド認証を考慮する必要があります。この認証方法は以下の2種類の方式がサポートされています。

- IAM SigV4認証 - いわゆるAWS IAMベースの認証方法で、APIリクエストに署名を付与することで経路の安全性を保証する仕組み
- JWTベアラートークン認証 - OIDC/OAuth互換のIdP(Cognito, Keycloak, ...)から発行されたアクセストークンを検証することで、安全性を保証する仕組み

今回は、JWTベアラートークンを利用し、以下のような構成をとってみたいと思います。

![architecture](/images/agentcore-with-kong-gateway/architecture.png)

### 1. Cognitoの設定

まずは、Cognitoで`agentcore-m2m-client`というクライアントアプリケーションを作成します。

![create-cognito-user-pool](/images/agentcore-with-kong-gateway/create-cognito-user-pool.png)

作成したクライアントアプリケーションの詳細画面からクライアントID、シークレットを参照できるので、控えておきましょう。

![client-overview](/images/agentcore-with-kong-gateway/client-overview.png)

次にリソースサーバーを追加します。サイドバーの**ブランディング** > **ドメイン**を選択し、**リソースサーバーを作成**をクリックします。

![create-resource-server](/images/agentcore-with-kong-gateway/create-resource-server.png)

リソースサーバーは以下のような情報を入れて作成します。

- リソースサーバー名: `AgentCore Runtime`
- リソースサーバー識別子: `agentcore`
- カスタムスコープ
  - スコープ名: `invoke`
  - 説明: `AgentCore Runtime Invoke`

![resource-server-spec](/images/agentcore-with-kong-gateway/resource-server-spec.png)

作成したカスタムスコープをアプリケーションクライアントに割り当てます。サイドバーから**ブランディング** > **マネージドログイン**を選択し、**ログインページ**タブからマネージドログインページの設定を編集しましょう。

![managed-login-page.png](/images/agentcore-with-kong-gateway/managed-login-page.png)

カスタムスコープとして作成した**agentcore/invoke**を追加してあげます。

![add-custom-scope](/images/agentcore-with-kong-gateway/add-custom-scope.png)

ここまで作成できたら、OAuth 2.0 Client Credentials Grantを使ってアクセストークンが取得可能であることを確認しましょう。

```sh
COGNITO_ENDPOINT=<your-cognito-endpoint>
CLIENT_ID=<your-client-id>
CLIENT_SECRET=<your-client-secret>
SCOPE=agentcore/invoke
curl -X POST $COGNITO_ENDPOINT/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&scope=$SCOPE"
```

実行結果：

```json
{"access_token":"eyJ...","expires_in":3600,"token_type":"Bearer"}
```

### 2. AgentCore Runtimeにエージェントをデプロイする

下記をみながら、Strandsを使って簡単なエージェントを作ってみました。

https://strandsagents.com/docs/user-guide/deploy/deploy_to_bedrock_agentcore/python/#option-a-sdk-integration

```py:agent.py
from strands import Agent
from bedrock_agentcore import BedrockAgentCoreApp

agent = Agent("openai.gpt-oss-120b-1:0")

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke_agent(payload):
    prompt = payload.get("prompt")
    return {"data": agent(prompt).message}

if __name__ == "__main__":
    app.run()
```

これを`bedrock-agentcore-starter-toolkit`を使ってデプロイします。

```sh
agentcore configure
```

基本は、デフォルトで進めていただいて問題ないですが、以下の箇所だけは注意してセットアップしてください。

- Authorization Configuration: デフォルトは、SigV4での認証のため、OAuth Authorizerを明示的に選択する
- OAuth Configuration
  - OAuth Discovery URL: `https://cognito-idp.<region>.amazonaws.com/<user-pool-id>/.well-known/openid-configuration`
  - Client ID: `<your-client-id>`
  - Scopes: `agentcore/invoke`
- Request Header Allowlist: `yes`を選択し、許可するヘッダー名として`Authorization`を指定する

以下、`agentcore configure`中のセッション例：

```sh
Configuring Bedrock AgentCore...

📂 Entrypoint Selection
Specify the entry point (use Tab for autocomplete):
  • File path: weather/agent.py
  • Directory: weather/ (auto-detects main.py, agent.py, app.py)
  • Current directory: press Enter
Entrypoint: agent.py
✓ Using file: agent.py

🏷️  Inferred agent name: agent
Press Enter to use this name, or type a different one (alphanumeric without '-')
Agent name [agent]: sandbox
✓ Using agent name: sandbox

🔍 Detected dependency file: requirements.txt
Press Enter to use this file, or type a different path (use Tab for autocomplete):
Path or Press Enter to use detected dependency file: requirements.txt
✓ Using requirements file: requirements.txt

🚀 Deployment Configuration
Select deployment type:
  1. Direct Code Deploy (recommended) - Python only, no Docker required
  2. Container - For custom runtimes or complex dependencies
Choice [1]: 1

Select Python runtime version:
  1. PYTHON_3_10
  2. PYTHON_3_11
  3. PYTHON_3_12
  4. PYTHON_3_13
Note: Current Python 3.14 not supported, using python3.11
Choice [2]: 2
✓ Deployment type: Direct Code Deploy (python.3.11)

🔐 Execution Role
Press Enter to auto-create execution role, or provide execution role ARN/name to use existing
Execution role ARN/name (or press Enter to auto-create):
✓ Will auto-create execution role

🏗️  S3 Bucket
Press Enter to auto-create S3 bucket, or provide S3 URI/path to use existing
S3 URI/path (or press Enter to auto-create):
✓ Will auto-create S3 bucket

🔐 Authorization Configuration
By default, Bedrock AgentCore uses IAM authorization.
Configure OAuth authorizer instead? (yes/no) [no]: yes

📋 OAuth Configuration
Enter OAuth discovery URL: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_7AU4QMCOc/.well-known/openid-configuration
Enter allowed OAuth client IDs (comma-separated): 5vhr3q1r4ebtslo52qc6rbfou8
Enter allowed OAuth audience (comma-separated):
Enter allowed OAuth allowed scopes (comma-separated): agentcore/invoke
Enter allowed OAuth custom claims as JSON string (comma-separated):
✓ OAuth authorizer configuration created

🔒 Request Header Allowlist
Configure which request headers are allowed to pass through to your agent.
Common headers: Authorization, X-Amzn-Bedrock-AgentCore-Runtime-Custom-*
Configure request header allowlist? (yes/no) [no]: yes

📋 Request Header Allowlist Configuration
Enter allowed request headers (comma-separated) [Authorization,X-Amzn-Bedrock-AgentCore-Runtime-Custom-*]: Authorization
✓ Request header allowlist configured with 1 headers
Configuring BedrockAgentCore agent: sandbox

Memory Configuration
Tip: Use --disable-memory flag to skip memory entirely

No existing memory resources found in your account

Options:
  • Press Enter to create new memory
  • Type 's' to skip memory setup

Your choice: s
✓ Skipping memory configuration
Memory disabled by user choice
Network mode: PUBLIC
Setting 'sandbox' as default agent
╭────────────────────────────────────────────────────────────────────────────────────────────────── Configuration Success ───────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Agent Details                                                                                                                                                                                                              │
│ Agent Name: sandbox                                                                                                                                                                                                        │
│ Deployment: direct_code_deploy                                                                                                                                                                                             │
│ Region: us-east-1                                                                                                                                                                                                          │
│ Account: 962309199027                                                                                                                                                                                                      │
│ Runtime: python3.11                                                                                                                                                                                                        │
│                                                                                                                                                                                                                            │
│ Configuration                                                                                                                                                                                                              │
│ Execution Role: Auto-create                                                                                                                                                                                                │
│ Network Mode: Public                                                                                                                                                                                                       │
│ S3 Bucket: Auto-create                                                                                                                                                                                                     │
│ Authorization: OAuth (customJWTAuthorizer)                                                                                                                                                                                 │
│                                                                                                                                                                                                                            │
│ Request Headers Allowlist: 2 headers configured                                                                                                                                                                            │
│                                                                                                                                                                                                                            │
│ Memory: Disabled                                                                                                                                                                                                           │
│                                                                                                                                                                                                                            │
│                                                                                                                                                                                                                            │
│ 📄 Config saved to: /Users/shuhei.kawamura/sandbox/agent-core/.bedrock_agentcore.yaml                                                                                                                                      │
│                                                                                                                                                                                                                            │
│ Next Steps:                                                                                                                                                                                                                │
│ agentcore deploy                                                                                                                                                                                                           │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

設定が完了すると、`.bedrock_agentcore.yaml`という設定ファイルが出力されるので、これを元にデプロイします。

```sh
agentcore deploy
```

`agentcore`のサブコマンドにinvokeコマンドが提供されているので、これを使って実行してみましょう。

```sh
agentcore invoke '{"prompt": "Kong Gatewayって何？"}'
```

実行結果：

```sh
Warning: OAuth is configured but no bearer token provided
╭───────────────────────────────────────────────────────────────────────────────────────────────────────── sandbox ──────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Logs: aws logs tail /aws/bedrock-agentcore/runtimes/sandbox-j1oXR2FMAu-DEFAULT --log-stream-name-prefix "2026/04/24/[runtime-logs" --follow                                                                                │
│       aws logs tail /aws/bedrock-agentcore/runtimes/sandbox-j1oXR2FMAu-DEFAULT --log-stream-name-prefix "2026/04/24/[runtime-logs" --since 1h                                                                              │
│ GenAI Dashboard: https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#gen-ai-observability/agent-core                                                                                                           │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

Invocation failed: An error occurred (AccessDeniedException) when calling the InvokeAgentRuntime operation: Authorization method mismatch. The agent is configured for a different authorization method than what was used in 
your request. Check the agent's authorization configuration and ensure your request uses the matching method (OAuth or SigV4)
Your AWS credentials or bearer token may be expired. Please re-login to get a new auth token.
```

実行に必要なJWTベアラートークンが存在しないため、実行に失敗することが確認できます。そのため、実行に必要なアクセストークンを再度取得し、パラメータに設定しましょう。

```sh
COGNITO_ENDPOINT=<your-cognito-endpoint>
CLIENT_ID=<your-client-id>
CLIENT_SECRET=<your-client-secret>
SCOPE=agentcore/invoke
curl -X POST $COGNITO_ENDPOINT/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&scope=$SCOPE"
```

アクセストークンを含めた実行例：

```sh
AT=eyJ...
agentcore invoke '{"prompt": "Kong Gatewayってなんですか？"}' --bearer-token $AT
```

実行結果：

```sh
Using bearer token for OAuth authentication
Using JWT authentication
╭───────────────────────────────────────────────────── sandbox ──────────────────────────────────────────────────────╮
│ Session: 149efd6e-8260-4e6e-b9c2-1b2b320897a4                                                                      │
│ ARN: arn:aws:bedrock-agentcore:us-east-1:962309199027:runtime/sandbox-j1oXR2FMAu                                   │
│ Logs: aws logs tail /aws/bedrock-agentcore/runtimes/sandbox-j1oXR2FMAu-DEFAULT --log-stream-name-prefix            │
│ "2026/04/24/[runtime-logs" --follow                                                                                │
│       aws logs tail /aws/bedrock-agentcore/runtimes/sandbox-j1oXR2FMAu-DEFAULT --log-stream-name-prefix            │
│ "2026/04/24/[runtime-logs" --since 1h                                                                              │
│ GenAI Dashboard: https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#gen-ai-observability/agent-core   │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

Response:
{"data": {"role": "assistant", "content": [{"reasoningContent": {"reasoningText": {"text": "The user asks in Japanese:
\"Kong Gatewayってなんですか？\" means \"What is Kong Gateway?\" So need to explain what Kong Gateway is. Provide
answer in Japanese, explain that it's an API gateway, open-source, built on Nginx, features, architecture, plugins,
enterprise version, use cases, benefits, etc.\n\nGive concise but thorough explanation.\n\nProbably respond in
Japanese."}}}, {"text": "**Kong Gateway（Kong API
Gateway）**は、マイクロサービスやコンテナ化されたアプリケーション向けに設計された、**高性能・拡張性のあるオープンソー
スの API ゲートウェイ**です。  \n元は Nginx の上に構築されたプラグインベースのプロキシで、現在は Go と
Lua（OpenResty）を組み合わせたハイブリッドアーキテクチャに進化しています。\n\n---\n\n## 主な役割・機能\n\n| カテゴリ |
具体的な機能 |\n|---|---|\n| **トラフィック制御** |
ルーティング（パス、ホスト、ヘッダー、メソッド等）、ロードバランシング、リクエスト/レスポンス変換 |\n|
**セキュリティ** | 認証・認可（APIキー、OAuth2、JWT、Basic Auth、LDAP など）、TLS 終端、IP
フィルタリング、レートリミット、WAF（Enterprise版） |\n| **可観測性** |
ロギング、メトリクス収集（Prometheus、Datadog、AWS CloudWatch など）、トレーシング（Jaeger、Zipkin） |\n|
**プラグインエコシステム** | 公式・サードパーティ製プラグインが 100 以上。カスタムプラグインは Lua/Go で簡単に作成可能
|\n| **マルチテナンシー** |
サービス・ルート・プラグインを「Workspaces」や「Consumers」ごとに分離でき、複数チームや顧客を安全に共存させる |\n|
**管理・運用** | Admin API、Kong Manager（Web UI）、Kong Control Plane / Data Plane（Kong Mesh/Service Mesh 向け）
|\n| **ハイブリッド・クラウド対応** | オンプレミス、Kubernetes、Docker、AWS/GCP/Azure の各マネージド環境でデプロイ可能
|\n| **スケーラビリティ** | 水平スケールが容易で、Kong Cluster モードや Kubernetes Ingress Controller
としての利用で数千ノード規模も実現可能 |\n\n---\n\n## アーキテクチャの概要\n\n1. **Kong のコア** –
Nginx（OpenResty）上に Lua スクリプトで構築されたプロキシ。リクエストはまず Nginx が受け取り、Lua
がプラグインチェーンを実行して処理を行う。  \n2. **データストア** – 設定情報は **PostgreSQL** または **Cassandra**
に永続化。Kong の Admin API がこのデータストアとやり取りし、設定の追加・変更を行う。  \n3. **プラグインシステム** –
プラグインは **Lua**（従来）または
**Go**（新しいプラグインフレームワーク）で記述でき、リクエスト/レスポンスの任意の段階でフックできる。  \n4.
**Kubernetes 連携** – `Kong Ingress Controller` としてデプロイすれば、Kubernetes の Ingress リソースを自動的に Kong
のルーティングやプラグイン設定に変換できる。\n\n---\n\n## エディション\n\n| エディション | 特徴 | 主な利用シーン
|\n|---|---|---|\n| **Kong Gateway (OSS)** | 完全オープンソース、基本的な API
ゲートウェイ機能と主要プラグインが利用可能 | 小規模チーム、PoC、スタートアップ |\n| **Kong Enterprise** |
商用サポート、RBAC、監査ログ、企業向けプラグイン（例: 高度なレートリミット、WAF、認証連携）・GUI（Kong
Manager）・Hybrid モード | 大規模組織、ミッションクリティカルな API、マルチテナント環境 |\n| **Kong Mesh** | Service
Mesh 用のデータプレーン（Envoy）とコントロールプレーンを提供。Kong Gateway の機能と Mesh の機能をシームレスに統合 |
マイクロサービス間通信の可視化・制御が必要なケース |\n\n---\n\n## 代表的な利用例\n\n| シナリオ | 具体的な活用イメージ
|\n|---|---|\n| **外部 API 公開** | 開発者ポータルと連携し、API キーや OAuth2 認証を付与した公開 API を安全に提供 |\n|
**マイクロサービス間の仲介** | 各サービスは内部のプライベートネットワークにだけ存在し、Kong
がトラフィックのロードバランシング、認証、レートリミットを一手に担う |\n| **レガシーアプリのモダナイゼーション** |
既存の monolith アプリに対してリクエスト変換やレスポンスキャッシュを実装し、段階的に API 化 |\n| **マルチテナント
SaaS** | テナントごとに Workspaces を分け、プラグイン設定やレートリミットをテナント単位で管理 |\n| **Kubernetes
環境での Ingress** | Kong Ingress Controller が Ingress リソースを自動で Kong
ルーティングに変換し、同時に認証・モニタリングも実装 |\n\n---\n\n## なぜ「Kong」なのか（選択理由）\n\n1.
**パフォーマンス** – Nginx の高速プロキシ機能と Lua/Go の軽量プラグインで、ミリ秒単位のレイテンシが実現できる。  \n2.
**拡張性** –
公式プラグインだけでなく、ユーザー独自のプラグインを簡単に追加でき、ビジネスロジックに合わせたカスタマイズが可能。
\n3. **オープンソース・ベンダーロックイン回避** – OSS コミュニティが活発で、ライセンスは
Apache‑2.0。ロックインの懸念が低い。  \n4. **マルチクラウド・ハイブリッド対応** –
どんなインフラでも同じ設定で動作させられるので、クラウドベンダーの変更やハイブリッド構成が容易。  \n5.
**エコシステム** – Kong Manager、Kong Konnect（SaaS版）や Kong Mesh など、API
管理・サービスメッシュまでカバーできる一連のツールが揃っている。\n\n---\n\n## 簡単な導入例（Docker）\n\n```bash\n#
Docker で Kong OSS と PostgreSQL を起動\ndocker network create kong-net\n\ndocker run -d --name kong-database \\\n
--network=kong-net \\\n  -p 5432:5432 \\\n  -e \"POSTGRES_USER=kong\" \\\n  -e \"POSTGRES_DB=kong\" \\\n  -e
\"POSTGRES_PASSWORD=kong\" \\\n  postgres:13\n\n# DB 初期化\ndocker run --rm \\\n  --network=kong-net \\\n  -e
\"KONG_DATABASE=postgres\" \\\n  -e \"KONG_PG_HOST=kong-database\" \\\n  -e \"KONG_PG_PASSWORD=kong\" \\\n
kong/kong:3.5.0 kong migrations bootstrap\n\n# Kong 本体起動\ndocker run -d --name kong \\\n  --network=kong-net \\\n
-e \"KONG_DATABASE=postgres\" \\\n  -e \"KONG_PG_HOST=kong-database\" \\\n  -e \"KONG_PG_PASSWORD=kong\" \\\n  -e
\"KONG_PROXY_ACCESS_LOG=/dev/stdout\" \\\n  -e \"KONG_ADMIN_ACCESS_LOG=/dev/stdout\" \\\n  -e
\"KONG_PROXY_ERROR_LOG=/dev/stderr\" \\\n  -e \"KONG_ADMIN_ERROR_LOG=/dev/stderr\" \\\n  -p 8000:8000 -p 8443:8443 -p
8001:8001 -p 8444:8444 \\\n  kong/kong:3.5.0\n```\n\n起動後、`http://localhost:8001` が **Admin
API**、`http://localhost:8000` が **プロキシエンドポイント**です。  \nここにサービスやルート、プラグインを `curl`
で登録すれば、すぐに API が利用できるようになります。\n\n---\n\n## まとめ\n\n- **Kong Gateway** は、API の **入口**
を統一的に管理し、**認証・認可・トラフィック制御・可観測性** をプラグインで提供する API ゲートウェイです。  \n-
**OSS版** と **Enterprise版** があり、規模や要件に合わせて選択可能。  \n- **Kubernetes**
ともシームレスに連携でき、マイクロサービス・クラウドネイティブ環境で広く採用されています。  \n\nAPI
を安全・高速に公開・管理したい組織にとって、Kong は「全ての API
の入口を一元化する」ための実績あるプラットフォームと言えるでしょう。"}], "metadata": {"usage": {"inputTokens": 75,
"outputTokens": 2235, "totalTokens": 2310}, "metrics": {"latencyMs": 8525, "timeToFirstByteMs": 462}}}}
```

### 3. Kong Gatewayを通した実行例

最後に実行した通り、エージェントのクライアントは適切なOAuthのフローを用いてアクセストークンを取得したのち、それを実行時に渡してあげる必要があります。Kong Gatewayでは、[Upstream OAuth](https://developer.konghq.com/plugins/upstream-oauth/)というプラグインが先ほどのやりとりを代替してくれます。つまり、エージェントクライアントの代わりにアクセストークンをCognitoから取得し、それをリクエストヘッダーに含めてUpstreamへ転送してくれる動きをします。

それを実現するためのKong Gatewayの設定例は以下の通りです。decKを使っていますが、もちろんGUIからでも同じように設定ができます。
実行には、`DECK_TOKEN_ENDPOINT`, `DECK_CLIENT_ID`, `DECK_CLIENT_SECRET`を環境変数として設定してください。

```yaml
_format_version: "3.0"

services:
  - name: agentcore-service
    # URLフィールドを使うとURLエンコードしたものがdecKによりデコードされてしまうので、
    # protocol + host + port + pathでURLを構成する
    protocol: https
    host: bedrock-agentcore.us-east-1.amazonaws.com
    port: 443
    path: /runtimes/arn%3Aaws%3Abedrock-agentcore%3Aus-east-1%3A962309199027%3Aruntime%2Fsandbox-j1oXR2FMAu/invocations
    routes:
      - name: agentcore-route
        protocols:
          - http
          - https
        paths:
          - /agentcore/invoke
        strip_path: true

plugins:
  - name: upstream-oauth
    service: agentcore-service
    config:
      client:
        ssl_verify: true
      oauth:
        token_endpoint: ${{ env "DECK_TOKEN_ENDPOINT" }}
        grant_type: client_credentials
        client_id: ${{ env "DECK_CLIENT_ID" }}
        client_secret: ${{ env "DECK_CLIENT_SECRET" }}
        scopes:
          - agentcore/invoke
```

いくつかポイントがあります。一つ目は、serviceリソースの指定方法です。転送先の指定時は、上記の例のように `protocol`, `host`, `port`, `path`の組み合わせで転送先を指定する方法と `url`というパラメータでまとめて指定する方法があります。しかし、decKを使ってURLエンコードしたARNをパスに含めると、Kong Gatewayへの適用時に自動的にデコードしてしまうようで、うまく転送ができなかったため、`protocol`, `host`, `port`, `path`の組み合わせで指定をしています。尚、AgentCoreを実行する際のURL形式については以下ドキュメントに記載があります。

https://docs.aws.amazon.com/ja_jp/bedrock-agentcore/latest/devguide/runtime-oauth.html

続いて、`upstream-oauth`プラグインについてです。ここでは、AgentCoreへアクセスする前にCognitoから提供されているトークンエンドポイントを実行し、アクセストークンを取得するのに必要な情報を記載しています。

```yaml
plugins:
  - name: upstream-oauth
    service: agentcore-service
    config:
      client:
        ssl_verify: true
      oauth:
        token_endpoint: ${{ env "DECK_TOKEN_ENDPOINT" }}
        grant_type: client_credentials
        client_id: ${{ env "DECK_CLIENT_ID" }}
        client_secret: ${{ env "DECK_CLIENT_SECRET" }}
        scopes:
          - agentcore/invoke
```

これで一通りの構成が作り終えたので、実際にKong Gatewayを通してAgentCore Runtimeを呼び出してみましょう。（クライアントからは、何の認証情報も送っていないことに注目してください。）

```sh
curl --request POST \
  --url http://localhost:8000/agentcore/invoke \
  --header 'content-type: application/json' \
  --data '{"prompt": "Kong Gatewayってなんですか？"}'
```

実行結果：

```sh
{"data": {"role": "assistant", "content": [{"reasoningContent": {"reasoningText": {"text": "The user asks in Japanese: \"Kong Gatewayってなんですか？\" They want to know what Kong Gateway is. Provide a clear explanation in Japanese, covering what it is, its features, architecture, use cases, maybe differences between open-source and enterprise, and possibly some examples. Should be concise but thorough. Use Japanese"}}}, {"text": "**Kong Gateway（コング・ゲートウェイ）**とは、マイクロサービスや API（Application Programming Interface）に対して、**統合的なトラフィック管理・セキュリティ・オーケストレーション機能を提供するオープンソースの API ゲートウェイ**です。もともとは NGINX をベースにした軽量・高速なプロキシとして開発され、現在は **Kong Inc.** がメンテナンス・商用サポートを行っています。\n\n---\n\n## 主な役割と特徴\n\n| カテゴリ | 内容 |\n|----------|------|\n| **トラフィックルーティング** | リクエストをサービスごとにルーティング（パス、ホスト、ヘッダー、メソッドなどの条件） |\n| **認証・認可** | API キー、OAuth 2.0、JWT、Basic 認証、OpenID Connect など多彩な認証プラグイン |\n| **セキュリティ** | レートリミット、IP フィルタリング、TLS 終端、WAF（Enterprise版） |\n| **トラフィック制御** | レートリミット、クォータ、リトライ、タイムアウト、バックエンドのヘルスチェック |\n| **プラグインエコシステム** | 公式プラグイン 50 種以上＋自作プラグイン（Lua、Go、JavaScript） |\n| **モニタリング・ロギング** | Prometheus メトリクス、Grafana ダッシュボード、ELK/Datadog 連携 |\n| **サービスディスカバリ** | Consul、Kubernetes、DNS などと連携し、動的にバックエンドを検出 |\n| **マルチプロトコル** | HTTP/1.1、HTTP/2、gRPC、WebSocket、TCP/UDP（Enterprise Plugin） |\n| **スケーラビリティ** | クラスタリング（DB‑less モード、PostgreSQL/MySQL バックエンド）で水平スケールが可能 |\n| **デプロイ柔軟性** | Docker、Kubernetes、Helm、Bare‑metal、VM いずれでも動作 |\n\n---\n\n## アーキテクチャの概要\n\n```\n[クライアント] → [Kong Proxy (Ingress) ] → 【プラグイン処理】 → [バックエンドサービス]\n```\n\n1. **Kong Proxy (Ingress)**  \n   * クライアントからのリクエストを受け取り、NGINX/LuaJIT の高速プロキシで処理。  \n2. **プラグインチェーン**  \n   * 認証、レートリミット、変換、ロギングなどのプラグインが順次実行されます。  \n3. **サービス・ルート**  \n   * Kong の「Service」（バックエンド）と「Route」（外部からのパス）を組み合わせて、リクエストを適切なマイクロサービスへ転送。  \n4. **DB‑less / DB モード**  \n   * **DB‑less**：設定を YAML/JSON でロードし、Kong 自体が状態を保持しない。  \n   * **DB モード**：PostgreSQL または MySQL に設定・メタデータを永続化し、クラスタリング時に共有。  \n\n---\n\n## エディション\n\n| エディション | 主な違い |\n|--------------|----------|\n| **Kong Gateway (Open Source)** | 基本的なプロキシ・ルーティング、認証、レートリミット、ログ、Prometheus 等。 |\n| **Kong Enterprise** | 追加のエンタープライズ機能：Dev Portal、RBAC、監査ログ、サービスメッシュ (Kong Mesh) 連携、GUI 管理コンソール、カスタムプラグインのサポート、SLAs とサポートサービス。 |\n| **Kong Mesh** | サービスメッシュとしての機能（Istio 代替）を提供。Kong Gateway と組み合わせて API とサービスメッシュを統一的に管理できる。 |\n\n---\n\n## 典型的な利用シーン\n\n1. **マイクロサービスアーキテクチャの入口**  \n   * すべての内部サービスへの入り口として、認証・レートリミットを一元管理。  \n2. **外部パートナー向け API の公開**  \n   * API キーや OAuth2.0 で外部開発者に安全に提供。  \n3. **レガシーシステムのラッピング**  \n   * 既存のモノリシック API を Kong のプラグインでモダン化（例：レスポンス変換、CORS 設定）。  \n4. **マルチテナンシー & SaaS**  \n   * テナントごとに Route とプラグインを分割し、課金や使用量制御を実装。  \n5. **Kubernetes 環境の Ingress**  \n   * `Kong Ingress Controller` を使って、K8s の Ingress リソースを Kong が自動的に設定。  \n\n---\n\n## 簡単な導入例（Docker）\n\n```bash\n# 1. Kong を DB-less モードで起動\ndocker run -d --name kong \\\n  -e KONG_DATABASE=off \\\n  -e KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative/kong.yml \\\n  -p 8000:8000 -p 8443:8443 -p 8001:8001 -p 8444:8444 \\\n  kong:3.5\n\n# 2. サービスとルートを設定（Kong Admin API）\ncurl -i -X POST http://localhost:8001/services \\\n  --data name=demo-service \\\n  --data url='http://httpbin.org'\n\ncurl -i -X POST http://localhost:8001/services/demo-service/routes \\\n  --data 'paths[]=/demo'\n```\n\n上記だけで `http://localhost:8000/demo` にアクセスすると、`httpbin.org` のエンドポイントへリバースプロキシされます。プラグインを追加すれば認証やレートリミットがすぐに有効になります。\n\n---\n\n## まとめ\n\n- **Kong Gateway** は、API・マイクロサービスの「入り口」に特化した高速・拡張性の高いゲートウェイです。  \n- **プラグインベース** の設計により、認証・セキュリティ・モニタリング・トラフィック制御をコードや設定だけで柔軟に組み合わせられます。  \n- **オープンソース版** と **Enterprise 版** があり、どちらもクラウド・オンプレミス・Kubernetes など多様な環境で動作します。  \n- 大規模なトラフィックやマルチテナント環境でも **水平スケール** が可能で、企業レベルの API 管理基盤として広く採用されています。\n\nもし具体的な導入シナリオやプラグイン実装例、Kubernetes での設定手順など、さらに掘り下げた情報が必要であれば遠慮なく質問してください！"}], "metadata": {"usage": {"inputTokens": 75, "outputTokens": 1765, "totalTokens": 1840}, "metrics": {"latencyMs": 7203, "timeToFirstByteMs": 450}}}}
```

トレースを見てみると、確かにUpstream OAuthプラグインがCognitoから提供されているトークンエンドポイントを実行していることが確認できます。

![upstream-oauth](/images/agentcore-with-kong-gateway/upstream-oauth.png)

## 終わりに

今回は、Upstream OAuthプラグインを使ってAmazon Bedrock AgentCore RuntimeにデプロイしたエージェントをJWTベアラートークン認証で実行する方法を紹介しました。これに限らず、Upstreamのさまざまな認証方法をKong Gatewayで吸収し、（今回はやりませんでしたが、）Gatewayとして単一の認証方式を提供するというのは、クライアントの実装負荷を減らすという意味でとても大きな意味があります。
