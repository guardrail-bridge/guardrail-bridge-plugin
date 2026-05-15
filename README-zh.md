# Guardrail Bridge 插件

面向 OpenClaw 的 pre-agent 安全插件。在消息分发前检测操纵尝试并拦截违规内容。

## 兼容性

- **支持的 OpenClaw 版本**：`>=2026.4.26`
- **支持的 Plugin API**：`>=2026.4.26`

打包产物基于 OpenClaw `2026.4.26` 构建，兼容性元数据同时声明在 `peerDependencies.openclaw` 和 `openclaw.compat.pluginApi` 中。

## 分发路径

- **ClawHub / OpenClaw 安装目标**：`clawhub:@guardrailbridge/guardrail-bridge`
- **npm 包名**：`@guardrailbridge/guardrail-bridge`

发布归档只包含运行时代码、插件 manifest、静态资源和最终用户文档，不包含开发文档。

## 为什么需要它

OpenClaw Agent 本身已经具备一定的基础安全能力。在我们的测试中，未启用 Guardrail Bridge 的 Agent 面对直接索要 API Key 的请求时，一开始会拒绝披露，并建议通过更安全的运维方式处理。

风险出现在后续多轮诱导、Base64 编码请求和紧急施压之后。没有额外 guardrail 时，Agent 最终返回了编码后的凭据；启用 Guardrail Bridge 后，这类后续外泄攻击会在披露前被策略拦截。

![Guardrail Bridge API key leakage comparison](https://raw.githubusercontent.com/guardrail-bridge/guardrail-bridge-plugin/main/assets/api-key-leakage-comparison.svg)

图中敏感值均已脱敏。完整案例见：[阻止 API Key 外泄案例](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/case-study-api-key-leakage.md)。

## 功能

插件在用户消息进入 Agent 前运行检查，基于两种安全策略拦截请求：

- **Blacklist**：使用 Aho-Corasick 多模匹配对本地关键字文件进行匹配。
- **HTTP**：远端审核 API，内置 provider：`dknownai`、`dknownai-cn`、`secra`、`hidylan`。

每个 channel 可以独立选择 connector 并覆写参数；全局 connector 可选。

## HTTP Provider

### DKnownAI

检测提示注入、越狱和 Agent 劫持尝试，适合需要远端安全审核的场景。

- **Provider 名称**：`dknownai`（国际）、`dknownai-cn`（中国）
- **API key 要求**：必填
- **API key 获取**：`dknownai` 请访问 [dknownai.com](https://dknownai.com/)；`dknownai-cn` 请访问 [dknowc.cn](https://www.dknowc.cn/)。

### Secra

远端内容审核 provider，可用于接入额外的消息安全审查能力。

- **Provider 名称**：`secra`
- **API key 要求**：必填
- **API URL 要求**：否（默认使用官方 Railway 托管端点；可通过 `apiUrl` 覆盖）
- **官网**：[secra.ai](https://secra.ai/)

### Hidylan

偏向提示注入检查的远端 provider，适合识别不安全指令和策略绕过尝试。

- **Provider 名称**：`hidylan`
- **API key 要求**：可选
- **官网**：[hidylan.ai](https://hidylan.ai/)

## 配置

### 快速开始：Blacklist

在 OpenClaw 配置中启用插件：

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
          blockMessage: "您的请求已被安全策略拦截。",
          fallbackOnError: "pass",
        },
      },
    },
  },
}
```

### HTTP Provider 示例：DKnownAI

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
            apiKey: {                     // OpenClaw SecretRef
              source: "env",
              provider: "default",
              id: "DKNOWNAI_API_KEY"
            },
          },
          fallbackOnError: "block",
        },
      },
    },
  },
}
```

### HTTP Provider 示例：DKnownAI 中国版

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
            apiKey: {                     // OpenClaw SecretRef
              source: "env",
              provider: "default",
              id: "DKNOWNAI_CN_API_KEY"
            },
          },
          fallbackOnError: "block",
        },
      },
    },
  },
}
```

### HTTP Provider 示例：Secra

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
            apiKey: {                     // OpenClaw SecretRef
              source: "env",
              provider: "default",
              id: "SECRA_API_KEY"
            },
          },
          fallbackOnError: "block",
        },
      },
    },
  }
}
```

### HTTP Provider 示例：Hidylan

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

### 配置 API Key

Guardrail Bridge 支持 [OpenClaw SecretRef](https://docs.openclaw.ai/gateway/secrets) 进行安全的密钥管理。

**使用 SecretRef**（推荐）：

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

SecretRef 支持三种来源：
- `env`：环境变量
- `file`：JSON 文件（JSON Pointer 路径）
- `exec`：外部命令（1Password、Vault、sops）

**明文密钥**（不推荐生产环境）：

```json5
"apiKey": "sk-..."
```

> **⚠️ 安全警告**：切勿将明文 API Key 提交到版本控制系统。生产环境务必使用 SecretRef。

3. **按 channel 覆写**：

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
              },
             },
             blockMessage: "公告频道仅接受合规内容。",
           },
         },
       },
     },
   }
   ```

### 通用字段

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `connector` | `""` | Connector 类型：`"blacklist"` 或 `"http"`。空字符串表示从配置自动推断。 |
| `timeoutMs` | 5000 | 单次检查超时（500–30000）。 |
| `fallbackOnError` | `"pass"` | 连接器失败时的回退操作：`"pass"` 或 `"block"`。 |
| `blockMessage` | `This request has been blocked by the guardrail-bridge policy.` | 拦截时返回给用户的消息。 |

### Blacklist 配置

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `blacklistFile` | `false` | 关键字文件源。`true` = `~/.openclaw/guardrail-bridge/keywords.txt`；string = 自定义路径；`false` = 禁用。 |
| `caseSensitive` | `false` | 启用大小写敏感匹配。 |
| `hot` | `false` | 文件变更时自动重新加载关键字文件。 |
| `hotDebounceMs` | 300 | 热重载防抖间隔（毫秒）。 |

### HTTP 配置

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `provider` | 是 | Provider 名称：`dknownai`、`dknownai-cn`、`secra` 或 `hidylan`。 |
| `apiKey` | 是（`hidylan` 除外） | Provider API key。可使用环境变量替换。 |
| `apiUrl` | 是（`secra` 需要） | Endpoint URL。`secra` provider 必须配置；其他 provider 可选覆盖。 |
| `model` | 否 | 模型名称。当前内置 provider 忽略该字段。 |
| `params` | 否 | Provider 特定参数（如 `project_id`、`region`）。 |

## 安装

支持通过 ClawHub 或 npm 两种方式安装，两者的安装标识不同。

### 通过 ClawHub 安装

```bash
openclaw plugins install clawhub:@guardrailbridge/guardrail-bridge
```

### 通过 npm 安装

```bash
openclaw plugins install npm:@guardrailbridge/guardrail-bridge
```

安装或修改插件配置后，需要重启 OpenClaw gateway。


## 文档

- English:
  - [Usage](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/usage.md)
  - [Manifest schema](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/manifest-schema.md)
  - [Security notes](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/security-notes.md)
- 中文:
  - [使用指南](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/usage-zh.md)
  - [Manifest schema](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/manifest-schema-zh.md)
  - [安全说明](https://github.com/guardrail-bridge/guardrail-bridge-plugin/blob/main/docs/security-notes-zh.md)

## 许可证

MIT
