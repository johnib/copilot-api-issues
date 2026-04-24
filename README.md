# copilot-api

[![npm version](https://img.shields.io/npm/v/copilot-api-node20?color=0969da&label=npm&logo=npm&logoColor=white)](https://www.npmjs.com/package/copilot-api-node20)
[![downloads](https://img.shields.io/npm/dm/copilot-api-node20?color=0969da&logo=npm&logoColor=white)](https://www.npmjs.com/package/copilot-api-node20)
![Node.js](https://img.shields.io/badge/node-%E2%89%A5%2020-339933?logo=nodedotjs&logoColor=white)
[![GitHub](https://img.shields.io/badge/github-johnib%2Fcopilot--api--issues-24292e?logo=github&logoColor=white)](https://github.com/johnib/copilot-api-issues)
![Closed Source](https://img.shields.io/badge/source-closed-6c737a)
[![License: MIT](https://img.shields.io/badge/license-MIT-22863a)](./LICENSE)

A proxy server that turns a GitHub Copilot subscription into fully compatible OpenAI and Anthropic API endpoints. Use Claude Code, Claude Cowork, Codex CLI, and any tool that speaks the API -- all through the Copilot plan you already pay for, no separate API keys required.

```
  Clients                    Proxy                     Backends
  ───────                    ─────                     ────────

  Claude Code  ─┐                                 ┌─▶ GitHub Copilot API
  Claude Cowork─┤  Anthropic    ┌────────────┐    │     Chat Completions
  Codex CLI    ─┤─────or───────▶│ copilot-api │───┤     Responses API
  OpenAI SDK   ─┤  OpenAI       └────────────┘    │     Embeddings
  curl         ─┤  format                         │
  Any tool     ─┘                                 └─▶ Tavily
                                                       Web Search
```

---

## Table of Contents

- [Key Features](#key-features)
- [Quick Start](#quick-start)
  - [Claude Code](#claude-code)
  - [Codex CLI](#codex-cli)
  - [Web Search](#web-search)
  - [Claude Cowork](#claude-cowork)
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
| **Web search** | Optional **Tavily-powered web search** available to both Claude Code and Codex sessions |
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
npx copilot-api-node20@latest start --codex --model gpt-5.3-codex
```

Configure `~/.codex/config.toml`:

```toml
model = "gpt-5.3-codex"
model_provider = "local"

[model_providers.local]
name = "Local Server"
base_url = "http://localhost:4141/v1"
env_key = "OPENAI_API_KEY"
```

Then run:

```bash
OPENAI_API_KEY=dummy codex
```

### Web Search

Provide a [Tavily](https://tavily.com) API key to enable web search tool calls in both Claude Code and Codex sessions:

```bash
npx copilot-api-node20@latest start --claude-code --tavily-api-key <key>
```

Alternatively, set `TAVILY_API_KEY` as an environment variable. The proxy operates normally when no key is provided -- web search is simply unavailable.

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

---

## API Endpoints

All endpoints are served at `http://localhost:4141` by default (`https://` when `--https` or `--claude-cowork` is used).

| Method | Path | Compatibility |
|:--|:--|:--|
| `POST` | `/v1/chat/completions` | OpenAI Chat Completions |
| `POST` | `/v1/responses` | OpenAI Responses API |
| `POST` | `/v1/messages` | Anthropic Messages |
| `POST` | `/v1/messages/count_tokens` | Anthropic Token Counting |
| `POST` | `/v1/embeddings` | OpenAI Embeddings |
| `GET` | `/v1/models` | OpenAI Models |
All OpenAI-compatible routes are also available without the `/v1/` prefix.

---

## CLI Reference

### Commands

| Command | Description |
|:--|:--|
| `copilot-api start` | Start the proxy server |
| `copilot-api auth` | Authenticate with GitHub |
| `copilot-api check-usage` | Display Copilot usage quota |
| `copilot-api debug` | Print diagnostic information |
| `copilot-api cowork-setup` | Manage Claude Cowork OS-level setup |

### `start` Flags

> `--claude-code` and `--codex` are mutually exclusive.

<details>
<summary>Full flag reference</summary>

<br>

| Flag | Alias | Default | Description |
|:--|:--|:--|:--|
| `--port` | `-p` | `4141` | Port to listen on |
| `--claude-code` | `-c` | | Enable Claude Code compatibility mode |
| `--codex` | | | Enable Codex CLI compatibility mode |
| `--model` | `-m` | | Primary model to use |
| `--small-model` | `-s` | | Lightweight model for fast tasks (Claude Code) |
| `--rate-limit` | `-r` | | Minimum seconds between requests |
| `--timeout` | `-t` | `600000` | Request timeout in ms (default 10 min) |
| `--account-type` | `-a` | `individual` | `individual`, `business`, or `enterprise` |
| `--github-token` | `-g` | | Provide a GitHub token directly (skips OAuth) |
| `--show-token` | | | Display tokens on fetch and refresh |
| `--tavily-api-key` | | | Tavily API key for web search |
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
