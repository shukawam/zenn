---
title: "Volcano SDK で始める AI Agent"
emoji: "🌋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["agent", "volcano", "kong"]
published: true
---

## はじめに

こんにちは🖐️ この記事は、[Kong Advent Calendar 2025](https://qiita.com/advent-calendar/2025/kong) の Day1 として書かれています。初日なのに Kong Gateway の話じゃないのが若干心苦しいですが、気楽に読んでいただければと🙏 今回は、Kong 社の年次イベントである API Summit で発表された [Volcano SDK](https://github.com/Kong/volcano-sdk) の代表パターンを触ってみたいと思います。

## Volcano SDK 🌋

Volcano SDK は、AI Agent を構築するための TypeScript ライブラリです。Kong では既に多くの AI エージェントを作っていますが、その際に既存のフレームワークやライブラリでは、抽象度が高すぎたり、逆に原始的すぎたりしたので、Kong にとってちょうど良い物を作った結果が Volcano SDK というわけです[^1]。2025/10 月に OSS 化されているので、執筆時点ではかなり後発なライブラリとなります。一通り触った個人の感想としては、

- シンプルなコードで AI Agent の構築ができる
- 良くも悪くも MCP First な作りになっている
- オブザーバビリティ関連機能がビルトインされているのが良い
- ビルトインされているリトライ戦略が充実しているのは、API Gateway ベンダーっぽさを感じる

と言ったところでしょうか。

[^1]: 開発の経緯などは、この辺を読むと原文が読めます - [https://konghq.com/blog/product-releases/volcano](https://konghq.com/blog/product-releases/volcano)

## 試してみよう

今回の検証に使ったコードは、以下のリポジトリの `/src/*` に格納しています。

https://github.com/shukawam/volcano-sandbox

`docker compose up -d` で環境を起動してもらうとオブザーバビリティ関連コンポーネント（Grafana, Prometheus, Grafana Loki, Grafana Tempo, ...）が起動するため、エージェントの振る舞いを観測したい方はぜひ試してみてください。また、それぞれのコードは、`npx tsx src/<指定ファイル名>` で実行が可能で、実行には OpenAI の API Key が必要になるため、環境変数に設定をしておいてください。

```sh
export OPENAI_API_KEY=...
```

### 最もシンプルな実行パターン

まずは、シンプルに LLM API を呼ぶだけのパターンを実装してみます。

```ts:simple-llm.ts
import { agent, llmOpenAI } from "volcano-sdk";

// ... 1
const llm = llmOpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
  model: "gpt-4o-mini",
});

// ... 2
const results = await agent({
  llm,
})
  .then({ prompt: "Volcano SDKとはなんですか？" })
  .run();

console.log(results[0]?.llmOutput);
```

上記のように実装します。まず、利用する LLM を宣言します。

```ts
const llm = llmOpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
  model: "gpt-4o-mini",
});
```

ここでは最低限のパラメータしか渡していないですが、`llmOpenAI` には `OpenAIConfig` を渡す必要があり以下のような作りになっています。

```ts
export type OpenAIConfig = {
  apiKey: string;
  model: string;
  baseURL?: string;
  options?: OpenAIOptions;
};
```

API リクエストの送信先も修正することができるので、例えばローカルに Kong Gateway を構築し、LLM 呼び出しをプロキシする場合は以下のように設定すれば良いです。

```ts
// Kong GatewayをプロキシしてLLM(OpenAI)を呼び出す場合
const llm = llmOpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
  model: "gpt-4o-mini",
  baseURL: "http://localhost:8000/openai", // Kong GatewayのRouteの設定によりますが、一例として
});
```

また、`options` として渡せるプロパティは以下のようになっています。このあたりは、OpenAI を普段使っている方であればお馴染みのパラメータだと思います。

```ts
export type OpenAIOptions = {
  temperature?: number;
  max_tokens?: number;
  max_completion_tokens?: number;
  top_p?: number;
  frequency_penalty?: number;
  presence_penalty?: number;
  stop?: string | string[];
  seed?: number;
  response_format?: {
    type: "json_object" | "text";
  };
  n?: number;
  logit_bias?: Record<string, number>;
  user?: string;
};
```

### OpenTelemetry を使ったテレメトリーシグナルの転送

Volcano SDK は、OpenTelemetry を使ってテレメトリー（メトリクス、トレース）をエクスポートするための仕組みが実装されています。ドキュメントは、以下に記載があるのですが、より簡単に試すには本記事で記載されている内容で十分かと思いますので、参考にしてください。

https://volcano.dev/docs/observability#opentelemetry-integration

まず、OpenTelemetry 関連の依存はデフォルトでは含まれていないので、自分で含めてあげる必要があります。

```sh
npm install @opentelemetry/api
npm install @opentelemetry/sdk-node
npm install @opentelemetry/exporter-otlp-http
```

最低限の動作プログラムは以下のようになります。

```ts:simple-llm-observability.ts
import { agent, llmOpenAI, createVolcanoTelemetry } from "volcano-sdk";

const serviceName = process.env.OTEL_SERVICE_NAME || "volcano-sandbox";
const otlpEndpoint =
  process.env.OTEL_EXPORTER_OTLP_ENDPOINT || "http://localhost:4318";

// ... 1
const telemetry = createVolcanoTelemetry({
  serviceName: serviceName,
  endpoint: otlpEndpoint,
  traces: true,
  metrics: true,
});

const llm = llmOpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
  model: "gpt-4o-mini",
});

// ... 3
try {
  // ... 2
  const results = await agent({
    llm,
    telemetry, // Enable observability feature.
  })
    .then({ prompt: "Volcano SDKとはなんですか？" })
    .run();
  console.log(`[LLM Output]: ${results[0]?.llmOutput}`);
} catch (error) {
  throw new Error("The agent returned no results.");
}
```

まずは、Volcano SDK から提供されている `createVolcanoTelemetry` 関数を利用し、OpenTelemetry との統合を設定します。

```ts
const telemetry = createVolcanoTelemetry({
  serviceName: serviceName,
  endpoint: otlpEndpoint,
  traces: true,
  metrics: true,
});
```

`traces`, `metrics` はデフォルトで `true` なので、ここで明示的に有効にしなくても実は良いのですが、わかりやすさのために明示的に有効化しています。また、[ドキュメント](https://volcano.dev/docs/observability#opentelemetry-integration)には、OpenTelemetry の NodeSDK をセットアップする手順が記載されていますが、`createVolcanoTelemetry` で `endpoint` のパラメータが設定されている場合は、ライブラリ中で NodeSDK のセットアップが自動的に行われる[^2]ので、この手順は不要になります。

[^2]: https://github.com/Kong/volcano-sdk/blob/main/src/telemetry.ts#L69-L105

また、[https://github.com/Kong/volcano-sdk/blob/main/src/telemetry.ts#L159-L297](https://github.com/Kong/volcano-sdk/blob/main/src/telemetry.ts#L159-L297) に記載されているようにライブラリ中で自動的に計装されるスパンは、以下のようになっています。

- `agent.span`: エージェントの実行スパンを示す
- `step.execute`: エージェント内の各実行ステップ（`.then() で連結している 1 ブロック`）の実行スパンを示す
- `llm.generate`: LLM の呼び出しスパンを示す
- `mcp.*`: MCP の呼び出しスパンを示す

このセットアップが完了した上で、エージェントの定義時に `createVolcanoTelemetry` の戻り値をプロパティとして追加すると各種テレメトリーが自動的に記録され、エクスポートされます。

```ts
const results = await agent({
  llm,
  telemetry, // Enable observability feature.
});
```

また、try-catch でエージェントの実行ブロックを囲っておくとエラーが発生した際にスパンにその詳細を記録してくれるため、try-catch の実装を追加しています。

```ts
// ... 3
try {
  // ... 2
  const results = await agent({
    llm,
    telemetry, // Enable observability feature.
  })
    .then({ prompt: "Volcano SDKとはなんですか？" })
    .run();
  console.log(`[LLM Output]: ${results[0]?.llmOutput}`);
} catch (error) {
  throw new Error("The agent returned no results.");
}
```

Volcano SDK のリポジトリ内には、Volcano SDK が出力するメトリクスを可視化するための Grafana Dashboard も含まれているのでうまく活用すると良いでしょう。

https://github.com/Kong/volcano-sdk/blob/main/observability-demo/grafana-volcano-dashboard.json

OpenTelemetry Collector の Pipeline 定義などを正しく設定し、エクスポートされると以下のように可視化することができます。

![traces](/images/ai-agent-with-kong-volcano/traces.png)
_Grafana Tempo で可視化された Volcano SDK のトレース_

![metrics](/images/ai-agent-with-kong-volcano/metrics-dashboard.png)
_Grafana Dashboard で可視化された Volcano SDK のメトリクス_

### MCP を使った LLM の拡張

近年では、AI Agent が使うツールとの連携方法を定義した MCP(Model Context Protocol)が事実上の標準として扱われています。MCP とは、LLM が自身が有する知識を超えて、外部のシステムやツールと双方向通信し、アクションを実行するための標準化されたプロトコルです。Volcano SDK もその流れを汲んで MCP First な作りになっています。MCP を活用した実装は以下の通りです。

```ts:mcp.ts
import { agent, llmOpenAI, mcp, createVolcanoTelemetry } from "volcano-sdk";

const serviceName = process.env.OTEL_SERVICE_NAME || "volcano-sandbox";
const otlpEndpoint =
  process.env.OTEL_EXPORTER_OTLP_ENDPOINT || "http://localhost:4318";

const telemetry = createVolcanoTelemetry({
  serviceName: serviceName,
  endpoint: otlpEndpoint,
  traces: true,
  metrics: true,
});

const llm = llmOpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
  model: "gpt-4o-mini",
});

// ... 1
const revenueTool = mcp("http://localhost:8080/mcp");

const steps = await agent({ llm, telemetry })
  .then({
    prompt: "売上のTop3を教えてください。",
    // ... 2
    mcps: [revenueTool], // Automatic tool selection
  })
  .run();

steps.forEach((step) => {
  console.log(`[Tool call] ${step.toolCalls}`);
  console.log(`[LLM output] ${step.llmOutput}`);
});
```

まずは、AI Agent として利用する MCP Server を登録します。

```ts
const revenueTool = mcp("http://localhost:8080/mcp");
```

`mcp` 関数のシグネチャとしては以下のようになっており、`url` の他に `MCPAuthConfig` が指定可能です。

```ts
export declare function mcp(
  url: string,
  options?: {
    auth?: MCPAuthConfig;
  }
): MCPHandle;
```

`MCPAuthConfig` の型定義は、以下のようになっており、Bearer トークンと OAuth 2.1[^3] による認可がサポートされています。

[^3]: [https://volcano.dev/docs/mcp-tools#mcp-authentication](https://volcano.dev/docs/mcp-tools#mcp-authentication) に記載があるので、OAuth 2.1 と記述していますが、実装的には OAuth 2.0 相当です。

```ts
export type MCPAuthConfig = {
  type: "oauth" | "bearer";
  token?: string;
  clientId?: string;
  clientSecret?: string;
  tokenEndpoint?: string;
  scope?: string;
  refreshToken?: string;
};
```

ちなみに、`STDIO` ベースの MCP Server と通信したい場合には以下のように実装すれば良いです。

```ts
const revenueTool = mcpStdio({
  command: "tsx",
  args: ["mcp/revenue-tool.ts"],
});
```

### マルチエージェントの実行パターン

Volcano SDK では、複数のエージェントが協調してタスクを解くマルチエージェントの実行パターンもサポートされています。以下のように実装します。

```ts:multi-agent.ts
import { agent, llmOpenAI } from "volcano-sdk";

const llm = llmOpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
  model: "gpt-4o-mini",
});

const planner = agent({
  name: "planner",
  description: "旅行のプランを計画するエージェントです。",
  instructions:
    "旅行プランを計画します。特に指示がない限り、比較検討ができるように3つのプランを提案してください。",
  llm,
});

const comparator = agent({
  name: "comparator",
  description: "プランの比較をするエージェントです。",
  instructions:
    "旅行プランの比較をします。比較軸は、「行動のしやすさ」、「価格の安さ」、「混雑度合い」です。",
});

const summarizer = agent({
  name: "summarizer",
  description: "プランの要約を行うエージェントです。",
  instructions:
    "旅行プランの要約をします。比較軸も踏まえてユーザーにわかりやすい形で要約してください。",
});

const result = await agent({
  llm,
  timeout: 90,
})
  .then({
    name: "travel-agent",
    prompt: "東京で2泊3日の旅行がしたいです。",
    agents: [planner, comparator],
    maxAgentIterations: 3,
  })
  .run();

const recommendPlan = await result.ask(
  llm,
  "要約されたプラン内容を提示してください。"
);
const agentUsed = await result.ask(
  llm,
  "どのエージェントがどのように使われたか示してください。"
);

console.log("\n" + recommendPlan);
console.log("\n" + agentUsed);

// プロセスを強制的に終了させないとLLMの入力を待ち続けてしまう。
process.exit(0);
```

実行結果を見てもらうと、`Coordinator` が与えられたプロンプトに対してどのエージェントを利用すれば良いかを判断し、適切にエージェントを呼び出していることがわかります。

```sh
🌋 Running Volcano agent [volcano-sdk v1.0.1] • docs at https://volcano.dev
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🤖 Step 1/1: 東京で2泊3日の旅行がしたいです。

🧠 Coordinator decision: USE planner
   ✅ Complete | 36 tokens | 0 tool calls | 1.2s | OpenAI-gpt-4o-mini

⚡ planner → 東京での2泊3日の旅行プランを計画してください。旅行の日程や観光地、おすすめの宿泊先を提案してくださ...
   ✅ Complete | 805 tokens | 0 tool calls | 16.1s | OpenAI-gpt-4o-mini

🧠 Coordinator decision: USE comparator
   ✅ Complete | 18 tokens | 0 tool calls | 2.1s | OpenAI-gpt-4o-mini

⚡ comparator → Compare the three travel plans for Tokyo, highligh...
🧠 Coordinator decision: USE summarizer
   ✅ Complete | 15 tokens | 0 tool calls | 0.9s | OpenAI-gpt-4o-mini

⚡ summarizer → 各プランの要約を作成してください。...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎉 Agent complete | 874 tokens | 0 tool calls | 20.3s | OpenAI-gpt-4o-mini
   ⏳ Waiting for LLM | 16.2s
以下は、東京での2泊3日の旅行プランの要約です。

### プラン1: 文化と歴史を楽しむ旅行
- **日程**
  - 1日目: 上野・浅草エリア（上野恩賜公園、浅草寺）
  - 2日目: 皇居・新宿エリア（皇居外苑、新宿御苑）
  - 3日目: 銀座・築地エリア（銀座のショッピング、築地市場）

- **宿泊先:** 上野周辺のビジネスホテル

---

### プラン2: モダン東京を満喫する旅行
- **日程**
  - 1日目: 渋谷・原宿エリア（渋谷スクランブル交差点、明治神宮）
  - 2日目: お台場エリア（ヴィーナスフォート、チームラボボーダレス）
  - 3日目: 六本木・赤坂エリア（六本木ヒルズ展望台、赤坂の会席料理）

- **宿泊先:** 渋谷または原宿近くのデザインホテル

---

### プラン3: 食とエンターテイメントに特化した旅行
- **日程**
  - 1日目: 秋葉原・上野エリア（秋葉原のオタク文化、アメ横）
  - 2日目: 恵比寿・代官山エリア（恵比寿ガーデンプレイス、代官山のカフェ）
  - 3日目: 銀座・築地エリア（銀座でラーメン、築地市場）

- **宿泊先:** 秋葉原周辺のカプセルホテル

これらのプランからお好きなテーマを選んで、特別な東京旅行をお楽しみください！
```

:::message

マルチエージェントで実行すると、プログラムが LLM の入力を受け続けて、終了しない場合がありました。[本家サンプルコード](https://github.com/Kong/volcano-sdk/blob/main/examples/06-multi-agent.ts)でも `process.exit(0);` が使われており、強制的にプロセスを終了させているため、同様に実装しておくと良いでしょう。

```ts
// プロセスを強制的に終了させないとLLMの入力を待ち続けてしまう。
process.exit(0);
```

:::

## 終わりに

Volcano SDK を使って AI Agent を構築する方法を紹介しました。Kong 社が提供しているライブラリということもあり、MCP First な作りになっている点やオブザーバビリティ関連機能がビルトインされている点など、他のライブラリにはない特徴があると思います。AI Agent の構築に興味がある方はぜひ試してみてください。以降のアドベントカレンダーも面白そうな記事がたくさん続きそうなのでお楽しみに 🦍

## 参考情報

https://konghq.com/blog/product-releases/volcano

https://volcano.dev/

https://github.com/Kong/volcano-sdk/tree/main/examples
