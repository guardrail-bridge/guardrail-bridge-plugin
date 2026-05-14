# Guardrails Usage Guide

This document explains how end users can enable and configure `guardrail-bridge` in OpenClaw.

## 1. How it works

The plugin registers an OpenClaw `before_dispatch` hook. It checks user messages before they are sent to the Agent. When a policy blocks the message, the plugin returns `{ handled: true, text: blockMessage }`, so OpenClaw stops the dispatch and replies with `blockMessage`. Otherwise, it returns `{ handled: false }` and the message continues to the Agent.

## 2. Configuration entry

Configure the plugin in the OpenClaw config file, usually `~/.openclaw/config.json5` or the path selected by your environment:

```json5
{
  plugins: {
    entries: {
      "guardrail-bridge": {
        // configuration
      },
    },
  },
}
```

> The global connector is optional. You can omit top-level `connector` and enable a connector only for a specific `channels.<channelId>` entry.

## 3. Connectors

### 3.1 Blacklist

```json5
{
  connector: "blacklist",
  blacklist: {
    blacklistFile: true,         // true = ~/.openclaw/guardrail-bridge/keywords.txt; string = custom path; false = disabled
    caseSensitive: false,
    hot: false,                  // reload automatically when the file changes
    hotDebounceMs: 300,
  },
  blockMessage: "This request has been blocked by the guardrail policy.",
}
```

The keyword file contains one keyword per line. Lines starting with `#` are comments. A keyword can optionally include a metadata level suffix, such as `keyword|high`; supported levels are `low`, `medium`, `high`, and `critical`. Levels are metadata only and do not change matching behavior.

Before matching, text is normalized with NFC normalization, full-width to half-width conversion, zero-width character stripping, and optional lowercasing. Matching uses Aho-Corasick multi-pattern search.

When `blacklistFile: true` is used for the first time and the default file does not exist, the plugin copies `assets/keywords.default.txt` to the default state path as a seed file.

### 3.2 HTTP

```json5
{
  connector: "http",
  http: {
    provider: "dknownai",               // or "dknownai-cn" / "secra" / "hidylan" / a custom provider
    apiKey: {                           // OpenClaw SecretRef (recommended)
      source: "env",
      provider: "default",
      id: "DKNOWNAI_API_KEY"
    },
    apiUrl: "",                         // optional endpoint override
    model: "",                          // ignored by current built-in providers
    params: {},                         // provider-specific parameters
  },
  timeoutMs: 5000,
  fallbackOnError: "pass",              // fallback action for network or provider errors
}
```

> **Endpoint source note for all HTTP providers**: Default endpoints for all HTTP providers (built-in and custom) are obtained from official provider documentation and websites. Providers may change their endpoints over time. For production deployments, please verify the current endpoint on the provider's official website. You can override any endpoint using the `apiUrl` configuration field.

Built-in providers:

| Name | apiKey | Default endpoint |
| --- | --- | --- |
| `dknownai` | Required | `https://open.dknownai.com/v1/guard` |
| `dknownai-cn` | Required | `https://open.dknowc.cn/v1/guard` |
| `secra` | Required | `https://secra-backend-production.up.railway.app` |
| `hidylan` | Optional | Built-in Hidylan endpoint; can be overridden with `apiUrl` |

#### Custom providers

```typescript
import { registerHttpProvider } from "@guardrailbridge/guardrail-bridge/api";

registerHttpProvider("my-provider", {
  async init(config) {
    // Initialize auth, connection pools, or provider state.
  },
  async check(text, context, config, fallbackOnError, timeoutMs) {
    return { action: "pass" };
  },
});
```

Built-in provider names (`dknownai`, `dknownai-cn`, `secra`, `hidylan`) are reserved and cannot be overridden.

## 4. Common fields

| Field | Default | Description |
| --- | --- | --- |
| `timeoutMs` | 5000 | Single check timeout in milliseconds. Valid range: 500–30000. |
| `fallbackOnError` | `pass` | Fallback action when a connector fails: `pass` or `block`. |
| `blockMessage` | `This request has been blocked by the guardrail-bridge policy.` | Message returned to the user when a request is blocked. |

## 5. Per-channel overrides

```json5
{
  "guardrail-bridge": {
    connector: "blacklist",
    blacklist: { blacklistFile: true },

    channels: {
      "discord:@announcements": {
        connector: "http",
        http: {
          provider: "dknownai",
          apiKey: {
            source: "env",
            provider: "default",
            id: "DKNOWNAI_API_KEY"
          }
        },
        blockMessage: "Only compliant content is allowed in the announcements channel.",
      },
      "telegram:@vip": {
        connector: "blacklist",
        blacklist: { blacklistFile: "/srv/guardrail-bridge/vip-keywords.txt" },
        blockMessage: "This VIP channel request has been blocked by policy.",
      },
    },
  },
}
```

Channel fields are partial overrides. `http` and `blacklist` are shallow-merged with the global objects. Scalar fields such as `blockMessage`, `fallbackOnError`, and `timeoutMs` directly replace the global value.

## 6. API Key Configuration

Guardrail Bridge supports OpenClaw SecretRef for secure API key management.

### Quick Example

```json5
{
  "http": {
    "provider": "dknownai",
    "apiKey": {
      "source": "env",
      "provider": "default",
      "id": "DKNOWNAI_API_KEY"
    }
  }
}
```

SecretRef supports three sources:
- `env`: Environment variables
- `file`: JSON files (JSON Pointer paths)
- `exec`: External commands (1Password, Vault, sops)

For detailed SecretRef configuration, see the [OpenClaw Secrets documentation](https://docs.openclaw.ai/gateway/secrets).

### Plugin Behavior

- **Runtime resolution**: SecretRef is resolved at each `check()` boundary
- **Minimal secret residency**: Plaintext secrets exist in memory only during request processing
- **Error handling**: SecretRef resolution failures follow `fallbackOnError` configuration

## 7. Troubleshooting

- On startup, the plugin logs `guardrail-bridge: plugin registered (...)` with the enabled channel handlers.
- If no effective connector is configured, it logs `guardrail-bridge: no effective connector configured, plugin disabled`.
- If HTTP connector initialization fails (e.g., missing apiKey or SecretRef resolution failure), it logs `guardrail-bridge: failed to init HTTP adapter: ...` and follows `fallbackOnError`.
- The blacklist connector writes the default keyword file on first use when `blacklistFile: true`. Use a custom path or `false` if you do not want this side effect.
