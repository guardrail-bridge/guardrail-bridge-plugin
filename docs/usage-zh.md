# Guardrails 使用指南

本文档说明最终用户如何在 OpenClaw 中启用和配置 `guardrail-bridge`。

## 1. 工作原理

插件注册 OpenClaw `before_dispatch` 钩子。它在用户消息发送给 Agent 之前检查消息内容。当策略拦截消息时，插件返回 `{ handled: true, text: blockMessage }`，OpenClaw 停止分发并回复 `blockMessage`；否则返回 `{ handled: false }`，消息继续发送给 Agent。

## 2. 配置入口

在 OpenClaw 配置文件中配置插件，通常是 `~/.openclaw/config.json5` 或环境选择的路径：

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

> 全局 connector 是可选的。你可以省略顶层的 `connector`，只为特定的 `channels.<channelId>` 条目启用 connector。

## 3. Connectors

### 3.1 Blacklist

```json5
{
  connector: "blacklist",
  blacklist: {
    blacklistFile: true,         // true = ~/.openclaw/guardrail-bridge/keywords.txt；string = 自定义路径；false = 禁用
    caseSensitive: false,
    hot: false,                  // 文件变更时自动重新加载
    hotDebounceMs: 300,
  },
  blockMessage: "This request has been blocked by the guardrail policy.",
}
```

关键词文件每行一个关键词。以 `#` 开头的行是注释。关键词可以可选地包含元数据级别后缀，如 `keyword|high`；支持的级别有 `low`、`medium`、`high`、`critical`。级别仅为元数据，不会改变匹配行为。

匹配前，文本会经过 NFC 规范化、全角转半角、零宽字符剥离和可选的小写转换处理。匹配使用 Aho-Corasick 多模式搜索。

当首次使用 `blacklistFile: true` 且默认文件不存在时，插件会将 `assets/keywords.default.txt` 复制到默认状态路径作为种子文件。

### 3.2 HTTP

```json5
{
  connector: "http",
  http: {
    provider: "dknownai",               // 或 "dknownai-cn" / "secra" / "hidylan" / 自定义
    apiKey: {                           // OpenClaw SecretRef（推荐）
      source: "env",
      provider: "default",
      id: "DKNOWNAI_API_KEY"
    },
    apiUrl: "",                         // 可选：覆盖 endpoint
    model: "",                          // 当前内置 provider 会忽略该字段
    params: {},                         // provider 特定参数
  },
  timeoutMs: 5000,
  fallbackOnError: "pass",              // 网络故障时的回退策略
}
```

> **所有 HTTP Provider 的端点地址说明**：所有 HTTP provider（内置和自定义）的默认端点地址均来自各 provider 的官方文档或官方网站。Provider 可能会变更其端点地址。生产环境部署时，请在 provider 官方网站确认当前端点地址。您可以通过 `apiUrl` 配置字段覆盖任何默认端点。

内置 provider：

| 名称                | apiKey 要求 | 默认 endpoint                                             |
| ------------------- | ----------- | --------------------------------------------------------- |
| `dknownai`          | 必填        | `https://open.dknownai.com/v1/guard`                       |
| `dknownai-cn`       | 必填        | `https://open.dknowc.cn/v1/guard`                         |
| `secra`             | 必填        | `https://secra-backend-production.up.railway.app`         |
| `hidylan`           | 可选        | 内置 Hidylan endpoint，可用 `apiUrl` 覆盖                 |

#### 注册自定义 provider

```typescript
import { registerHttpProvider } from "@guardrailbridge/guardrail-bridge/api";

registerHttpProvider("my-provider", {
  async init(config) {
    /* 一次性初始化（auth、连接池等） */
  },
  async check(text, context, config, fallbackOnError, timeoutMs) {
    // 返回 { action: "pass" } 或 { action: "block", blockMessage?: "…" }
  },
});
```

注意：内置 provider 的名字（`dknownai` / `dknownai-cn` / `secra` / `hidylan`）不可被覆盖。

## 4. 通用字段

| 字段              | 默认值 | 说明                                       |
| ----------------- | ------ | ------------------------------------------ |
| `timeoutMs`       | 5000   | 单次检查超时（500–30000）                  |
| `fallbackOnError` | `pass` | 出错回退：`pass` 放行 \| `block` 拦截     |
| `blockMessage`    | `This request has been blocked by the guardrail-bridge policy.` | 拦截时回复给用户的话术 |

## 5. 按 channel 覆盖

```json5
{
  "guardrail-bridge": {
    connector: "blacklist",            // 全局默认
    blacklist: { blacklistFile: true },

    channels: {
      "discord:@announcements": {
        connector: "http",             // 该 channel 切换到 HTTP
        http: {
          provider: "dknownai",
          apiKey: {                     // OpenClaw SecretRef
            source: "env",
            provider: "default",
            id: "DKNOWNAI_API_KEY"
          }
        },
        blockMessage: "公告频道仅接受合规内容。",
      },
      "telegram:@vip": {
        connector: "blacklist",
        blacklist: { blacklistFile: "/srv/guardrail-bridge/vip-keywords.txt" },
        blockMessage: "VIP 频道请求被安全策略拦截。",
      },
    },
  },
}
```

每个 channel 的字段是**部分覆盖**：`http` / `blacklist` 子对象会与全局对应字段做浅合并，`blockMessage` / `fallbackOnError` / `timeoutMs` 直接覆盖。

## 6. API 密钥配置

Guardrail Bridge 支持通过 OpenClaw SecretRef 安全地管理 API 密钥。

### SecretRef 配置示例

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

### SecretRef 详细文档

SecretRef 是 OpenClaw 平台的密钥管理功能。详细配置请参考 [OpenClaw Secrets 官方文档](https://docs.openclaw.ai/gateway/secrets)。

### 插件密钥处理特性

- **运行时解析**：插件在每次 `check()` 时动态解析 SecretRef
- **密钥最短驻留**：解析后的明文只在请求处理期间存在于内存中
- **错误处理**：SecretRef 解析失败时根据 `fallbackOnError` 配置返回 `"pass"` 或 `"block"`

## 7. 故障排查

- 启动时插件会日志输出 `guardrail-bridge: plugin registered (...)`，描述启用了哪些 channel handler。
- 若所有 connector 都未配置，会输出 `guardrail-bridge: no effective connector configured, plugin disabled`。
- HTTP connector 初始化失败（如 apiKey 缺失或 SecretRef 解析失败）会输出 `guardrail-bridge: failed to init HTTP adapter: ...`，并按 `fallbackOnError` 决定后续放行或拦截。
- Blacklist connector 默认会在第一次启用时把内置 `keywords.default.txt` 写入用户状态目录；如不希望写入，把 `blacklistFile` 改成自定义路径或 `false`。
