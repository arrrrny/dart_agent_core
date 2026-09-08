This page catalogs every LLM provider that `dart_agent_core` can connect to, explains how the unified client architecture works, and shows you the exact code to initialize each one. Whether you want to call OpenAI directly, run a local model with Ollama, or tap into Chinese cloud providers like Qwen and GLM — this page has the configuration you need.

## Architecture Overview

`dart_agent_core` abstracts all LLM communication behind a single [`LLMClient`](lib/src/core/llm_client.dart#L11-L25) interface. Every provider client implements the same two methods — `generate()` for single-turn requests and `stream()` for real-time token streaming — so your agent code never changes when you switch providers.

```mermaid
graph TB
    subgraph Agent Layer
        SA[StatefulAgent]
    end

    subgraph Unified Interface
        IFC[LLMClient interface<br/>generate() + stream()]
    end

    subgraph API Compatibility Families
        OAI[OpenAI Chat<br/>Completions API]
        RSP[OpenAI Responses<br/>API]
        ANT[Anthropic Messages<br/>API]
        GEN[Google Generative<br/>Language API]
        BED[AWS Bedrock<br/>API]
    end

    subgraph Client Classes
        OC[OpenAIClient]
        RC[ResponsesClient]
        CC[ClaudeClient]
        GC[GeminiClient]
        BC[BedrockClaudeClient]
    end

    SA --> IFC
    IFC --> OC
    IFC --> RC
    IFC --> CC
    IFC --> GC
    IFC --> BC

    OC -.->|compatible| OAI
    RC -.->|compatible| RSP
    CC -.->|compatible| ANT
    GC -.->|uses| GEN
    BC -.->|uses| BED

    style IFC fill:#e8f4fd,stroke:#1a73e8
    style OC fill:#e8f8e8,stroke:#34a853
    style RC fill:#e8f8e8,stroke:#34a853
    style CC fill:#fff3e0,stroke:#f9a825
    style GC fill:#fce4ec,stroke:#e91e63
    style BC fill:#fce4ec,stroke:#e91e63
```

The critical design principle is that **many providers share an API format**. If a provider exposes an OpenAI-compatible Chat Completions endpoint, you simply point `OpenAIClient` to its `baseUrl`. This means a single client class covers OpenAI itself plus Kimi, Qwen, GLM, Ollama, and OpenRouter — without any code changes in your agent logic.

Sources: [llm_client.dart](lib/src/core/llm_client.dart#L11-L25), [dart_agent_core.dart](lib/dart_agent_core.dart#L1-L32), [doc/providers.md](doc/providers.md#L1-L20)

## Client Classes at a Glance

The library ships five client classes. Each one targets a specific API format and authentication scheme.

| Client Class | API Format | Authentication | Primary Use Case |
|---|---|---|---|
| [`OpenAIClient`](lib/src/llm/openai_client.dart) | OpenAI Chat Completions | Bearer token (`apiKey`) | OpenAI + all Chat-Completions-compatible providers |
| [`ResponsesClient`](lib/src/llm/responses_client.dart) | OpenAI Responses API | Bearer token (`apiKey`) | OpenAI o-series + Responses-compatible providers |
| [`ClaudeClient`](lib/src/llm/claude_client.dart) | Anthropic Messages API | `x-api-key` header | Anthropic Claude (direct) + Anthropic-compatible providers |
| [`GeminiClient`](lib/src/llm/gemini_client.dart) | Google Generative Language API | API key (query parameter) | Google Gemini models |
| [`BedrockClaudeClient`](lib/src/llm/bedrock_claude_client.dart) | AWS Bedrock (Anthropic model) | AWS Signature V4 | Claude models via Amazon Bedrock |

Sources: [openai_client.dart](lib/src/llm/openai_client.dart#L1-L45), [responses_client.dart](lib/src/llm/responses_client.dart#L1-L45), [claude_client.dart](lib/src/llm/claude_client.dart#L1-L45), [gemini_client.dart](lib/src/llm/gemini_client.dart#L1-L45), [bedrock_claude_client.dart](lib/src/llm/bedrock_claude_client.dart#L1-L45)

## OpenAI

OpenAI is the default provider. `dart_agent_core` supports two distinct OpenAI API formats.

### Chat Completions API

The standard Chat Completions endpoint. The library appends `/chat/completions` to whatever `baseUrl` you provide.

```dart
final client = OpenAIClient(
  apiKey: Platform.environment['OPENAI_API_KEY'] ?? '',
);
final config = ModelConfig(model: 'gpt-4o', temperature: 0.7);
```

Sources: [doc/providers.md](doc/providers.md#L6-L35), [simple_agent_with_openai_example.dart](example/simple_agent_with_openai_example.dart#L11-L16)

### Responses API

OpenAI's newer Responses API uses a different request format. `ResponsesClient` handles response ID chaining automatically — it extracts `responseId` from the last `ModelMessage` and passes it as `previous_response_id` on the next request, enabling server-side context management.

```dart
final client = ResponsesClient(
  apiKey: Platform.environment['OPENAI_API_KEY'] ?? '',
);
final config = ModelConfig(model: 'gpt-4o');
```

You can also verify whether a stored response ID is still valid on the server:

```dart
final isValid = await client.checkResponseId(responseId);
```

Sources: [doc/providers.md](doc/providers.md#L37-L51), [responses_client.dart](lib/src/llm/responses_client.dart#L56-L60)

## Google Gemini

`GeminiClient` integrates with the Google Generative Language API. It handles Gemini-specific formatting automatically, including system instructions, content roles, and function call schemas.

```dart
final client = GeminiClient(
  apiKey: Platform.environment['GEMINI_API_KEY'] ?? '',
);
final config = ModelConfig(model: 'gemini-2.5-pro', temperature: 0.5);
```

Gemini supports the `topK` parameter in `ModelConfig` that other providers ignore.

Sources: [doc/providers.md](doc/providers.md#L53-L61), [simple_agent_with_gemini_example.dart](example/simple_agent_with_gemini_example.dart#L17-L20)

## Amazon Bedrock (Claude)

Bedrock is AWS's managed service for running Claude models. Unlike other providers, authentication uses **AWS Signature V4** — the client computes the signature automatically from your credentials. You do not need an API key; you need AWS access key ID, secret access key, and an optional session token for temporary credentials.

```dart
final client = BedrockClaudeClient(
  region: 'us-east-1',
  accessKeyId: Platform.environment['AWS_ACCESS_KEY_ID'] ?? '',
  secretAccessKey: Platform.environment['AWS_SECRET_ACCESS_KEY'] ?? '',
  sessionToken: Platform.environment['AWS_SESSION_TOKEN'], // optional
);
final config = ModelConfig(model: 'us.anthropic.claude-3-5-sonnet-20241022-v2:0');
```

The model ID must follow Bedrock's naming convention: `us.anthropic.claude-...` with a region prefix.

Sources: [doc/providers.md](doc/providers.md#L63-L79), [simple_agent_with_bedrock_claude_example.dart](example/simple_agent_with_bedrock_claude_example.dart#L21-L26)

## Anthropic Claude (Direct)

For calling Claude directly through the Anthropic API (not through AWS Bedrock), use `ClaudeClient`. It authenticates via the `x-api-key` header and uses the `anthropic-version` header for API versioning.

```dart
final client = ClaudeClient(
  apiKey: Platform.environment['ANTHROPIC_API_KEY'] ?? '',
);
final config = ModelConfig(model: 'claude-sonnet-4-20250514');
```

Sources: [doc/providers.md](doc/providers.md#L200-L208), [claude_client.dart](lib/src/llm/claude_client.dart#L35-L45), [simple_agent_with_claude_example.dart](example/simple_agent_with_claude_example.dart#L11-L14)

## OpenAI-Compatible Providers

Many LLM providers expose an OpenAI-compatible Chat Completions API. To use them, you only need to create an `OpenAIClient` with a custom `baseUrl` — no client subclass required.

### Kimi (Moonshot AI)

```dart
final client = OpenAIClient(
  apiKey: Platform.environment['MOONSHOT_API_KEY'] ?? '',
  baseUrl: 'https://api.moonshot.cn/v1',
);
final config = ModelConfig(model: 'kimi-k2');
// For thinking models: ModelConfig(model: 'kimi-k2-thinking')
// reasoning_content is handled automatically
```

Sources: [doc/providers.md](doc/providers.md#L99-L107), [simple_agent_with_kimi_example.dart](example/simple_agent_with_kimi_example.dart#L18-L22)

### Qwen (Alibaba DashScope)

```dart
final client = OpenAIClient(
  apiKey: Platform.environment['DASHSCOPE_API_KEY'] ?? '',
  baseUrl: 'https://dashscope.aliyuncs.com/compatible-mode/v1',
);
final config = ModelConfig(model: 'qwen3.5-plus');
```

Sources: [doc/providers.md](doc/providers.md#L109-L117), [simple_agent_with_qwen_example.dart](example/simple_agent_with_qwen_example.dart#L18-L22)

### Zhipu GLM

```dart
final client = OpenAIClient(
  apiKey: Platform.environment['GLM_API_KEY'] ?? '',
  baseUrl: 'https://open.bigmodel.cn/api/coding/paas/v4',
);
final config = ModelConfig(model: 'GLM-4.7');
```

Sources: [doc/providers.md](doc/providers.md#L119-L127), [simple_agent_with_glm_example.dart](example/simple_agent_with_glm_example.dart#L18-L22)

### Ollama (Local)

Ollama runs models locally on your machine and exposes an OpenAI-compatible API at `localhost:11434/v1`. No API key is required.

```dart
final client = OpenAIClient(
  apiKey: '', // Ollama does not require an API key
  baseUrl: 'http://localhost:11434/v1',
);
final config = ModelConfig(model: 'qwen2.5:7b');
```

Sources: [doc/providers.md](doc/providers.md#L129-L137), [simple_agent_with_ollama_example.dart](example/simple_agent_with_ollama_example.dart#L14-L18)

### OpenRouter

OpenRouter provides a unified gateway to hundreds of models from different providers through a single OpenAI-compatible API.

```dart
final client = OpenAIClient(
  apiKey: Platform.environment['OPENROUTER_API_KEY'] ?? '',
  baseUrl: 'https://openrouter.ai/api/v1',
);
final config = ModelConfig(
  model: 'anthropic/claude-opus-4.6',
  extra: {
    'thinking': {'type': 'enabled', 'budget_tokens': 2000},
  },
);
```

Sources: [doc/providers.md](doc/providers.md#L139-L147), [simple_agent_with_openrouter_example.dart](example/simple_agent_with_openrouter_example.dart#L18-L24)

## Responses API-Compatible Providers

### Volcengine Doubao-Seed

Volcengine's Doubao-Seed models are compatible with the OpenAI Responses API format.

```dart
final client = ResponsesClient(
  apiKey: Platform.environment['ARK_API_KEY'] ?? '',
  baseUrl: 'https://ark.cn-beijing.volces.com/api/v3',
);
final config = ModelConfig(model: 'doubao-seed-1-8-251228');
```

Sources: [doc/providers.md](doc/providers.md#L149-L160), [simple_agent_with_seed_example.dart](example/simple_agent_with_seed_example.dart#L18-L22)

## Anthropic API-Compatible Providers

### MiniMax

MiniMax exposes an Anthropic-compatible API endpoint. Use `ClaudeClient` with MiniMax's base URL.

```dart
final client = ClaudeClient(
  apiKey: Platform.environment['MINIMAX_API_KEY'] ?? '',
  baseUrl: 'https://api.minimaxi.com/anthropic',
);
final config = ModelConfig(model: 'MiniMax-M2.5');
```

Sources: [doc/providers.md](doc/providers.md#L216-L226), [simple_agent_with_minimax_example.dart](example/simple_agent_with_minimax_example.dart#L18-L22)

## Provider Configuration Reference

### OpenAI Client Parameters

The `OpenAIClient` constructor exposes several tuning parameters beyond `apiKey` and `baseUrl`.

| Parameter | Default | Description |
|---|---|---|
| `timeout` | `300s` | Request timeout for both send and receive |
| `connectTimeout` | `60s` | TCP connection timeout |
| `proxyUrl` | `null` | HTTP proxy URL (supports `http://user:pass@host:port`) |
| `maxRetries` | `3` | Number of retries on transient errors |
| `initialRetryDelayMs` | `1000` | Initial exponential backoff delay in milliseconds |
| `maxRetryDelayMs` | `10000` | Cap on exponential backoff delay |

Sources: [openai_client.dart](lib/src/llm/openai_client.dart#L30-L38), [doc/providers.md](doc/providers.md#L25-L33)

### Common Parameters Across All Clients

Every client class accepts a consistent set of constructor parameters, enabling uniform configuration regardless of which provider you use.

| Parameter | Available In | Description |
|---|---|---|
| `timeout` | All clients | Request timeout (default 300s) |
| `connectTimeout` | OpenAI, Responses, Gemini | TCP connection timeout (default 60s) |
| `proxyUrl` | All clients | HTTP proxy URL |
| `maxRetries` | All clients | Retry count (default 3) |
| `initialRetryDelayMs` | All clients | Initial retry delay (default 1000ms, Gemini: 5000ms) |
| `maxRetryDelayMs` | All clients | Max retry delay (default 10000ms, Gemini: 30000ms) |

Sources: [openai_client.dart](lib/src/llm/openai_client.dart#L30-L38), [gemini_client.dart](lib/src/llm/gemini_client.dart#L20-L25), [claude_client.dart](lib/src/llm/claude_client.dart#L35-L42), [bedrock_claude_client.dart](lib/src/llm/bedrock_claude_client.dart#L30-L38), [responses_client.dart](lib/src/llm/responses_client.dart#L30-L38)

## ModelConfig

`ModelConfig` is the unified configuration object passed to `StatefulAgent` and forwarded to the LLM client on every call.

```dart
final config = ModelConfig(
  model: 'gpt-4o-mini',
  temperature: 0.7,
  maxTokens: 4096,
  topP: 0.9,
  // topK: Gemini only
  // extra: provider-specific parameters
  // generationConfig: Gemini-specific generation config
);
```

| Field | Type | Description |
|---|---|---|
| `model` | `String` | **Required.** Model identifier (provider-specific) |
| `temperature` | `double?` | Sampling temperature (0.0–2.0) |
| `maxTokens` | `int?` | Maximum tokens in the response |
| `topP` | `double?` | Nucleus sampling threshold |
| `topK` | `double?` | Top-K sampling (Gemini-specific) |
| `extra` | `Map<String, dynamic>?` | Provider-specific parameters merged into the request body |
| `generationConfig` | `Map<String, dynamic>?` | Gemini-specific generation configuration |

Sources: [llm_client.dart](lib/src/core/llm_client.dart#L94-L136), [doc/providers.md](doc/providers.md#L81-L95)

## Provider-Specific Parameters via `extra`

The `extra` map is your escape hatch for provider-specific features that `ModelConfig` does not have dedicated fields for. Values in `extra` are merged directly into the HTTP request body.

### Claude Extended Thinking (Bedrock)

Enable step-by-step reasoning with a thinking budget:

```dart
final config = ModelConfig(
  model: 'us.anthropic.claude-3-7-sonnet-20250219-v1:0',
  maxTokens: 16000,
  extra: {
    'thinking': {'type': 'enabled', 'budget_tokens': 10000},
  },
);
```

When thinking is enabled, the model's reasoning process is available in `modelMessage.thought` and the verification signature in `modelMessage.thoughtSignature`.

Sources: [doc/providers.md](doc/providers.md#L97-L113), [simple_agent_with_thinking_example.dart](example/simple_agent_with_thinking_example.dart#L49-L52)

### OpenAI Reasoning Models (o-series)

Control reasoning effort for models like `o3-mini`:

```dart
final config = ModelConfig(
  model: 'o3-mini',
  extra: {'reasoning_effort': 'high'},
);
```

Sources: [doc/providers.md](doc/providers.md#L115-L121)

## Thinking and Reasoning Models

Models that support extended thinking — such as `kimi-k2-thinking`, `o1`, `o3`, and `deepseek-r1` — return a `reasoning_content` field in their responses. The framework handles this transparently:

- **Streaming and non-streaming** responses parse `reasoning_content` into `ModelMessage.thought`
- **Multi-turn conversations** re-send `reasoning_content` in assistant messages as required by the API
- **Access the thinking content** via `modelMessage.thought`

```dart
final lastMessage = responses.last as ModelMessage;
if (lastMessage.thought != null) {
  print('Agent thought process:\n${lastMessage.thought}');
}
print('Agent response: ${lastMessage.textOutput}');
```

Sources: [doc/providers.md](doc/providers.md#L228-L233), [simple_agent_with_thinking_example.dart](example/simple_agent_with_thinking_example.dart#L74-L80), [simple_agent_with_openrouter_example.dart](example/simple_agent_with_openrouter_example.dart#L48-L51)

## Proxy Support

All clients support HTTP proxies via the `proxyUrl` parameter. Basic authentication is supported in the proxy URL.

```dart
final client = OpenAIClient(
  apiKey: '...',
  proxyUrl: 'http://user:password@proxy.example.com:8080',
);
```

Sources: [doc/providers.md](doc/providers.md#L155-L162)

## Provider Compatibility Matrix

This table summarizes which client class to use for each provider, what API format they follow, and the recommended environment variable for the API key.

| Provider | Client Class | API Compatibility | API Key Env Var |
|---|---|---|---|
| OpenAI | `OpenAIClient` | Chat Completions | `OPENAI_API_KEY` |
| OpenAI (o-series) | `ResponsesClient` | Responses API | `OPENAI_API_KEY` |
| Google Gemini | `GeminiClient` | Google Generative Language | `GEMINI_API_KEY` |
| Amazon Bedrock (Claude) | `BedrockClaudeClient` | AWS Bedrock | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` |
| Anthropic (Direct) | `ClaudeClient` | Anthropic Messages | `ANTHROPIC_API_KEY` |
| Kimi (Moonshot AI) | `OpenAIClient` | OpenAI Chat Completions | `MOONSHOT_API_KEY` |
| Qwen (DashScope) | `OpenAIClient` | OpenAI Chat Completions | `DASHSCOPE_API_KEY` |
| Zhipu GLM | `OpenAIClient` | OpenAI Chat Completions | `GLM_API_KEY` |
| Ollama (Local) | `OpenAIClient` | OpenAI Chat Completions | *(none required)* |
| OpenRouter | `OpenAIClient` | OpenAI Chat Completions | `OPENROUTER_API_KEY` |
| Volcengine Doubao-Seed | `ResponsesClient` | OpenAI Responses API | `ARK_API_KEY` |
| MiniMax | `ClaudeClient` | Anthropic Messages | `MINIMAX_API_KEY` |

## Next Steps

Now that you know which provider to use, continue to [StatefulAgent and the Agentic Loop](5-statefulagent-and-the-agentic-loop) to learn how agents consume these clients, or revisit [Quick Start](2-quick-start) to build your first working agent end-to-end.