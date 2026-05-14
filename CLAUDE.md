# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Standalone external plugin for OpenClaw — `guardrail-bridge`. Hooks the OpenClaw host's `before_dispatch` event to inspect inbound user messages and optionally block them based on two connector strategies.

## Common commands

```bash
npm install
npm test                       # all vitest suites
npm test -- src/normalize.test.ts   # one test file
npm test -- -t "blacklist hot reload"  # filter by name
npm run typecheck              # tsc --noEmit
npm run build                  # emit runtime JS to dist/
npm pack                       # build and create local install tarball
```

Published/local archive installs ship compiled runtime output under `dist/`. `openclaw.extensions` keeps the TypeScript source entry and `openclaw.runtimeExtensions` points OpenClaw at `./dist/index.js` for packaged runtime loading.

## Architecture

### Loading & lifecycle

- `index.ts` exports `default plugin = { id, name, description, register(api) }`. `api: OpenClawPluginApi` is host-injected.
- `register()` reads `api.pluginConfig`, computes one `EffectiveChannelConfig` per channel + a global one (via `src/config.ts:resolveChannelConfig`), creates the connector backends, and binds a single `api.on("before_dispatch", ...)` hook that dispatches per-channel.
- Connector instances are **deduplicated** by hashed config key, so two channels using identical settings share the same backend (and hot-reload watcher).
- Cleanup goes through `api.registerService({ stop })` calling collected disposables.

### Two connector types

All connectors converge on a single `BackendFn = (text, context) => Promise<GuardrailsDecision>` (see `src/config.ts`). The host-facing handler in `src/handler.ts` maps `{ action: "pass" }` → `{ handled: false }` and `{ action: "block" }` → `{ handled: true, text: blockMessage }`.

- **Blacklist** (`src/builtin-blacklist-connector.ts`): Aho-Corasick (`@monyone/aho-corasick`) over a keyword file. Default path is `<openclaw-state-dir>/guardrail-bridge/keywords.txt`, seeded from `assets/keywords.default.txt` on first use. Optional `fs.watch`-based hot reload with debounce. Text passes through `src/normalize.ts` first (NFC, fullwidth→halfwidth, zero-width strip, optional lowercase).
- **HTTP** (`src/http-connector.ts` + `src/providers/`): provider registry pattern. Built-ins (`dknownai`, `dknownai-cn`, `secra`, `hidylan`) cannot be overridden; custom adapters register via `registerHttpProvider()` (re-exported from `api.ts`). All HTTP calls go through `fetchWithSsrFGuard` from the SDK so SSRF policy and timeouts are enforced by the host. Adapters implement `GuardrailsProviderAdapter` (`src/provider-types.ts`).

### SDK contact surface

This plugin imports from the OpenClaw plugin SDK only through the declared public subpaths below:

| Subpath                                         | Symbol                   | Used in                              |
| ----------------------------------------------- | ------------------------ | ------------------------------------ |
| `openclaw/plugin-sdk/core`                      | `OpenClawPluginApi`      | `index.ts`                           |
| `openclaw/plugin-sdk/runtime-secret-resolution` | `resolveSecretRefValues` | `index.ts`                           |
| `openclaw/plugin-sdk/state-paths`               | `resolveStateDir`        | `src/builtin-blacklist-connector.ts` |
| `openclaw/plugin-sdk/ssrf-runtime`              | `fetchWithSsrFGuard`     | `src/providers/*.ts`                 |

These are bare specifiers. At runtime, the OpenClaw host (the `openclaw` npm package) provides them via its subpath `exports`. For local typecheck, this repo declares `openclaw` in `peerDependencies` (runtime) and `devDependencies` (so `npm install` pulls real types into `node_modules/openclaw/dist/plugin-sdk/*.d.ts`). No local SDK stub is maintained — when the SDK signature changes, update the call sites and rerun `npm run typecheck`. See `docs/plugin-sdk/sdk-migration.md` for documented deprecation/migration paths.

`src/handler.ts` deliberately defines its own `BeforeDispatchEvent` / `BeforeDispatchContext` / `BeforeDispatchResult` types to avoid coupling to non-public SDK paths — preserve this pattern.

### Configuration model

`src/config.ts` is the single source for config types and `resolveConfig` / `resolveChannelConfig`. Per-channel overrides do **partial merge** for object fields (`http`, `blacklist`) and **direct overwrite** for scalars (`blockMessage`, `fallbackOnError`, `timeoutMs`). The auto-detection rule when `connector` is empty: pick whichever of http/blacklist looks configured. Channels can independently enable a connector even when the global one is empty.

The runtime `configSchema` lives in `openclaw.plugin.json` — keep that schema and the TypeScript types in `src/config.ts` in sync.

### Tests

Each `src/*.ts` has a co-located `*.test.ts`. The top-level `index.test.ts` is the integration suite for the plugin entry. Tests mock SDK subpaths with `vi.mock("openclaw/plugin-sdk/...", ...)` — see `src/builtin-blacklist-connector.test.ts` for the pattern.

## Important conventions

- ESM with NodeNext resolution. `.ts` source uses `.js` import suffixes (`from "./config.js"`) — the standard for TypeScript ESM emitted as Node ESM. Don't rewrite to `.ts` extensions.
- The plugin entry in `index.ts` is the legacy plain-object shape. Preserve it unless explicitly migrating the entry API.
- Built-in HTTP provider names (`dknownai`, `dknownai-cn`, `secra`, `hidylan`) are reserved — `registerHttpProvider` rejects collisions.
- `package.json` declares `openclaw.compat` and `openclaw.build`; SDK is **not** a runtime dependency.

## Operational guardrails

Hard rules for working in this repository.

### Never bypass signing or hooks without explicit user request

Never add `--no-verify`, `--no-gpg-sign`, `-c commit.gpgsign=false`, `-c core.hooksPath=/dev/null`, or any equivalent bypass flag to a git command unless the user has explicitly asked for that bypass in the current turn.

If a commit fails on a hook or signing check, investigate the root cause and tell the user what's blocking — don't paper over it.

### Keep SecretRef responsibilities clear

- OpenClaw owns SecretRef management. This plugin should consume SecretRef values through the public SDK, not document or reimplement the platform secret-management system.
- Plugin docs should show only the minimal recommended `env` SecretRef example and link to the official OpenClaw SecretRef documentation for other sources and setup details.
- Never write real API keys, local tokens, or private machine-specific secret values into repository docs, tests, fixtures, examples, or release notes.

### API key configuration rules

- Prefer SecretRef object values for `http.apiKey`.
- Plaintext `apiKey` remains a compatibility path only, and docs must label it as not recommended for production.
- HTTP provider examples should keep `fallbackOnError` at the default `"pass"` unless a specific security scenario explicitly requires `"block"`.

### Documentation sync rules

- Check English and Chinese docs together when changing user-facing behavior: `README.md`, `README-zh.md`, `docs/usage.md`, and `docs/usage-zh.md`.
- When changing config fields, provider behavior, defaults, install identifiers, or schema constraints, also check README, usage docs, and manifest schema docs for drift.
- Do not describe OpenClaw platform-level capabilities as plugin-owned behavior.

### Local validation rules

- For SecretRef, provider, or config schema changes, run at least `npm test` and `npm run typecheck`.
- Before packaging or release, run `npm run build` and `npm pack`.
- Local OpenClaw validation may use `~/.openclaw/openclaw.json`, but local secrets, tokens, and private paths must stay out of committed files and public documentation.

### HTTP provider secret handling

- Provider initialization must not receive or retain plaintext API keys.
- Resolve API keys at the `check()` boundary so plaintext secrets have the shortest practical residency time.
- Adapter deduplication keys must not include `apiKey`, preventing secret material from entering long-lived cache keys or log-like values.

### SDK import rules

- Only import OpenClaw SDK APIs from public exported subpaths.
- Before adding a new SDK import, verify the symbol is exported by the `openclaw` package exports and type declarations.
- Keep the SDK contact surface table in this file updated whenever SDK imports change.

### Release checklist

- Confirm `package.json` version, `openclaw.compat`, README install identifiers, and npm package names are aligned before release.
- Confirm the packed archive excludes development docs, temporary test files, local OpenClaw config, and real secrets.
- Check `git status` before release work and avoid overwriting unrelated user changes.

### Git commit messages

- Do not add `Co-Authored-By: Claude` or similar AI attribution lines to commit messages.
- Unless the user explicitly asks for it, do not include generated-by tooling identifiers in commit messages.
- Keep commit messages focused on the purpose and impact of the repository change.

