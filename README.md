# copilot-api

[![npm version](https://img.shields.io/npm/v/copilot-api-node20?color=0969da&label=npm&logo=npm&logoColor=white)](https://www.npmjs.com/package/copilot-api-node20)
[![downloads](https://img.shields.io/npm/dm/copilot-api-node20?color=0969da&logo=npm&logoColor=white)](https://www.npmjs.com/package/copilot-api-node20)
![Node.js](https://img.shields.io/badge/node-%E2%89%A5%2020-339933?logo=nodedotjs&logoColor=white)
[![GitHub](https://img.shields.io/badge/github-johnib%2Fcopilot--api--issues-24292e?logo=github&logoColor=white)](https://github.com/johnib/copilot-api-issues)
![Closed Source](https://img.shields.io/badge/source-closed-6c737a)
[![License: MIT](https://img.shields.io/badge/license-MIT-22863a)](./LICENSE)

A proxy server that turns a GitHub Copilot subscription into fully compatible OpenAI and Anthropic API endpoints. Use Claude Code, Claude Cowork, Codex CLI, and any tool that speaks the API -- all through the Copilot plan you already pay for, no separate API keys required.

Want Claude Code on GPT and other non-Claude models? Pick one from your Copilot plan, or serve your own **Microsoft Foundry / Azure OpenAI** deployments with `--provider foundry`, signing in with an API key or Entra ID. Add `--fallback copilot` and Copilot answers whenever Foundry throttles, fails or times out. See [Microsoft Foundry](#microsoft-foundry--azure-openai).

```
  Clients                        Proxy                         Backends
  ───────                        ─────                         ────────

  Claude Code   ─┐                                        ┌─▶ GitHub Copilot API
  Claude Cowork ─┤  /v1/messages     ┌─────────────┐      │     Messages
  Codex CLI     ─┤───── or ─────────▶│ copilot-api │──────┤     Chat Completions
  OpenAI SDK    ─┤  /v1/chat/        └─────────────┘      │     Responses API
  curl          ─┤  completions                           │     Embeddings
  Any tool      ─┘                                        │
                                                          ├─▶ Microsoft Foundry /
                                                          │   Azure OpenAI
                                                          │   (--provider foundry)
                                                          └─▶ Web Search
                                                              (Tavily / Exa)
```

---

## Table of Contents

- [Key Features](#key-features)
- [Quick Start](#quick-start)
  - [Claude Code](#claude-code)
  - [Codex CLI](#codex-cli)
  - [Web Search](#web-search)
  - [Claude Cowork](#claude-cowork)
  - [Microsoft Foundry / Azure OpenAI](#microsoft-foundry--azure-openai)
- [API Endpoints](#api-endpoints)
- [CLI Reference](#cli-reference)
  - [Commands](#commands)
  - [start Flags](#start-flags)
  - [cowork-setup](#cowork-setup)
- [Telemetry](#telemetry)
- [Closed Source](#closed-source)
- [Notices](#notices)
- [Attribution](#attribution)
- [License](#license)

---

## Key Features

| Feature | Description |
|:--|:--|
| **Full Claude Code support** | All Claude models, streaming, extended thinking, and the complete **1M token context window** |
| **Claude Cowork support** | Run Claude's desktop app through Copilot — **one-command setup** via [Tailscale](https://tailscale.com) for zero-trust-hassle HTTPS |
| **Full Codex CLI support** | Complete **OpenAI Responses API** implementation purpose-built for Codex CLI |
| **Claude Code on non-Claude models** | GPT and other models from Copilot or Foundry, translated to the Anthropic API, with tools, streaming and keepalive pings while the model thinks |
| **Microsoft Foundry / Azure OpenAI** | Your own Azure deployments with `--provider foundry`: API key or Entra ID (`az login`, managed identity, service principal), no GitHub account needed |
| **Copilot fallback** | `--fallback copilot`: Copilot answers when Foundry rate-limits, errors or times out, and finishes a Claude Code turn on an OpenAI deployment whose stream fails before any output; then traffic moves back to Foundry |
| **Web search** | Optional web search for Claude Code and Codex sessions, via **Tavily and/or Exa** with automatic failover |
| **Dual API compatibility** | OpenAI Chat Completions, Anthropic Messages, and OpenAI Responses APIs side by side |
| **HTTPS with auto-generated certs** | Self-signed TLS certs with configurable SANs -- works across VMs, containers, and LAN |
| **Zero configuration** | Single command to start -- authenticates via GitHub device-code OAuth |

---

## Quick Start

**Prerequisites:** Node.js 20+ (or Bun) and an active GitHub Copilot subscription.

### Claude Code

Start the proxy with Claude Code mode:

```bash
npx copilot-api-node20@latest start --claude-code
```

On first run, you authenticate via GitHub device-code OAuth. Once authenticated, the proxy starts on `http://localhost:4141` and prints a ready-to-paste command. In another terminal:

```bash
ANTHROPIC_BASE_URL=http://localhost:4141 claude
```

To skip the interactive model picker, specify models upfront:

```bash
npx copilot-api-node20@latest start \
  --claude-code \
  --model claude-opus-4.6-1m \
  --small-model claude-haiku-4.5
```

### Codex CLI

Start the proxy with Codex mode:

```bash
npx copilot-api-node20@latest start --codex --model gpt-5.6-sol
```

Configure `~/.codex/config.toml`:

```toml
model = "gpt-5.6-sol"
model_provider = "local"

[model_providers.local]
name = "Local Server"
base_url = "http://localhost:4141/v1"
wire_api = "responses"
requires_openai_auth = false

[model_providers.local.http_headers]
Authorization = "Bearer dummy"
```

Then run:

```bash
codex
```

### Web Search

Web search tool calls (in both Claude Code and Codex sessions) are served by one
or more search providers. Configure providers via environment variables — the
proxy operates normally when none are set (web search is simply unavailable).

| Provider | Env var | Free tier |
| --- | --- | --- |
| [Tavily](https://tavily.com) | `TAVILY_API_KEY` | 1,000 searches/mo |
| [Exa](https://exa.ai) | `EXA_API_KEY` | 1,000 searches/mo |

Set one or both — web search works with a single provider, and configuring two
enables automatic failover.

```bash
TAVILY_API_KEY=<key> EXA_API_KEY=<key> npx copilot-api-node20@latest start --claude-code
```

When more than one provider is configured, searches run against them in order
and **fail over** to the next provider on error or timeout. Control the order
with `--web-search-provider` (comma-separated, primary first); it defaults to all
configured providers, Tavily first:

```bash
# Prefer Exa, fall back to Tavily
EXA_API_KEY=<key> TAVILY_API_KEY=<key> \
  npx copilot-api-node20@latest start --claude-code --web-search-provider exa,tavily
```

The `--tavily-api-key` flag is **deprecated** — it still works and is equivalent
to `TAVILY_API_KEY`, but prints a deprecation warning on startup. Prefer the
environment variables, which also keep keys out of the process list.

### Claude Cowork

Use Claude's desktop app ([Cowork](https://claude.ai/desktop)) with your Copilot subscription. Requires [Tailscale](https://tailscale.com) (free for personal use) for a real Let's Encrypt TLS certificate — no self-signed cert headaches.

**1. Install [Tailscale](https://tailscale.com/download)** and sign in. On macOS, use the [App Store or .pkg download](https://tailscale.com/download/mac) (not brew). See [docs/claude-cowork.md](./docs/claude-cowork.md) for Linux/Windows.

**2. Generate a TLS cert and configure the proxy:**

```bash
npx copilot-api-node20@latest cowork-setup install
npx copilot-api-node20@latest start --claude-cowork
```

**3.** Follow Anthropic's [Cowork 3P Installation guide](https://claude.com/docs/cowork/3p/installation). When prompted for the gateway URL, paste the Tailscale hostname from the startup banner (e.g. `https://your-machine.tail12345.ts.net:4141`), then **restart Cowork**.

Run `cowork-setup doctor` at any time to diagnose issues:

```bash
npx copilot-api-node20@latest cowork-setup doctor
```

See **[docs/claude-cowork.md](./docs/claude-cowork.md)** for the full setup guide and troubleshooting. For Cowork's own configuration docs, see [Claude Cowork 3P Installation](https://claude.com/docs/cowork/3p/installation).

### Microsoft Foundry / Azure OpenAI

Point the proxy at your own Azure resource instead of GitHub Copilot. No GitHub account or Copilot subscription is needed:

```bash
export FOUNDRY_ENDPOINT=https://<resource>.openai.azure.com
export FOUNDRY_API_KEY=<key>      # or omit it and `az login` for Entra ID
npx copilot-api-node20@latest start --provider foundry
```

Then launch Claude Code with a **deployment name** as the model:

```bash
ANTHROPIC_BASE_URL=http://localhost:4141 ANTHROPIC_AUTH_TOKEN=dummy \
  ANTHROPIC_DEFAULT_HAIKU_MODEL=<fast-deployment> \
  claude --model <deployment>
```

The model you ask for is the model you get: the proxy never renames or substitutes one. It routes by model family:

| Model family | Served by |
|:--|:--|
| Claude | Your resource's native `/anthropic/v1/messages`, forwarded byte for byte |
| OpenAI (`gpt-*`, `o*`, codex) | `/openai/v1/responses` (translated for Claude Code) |
| Others (DeepSeek, Grok, …) | `/openai/v1/chat/completions` |

**Sign-in:** by default, with `FOUNDRY_API_KEY` set the proxy uses the key. Without it, it uses Entra ID through the Azure SDK, so `az login`, a managed identity or a service principal all work (`FOUNDRY_AUTH=key|entra` picks one explicitly). The identity needs **Cognitive Services OpenAI User** (GPT) or **Azure AI User** (Claude) on the resource. See [Authentication](./docs/foundry.md#authentication).

**Have a Copilot subscription too?** Add `--fallback copilot`. When Foundry rate-limits (429), fails (408, 500, 502, 503 or 504), times out or drops the connection, Copilot answers the same request. For Claude Code on an OpenAI deployment, a Foundry stream that fails before any text or tool call reaches it is finished by Copilot as well. The model stays on Copilot for Foundry's `retry-after` (initially 10 seconds without one), then goes back. See [Copilot fallback](./docs/foundry.md#copilot-fallback).

Codex works with the usual `config.toml` (see [Codex CLI](#codex-cli)) with `model` set to a deployment name. See **[docs/foundry.md](./docs/foundry.md)** for configuration, routing details, reasoning effort, and troubleshooting (including `NODE_EXTRA_CA_CERTS` behind TLS-inspecting corporate proxies).

### First run & troubleshooting

On a successful start the proxy runs a **one-shot self-test generation** and prints the result, so you know immediately whether everything works end-to-end — you'll see something like:

```
Self-test passed — generation works via claude-haiku-4.5 (612ms). Point your client at http://127.0.0.1:4141
```

If it fails, the line tells you why (e.g. no Copilot entitlement, premium quota exhausted, model not supported on your plan). It spends one generation per boot (preferring a cheap, non-premium model when available); disable it with `--no-smoke-test` (or `COPILOT_API_NO_SMOKE_TEST=1`) — e.g. in CI or if you restart frequently.

**Stuck? Run the doctor.** It checks your token, auth, Copilot entitlement, model catalog, and one real generation, then tells you exactly what to fix:

```bash
npx copilot-api-node20@latest doctor
```

Quick smoke test from another terminal (no client needed):

```bash
curl http://localhost:4141/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"claude-haiku-4.5","messages":[{"role":"user","content":"ping"}],"max_tokens":1}'
```

| Symptom | Likely cause | Fix |
|:--|:--|:--|
| Client connects but nothing happens | Client not pointed at the proxy | Set the base URL (see below); run `doctor` |
| `400 ... model is not supported` | Premium model not entitled on your plan | Use `claude-haiku-4.5`, or check entitled models with `doctor` |
| `402 quota_exceeded` | Monthly premium quota used up | Switch to a non-premium model (haiku) or wait for reset |
| `404` on `/v1/v1/...` | Base URL already ends in `/v1` | Drop the trailing `/v1` from your client's base URL |
| Works locally but not from Docker/another machine | Bound to localhost | Start with `--hostname 0.0.0.0` |

**Client base URLs** — Claude Code: `ANTHROPIC_BASE_URL=http://localhost:4141`; Codex: `OPENAI_BASE_URL=http://localhost:4141/v1`; OpenAI SDKs: `baseURL: "http://localhost:4141/v1"`. See **[docs/getting-started.md](./docs/getting-started.md)** for the full per-client setup, the entitled-model matrix, and detailed troubleshooting.

---

## API Endpoints

All endpoints are served at `http://localhost:4141` by default (`https://` when `--https` or `--claude-cowork` is used). The server binds to `127.0.0.1` (localhost only) by default. To expose to LAN/network/Docker, pass `--hostname 0.0.0.0`.

| Method | Path | Purpose |
|:--|:--|:--|
| `GET` | `/` | Basic server-running response |
| `GET` | `/health` | Proxy, token, and upstream health status |
| `POST` | `/v1/chat/completions` | OpenAI Chat Completions |
| `POST` | `/chat/completions` | OpenAI Chat Completions without the `/v1` prefix |
| `POST` | `/v1/responses` | OpenAI Responses API |
| `POST` | `/responses` | OpenAI Responses API without the `/v1` prefix |
| `POST` | `/v1/messages` | Anthropic Messages |
| `POST` | `/v1/messages/count_tokens` | Anthropic Token Counting |
| `POST` | `/v1/embeddings` | OpenAI Embeddings |
| `POST` | `/embeddings` | OpenAI Embeddings without the `/v1` prefix |
| `GET` | `/v1/models` | OpenAI Models |
| `GET` | `/models` | OpenAI Models without the `/v1` prefix |
| `GET` | `/usage` | GitHub Copilot usage and quota information |
| `GET` | `/token` | Copilot token debug endpoint; disabled unless started with `--show-token` |

---

## CLI Reference

### Commands

| Command | Description |
|:--|:--|
| `copilot-api start` | Start the proxy server |
| `copilot-api auth` | Authenticate with GitHub |
| `copilot-api check-usage` | Display Copilot usage quota (credits, spend, burn rate) |
| `copilot-api check-usage --watch` | Same, refreshed on an interval (`--interval <seconds>`, default 60, min 10) |
| `copilot-api doctor` | Diagnose your setup end-to-end (token, auth, entitlement, models, one real generation) and print exact fixes |
| `copilot-api debug` | Print diagnostic information |
| `copilot-api cowork-setup` | Manage Claude Cowork OS-level setup |

### `start` Flags

> `--claude-code` and `--codex` are mutually exclusive.

<details>
<summary>Full flag reference</summary>

<br>

| Flag | Alias | Default | Description |
|:--|:--|:--|:--|
| `--provider` | | `copilot` | Upstream: `copilot`, or `foundry` for Microsoft Foundry / Azure OpenAI (reads `FOUNDRY_ENDPOINT`, plus `FOUNDRY_API_KEY` or an Entra ID login; see [docs/foundry.md](./docs/foundry.md)). Also `COPILOT_API_PROVIDER` |
| `--fallback` | | | With `--provider foundry`: `copilot` answers from GitHub Copilot when Foundry rate-limits, fails or times out (and, for Claude Code on an OpenAI deployment, when a Foundry stream fails before any output), and keeps that model there for Foundry's `retry-after`, or initially 10 s without one (see [docs/foundry.md](./docs/foundry.md#copilot-fallback)). Also `COPILOT_API_FALLBACK` |
| `--port` | `-p` | `4141` | Port to listen on |
| `--hostname` | `-H` | `127.0.0.1` | Hostname/IP to bind to. Use `0.0.0.0` for LAN/Docker exposure |
| `--claude-code` | `-c` | | Enable Claude Code compatibility mode |
| `--codex` | | | Enable Codex CLI compatibility mode |
| `--model` | `-m` | | Primary model to use |
| `--small-model` | `-s` | | Lightweight model for fast tasks (Claude Code) |
| `--manual` | | `false` | Require manual approval before forwarding requests |
| `--rate-limit` | `-r` | | Minimum seconds between requests |
| `--wait` | `-w` | `false` | Wait instead of returning an error when the configured rate limit is hit |
| `--timeout` | `-t` | `600000` | Request timeout in ms (default 10 min). A streamed Claude Code turn on a non-Claude model waits at most 4 min for the upstream to start answering |
| `--connect-timeout` | | `15000` | Connectivity probe timeout in ms; overrides `COPILOT_CONNECT_TIMEOUT_MS` |
| `--account-type` | `-a` | `individual` | `individual`, `business`, or `enterprise` |
| `--github-token` | `-g` | | Provide a GitHub token directly (skips OAuth) |
| `--show-token` | | | Display tokens on fetch and refresh, and enable the local `/token` debug endpoint |
| `--error-reporting` | | `true` | Send sanitized server-error reports. Disable with `--no-error-reporting` or `COPILOT_API_ERROR_REPORTING=0` |
| `--smoke-test` | | `true` | Run one self-test generation after startup and print PASS/FAIL. Disable with `--no-smoke-test` or `COPILOT_API_NO_SMOKE_TEST=1` |
| `--tavily-api-key` | | | **Deprecated** — prefer `TAVILY_API_KEY` env var. Tavily API key for web search |
| `--web-search-provider` | | | Web search providers in primary→fallback order, e.g. `exa,tavily` (keys via `TAVILY_API_KEY` / `EXA_API_KEY`) |
| `--verbose` | `-v` | | Enable debug-level logging |
| `--https` | | | Serve over HTTPS using a self-signed cert |
| `--https-cert` | | | Path to a PEM-encoded TLS certificate (use with `--https-key`) |
| `--https-key` | | | Path to a PEM-encoded TLS private key (use with `--https-cert`) |
| `--https-hosts` | | | Extra hostnames or IPs to include in the cert's SANs (comma-separated) |
| `--claude-cowork` | | | Auto-detect Tailscale cert and enable HTTPS for Cowork. Run `cowork-setup install` first. See [docs/claude-cowork.md](./docs/claude-cowork.md) |

</details>

### `cowork-setup`

Manage [Tailscale](https://tailscale.com)-based HTTPS for Claude Cowork.

| Subcommand | Description |
|:--|:--|
| `cowork-setup doctor` | Diagnose Tailscale + cert status |
| `cowork-setup install` | Generate a Tailscale TLS certificate for Cowork |
| `cowork-setup revert` | Remove the generated cert files |

Requires Tailscale installed and connected (`tailscale up`). See [docs/claude-cowork.md](./docs/claude-cowork.md).

Platform support: **macOS**, **Linux**, **Windows** — anywhere Tailscale runs.

---

## Telemetry

This tool collects anonymous usage telemetry via [OpenTelemetry](https://opentelemetry.io/). Only operational metrics are collected -- request counts, latency, model usage, and error rates. No prompts, completions, or personal information is ever collected.

When a server-side error occurs, an anonymous, sanitized error report (error type, status code, model, route, and runtime/version metadata -- never prompts, completions, headers, or tokens) is automatically sent to help diagnose and fix bugs. Disable it with `--no-error-reporting` or `COPILOT_API_ERROR_REPORTING=0`.

The startup banner's "Recent Posts" list is refreshed in the background at most once a day from `https://barazany.dev/posts.json` (a plain GET, nothing sent). It never delays startup and falls back to a cached or bundled list when offline. Disable it with `COPILOT_API_NO_BLOG=1` (`COPILOT_API_SKIP_UPDATE_CHECK=1` also disables it).

**By using copilot-api, you agree to this telemetry data collection.**

---

## Closed Source

The source code is not publicly available.

This fork represents ~9 months of sustained engineering work -- bug fixes, new features, and hardening that go well beyond the original project. Keeping it closed-source protects that investment and ensures a single, well-maintained distribution.

The original open-source project is available at [ericc-ch/copilot-api](https://github.com/ericc-ch/copilot-api).

---

## Notices

This project relies on reverse-engineered, undocumented GitHub APIs. It is **not affiliated with or endorsed by GitHub, Microsoft, Anthropic, or OpenAI** and may break without notice.

- Use `--rate-limit` to throttle requests and reduce abuse-detection risk.
- Review the [GitHub Acceptable Use Policies](https://docs.github.com/en/site-policy/acceptable-use-policies/github-acceptable-use-policies) and [GitHub Copilot Terms](https://docs.github.com/en/site-policy/github-terms/github-terms-for-additional-products-and-features#github-copilot) before use.
- This software is provided as-is with no warranty.

---

## Attribution

Originally created by **Erick Christian** -- [ericc-ch/copilot-api](https://github.com/ericc-ch/copilot-api). His work built the foundation: authentication flow, API translation layer, and streaming implementation.

This fork has diverged significantly over ~9 months with extensive bug fixes and major new features -- Anthropic Messages API, OpenAI Responses API, Claude Code mode, Codex CLI support, web search integration, usage telemetry, and more. It is maintained as a separate project by [johnib](https://github.com/johnib).

---

## License

[MIT](./LICENSE)
