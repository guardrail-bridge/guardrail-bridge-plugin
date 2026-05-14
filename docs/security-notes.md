# Security Notes

## Naming clarification

OpenClaw security documentation often uses the word "guardrails" for general safety controls, such as exec approvals, prompt controls, tool policies, allowlists, and sandboxing. That is separate from this plugin.

- General OpenClaw guardrails protect operator intent and tool execution.
- `guardrail-bridge` checks inbound user message content before dispatch, using blacklist or remote HTTP moderation connectors.

These layers are complementary and do not replace each other.

## Plugin security considerations

### HTTP connector

- All built-in providers send requests through the SDK `fetchWithSsrFGuard`, so the OpenClaw host can enforce SSRF policy, private-network blocking, host allowlists, and timeouts.
- API keys are resolved at runtime through OpenClaw SecretRef. The plugin receives `apiKey` as either a plain string or a SecretRef object, resolves it at each check() boundary, and never stores the plaintext secret in long-lived state.
- `timeoutMs` defaults to 5 seconds and is capped at 30 seconds. Timeout behavior follows `fallbackOnError`.

### Blacklist connector

- When `blacklistFile: true` is enabled for the first time, the plugin copies `assets/keywords.default.txt` to `~/.openclaw/guardrail-bridge/keywords.txt`. Use a custom path or `false` if you do not want this file write.
- The keyword file has no special access control. Ensure the user running OpenClaw can read and write the selected path.
- Text normalization covers common bypass forms such as full-width characters and zero-width characters, but this is not an adversarial detection system. Use an HTTP provider for higher-risk environments.

### Fallback policy

- The default `fallbackOnError: "pass"` favors availability when a network or provider error occurs.
- High-compliance environments should use `fallbackOnError: "block"` and reliable provider/channel-specific configuration.

## Boundary

This plugin only checks inbound user message content. It does not replace OpenClaw host sandboxing, approvals, SSRF protection, secret management, or tool execution policies.
