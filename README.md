# Guardrail Bridge Plugin

Pre-agent security plugin for OpenClaw. Detects manipulation attempts and blocks policy-violating content before Agent dispatch.

## Compatibility

- **Supported OpenClaw versions**: `>=2026.4.26`
- **Supported Plugin API**: `>=2026.4.26`

The packaged runtime is built against OpenClaw `2026.4.26`, and the compatibility metadata is declared in both `peerDependencies.openclaw` and `openclaw.compat.pluginApi`.

## Distribution Paths

- **ClawHub / OpenClaw install target**: `clawhub:@guardrailbridge/guardrail-bridge`
- **npm package**: `@guardrailbridge/guardrail-bridge`

Published archives include the runtime bundle, plugin manifest, assets, and end-user documentation only.

## Why it matters

OpenClaw agents already have baseline safety behavior. In our test, the unprotected agent initially refused a direct request to reveal an API key and suggested safer operational steps.

The risk appeared after the conversation shifted into multi-turn pressure and an encoding request. Without an additional guardrail, the agent eventually returned a Base64-encoded credential. With Guardrail Bridge enabled, that later-stage exfiltration attempt was blocked before disclosure.

![Guardrail Bridge API key leakage comparison](https://raw.githubusercontent.com/guardrail-bridge/guardrail-bridge-plugin/main/assets/api-key-leakage-comparison.svg)

Sensitive values are redacted. See the full case study: [Blocking API Key Exfiltration](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/case-study-api-key-leakage.md).

## What It Does

This plugin runs before user messages are dispatched to the Agent and can block requests based on two safety strategies:

- **Blacklist**: Local keyword matching using Aho-Corasick multi-pattern search over a configurable keyword file.
- **HTTP**: Remote moderation API with built-in providers: `dknownai`, `dknownai-cn`, `secra`, `hidylan`.

Each channel can choose its own connector and override connector options. A global connector is optional.

## HTTP Providers

### DKnownAI

Detects prompt injection, jailbreak, and agent hijacking attempts for deployments that need remote security review.

- **Provider names**: `dknownai` (international), `dknownai-cn` (China)
- **API key required**: Yes
- **Get API keys**: use [dknownai.com](https://dknownai.com/) for `dknownai`, or [dknowc.cn](https://www.dknowc.cn/) for `dknownai-cn`.

### Secra

Remote content moderation provider for adding extra message safety review.

- **Provider name**: `secra`
- **API key required**: Yes
- **API URL required**: No (defaults to official Railway-hosted endpoint; can be overridden with `apiUrl`)
- **Website**: [secra.ai](https://secra.ai/)

### Hidylan

Remote prompt-injection checking provider for identifying unsafe instructions and policy-bypass attempts.

- **Provider name**: `hidylan`
- **API key required**: Optional
- **Website**: [hidylan.ai](https://hidylan.ai/)

## Configuration

### Quick Start: Blacklist

Enable the plugin in the OpenClaw config:

```json5
{
  plugins: {
    entries: {
      "guardrail-bridge": {
        enabled: true,
        config: {
          connector: "blacklist",
          blacklist: {
            blacklistFile: true,
            caseSensitive: false,
            hot: true,
          },
          blockMessage: "This request has been blocked by the guardrail policy.",
          fallbackOnError: "pass",
        },
      },
    },
  },
}
```

### HTTP Provider Example: DKnownAI

```json5
{
  plugins: {
    entries: {
      "guardrail-bridge": {
        enabled: true,
        config: {
          connector: "http",
          http: {
            provider: "dknownai",
            apiKey: {
              source: "env",
              provider: "default",
              id: "DKNOWNAI_API_KEY"
            }
          },
          fallbackOnError: "pass",
        },
      },
    },
  },
}
```

### HTTP Provider Example: DKnownAI China

```json5
{
  plugins: {
    entries: {
      "guardrail-bridge": {
        enabled: true,
        config: {
          connector: "http",
          http: {
            provider: "dknownai-cn",
            apiKey: {
              source: "env",
              provider: "default",
              id: "DKNOWNAI_CN_API_KEY"
            },
          },
          fallbackOnError: "pass",
        },
      },
    },
  },
}
```

### HTTP Provider Example: Secra

```json5
{
  plugins: {
    entries: {
      "guardrail-bridge": {
        enabled: true,
        config: {
          connector: "http",
          http: {
            provider: "secra",
            apiKey: {
              source: "env",
              provider: "default",
              id: "SECRA_API_KEY"
            }
          },
          fallbackOnError: "pass",
        },
      },
    },
  }
}
```

### HTTP Provider Example: Hidylan

```json5
{
  plugins: {
    entries: {
      "guardrail-bridge": {
        enabled: true,
        config: {
          connector: "http",
          http: {
            provider: "hidylan",
            apiKey: {
              source: "env",
              provider: "default",
              id: "HIDYLAN_API_KEY"
            },
          },
          fallbackOnError: "pass",
        },
      },
    },
  }
}
```

### Configuring API Keys

Guardrail Bridge supports [OpenClaw SecretRef](https://docs.openclaw.ai/gateway/secrets) for secure credential management. API keys are resolved at runtime from external sources and never stored in plaintext in configuration files.

**Using SecretRef** (recommended):

```json5
{
  http: {
    provider: "dknownai",
    apiKey: {
      source: "env",
      provider: "default",
      id: "DKNOWNAI_API_KEY"
    }
  }
}
```

SecretRef supports three sources:

| Source | Description | Example |
|--------|-------------|---------|
| `env` | Environment variable | `{ source: "env", provider: "default", id: "MY_KEY" }` |
| `file` | JSON file (JSON Pointer path) | `{ source: "file", provider: "my-secrets", id: "/providers/openai/apiKey" }` |
| `exec` | External command (1Password, Vault, sops) | `{ source: "exec", provider: "vault", id: "openai/api-key" }` |

For detailed SecretRef configuration and setup, see the [OpenClaw Secrets documentation](https://docs.openclaw.ai/gateway/secrets).

**Plain text** (not recommended):

```json5
"apiKey": "sk-..."
```

> **⚠️ Security Warning**: Never commit API keys in plaintext to version control. Always use SecretRef for production deployments.

**Per-channel override**:

```json5
{
  "guardrail-bridge": {
    config: {
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
          blockMessage: "Only compliant content is allowed."
        },
      },
    },
  }
}
```

### Common Fields

| Field | Default | Description |
| --- | --- | --- |
| `connector` | `""` | Connector type: `"blacklist"` or `"http"`. Empty auto-detects from config. |
| `timeoutMs` | 5000 | Single check timeout in milliseconds (500–30000). |
| `fallbackOnError` | `"pass"` | Fallback action when a connector fails: `"pass"` or `"block"`. |
| `blockMessage` | `This request has been blocked by the guardrail-bridge policy.` | Message returned to the user when a request is blocked. |

### Blacklist Configuration

| Field | Default | Description |
| --- | --- | --- |
| `blacklistFile` | `false` | Keyword file source. `true` = `~/.openclaw/guardrail-bridge/keywords.txt`; string = custom path; `false` = disabled. |
| `caseSensitive` | `false` | Enables case-sensitive matching. |
| `hot` | `false` | Automatically reload the keyword file when it changes. |
| `hotDebounceMs` | 300 | Hot-reload debounce interval in milliseconds. |

### HTTP Configuration

| Field | Required | Description |
| --- | --- | --- |
| `provider` | Yes | Provider name: `dknownai`, `dknownai-cn`, `secra`, or `hidylan`. |
| `apiKey` | Yes (except `hidylan`) | Provider API key. Can use environment variable substitution. |
| `apiUrl` | Yes (for `secra`) | Endpoint URL. Required for `secra` provider; optional override for others. |
| `model` | No | Model name. Current built-in providers ignore this field. |
| `params` | No | Provider-specific parameters (e.g., `project_id`, `region`). |

## Installation

You can install the plugin through either ClawHub or npm. The install identifiers are different.

### Install from ClawHub

```bash
openclaw plugins install clawhub:@guardrailbridge/guardrail-bridge
```

### Install from npm

```bash
openclaw plugins install npm:@guardrailbridge/guardrail-bridge
```

Restart the OpenClaw gateway after installing or changing plugin configuration.


## Documentation

- English:
  - [Usage](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/usage.md)
  - [Manifest schema](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/manifest-schema.md)
  - [Security notes](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/security-notes.md)
- 中文:
  - [README-zh](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/README-zh.md)
  - [使用指南](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/usage-zh.md)
  - [Manifest schema](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/manifest-schema-zh.md)
  - [安全说明](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/security-notes-zh.md)

## License

MIT
