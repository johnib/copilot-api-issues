# Changelog

User-facing changes to [`copilot-api-node20`](https://www.npmjs.com/package/copilot-api-node20).
Full list: [Releases](https://github.com/johnib/copilot-api-issues/releases).

## 0.44.2 (2026-10-30)

### Bug Fixes

* keep OpenTelemetry on the 2.x line (0.44.1 exported no spans)

## 0.44.1 (2026-10-27)

### Bug Fixes

* raise hono floor to 4.12.23 (runtime Context export), closes [copilot-api-issues#14](https://github.com/copilot-api-issues/issues/14)

## 0.44.0 (2026-10-23)

### Features

* Microsoft Foundry provider (API key or Entra ID) with GitHub Copilot fallback
* `--provider foundry`: use your own Microsoft Foundry / Azure OpenAI deployments, signing in with an API key or Entra ID (az login, managed identity, service principal)
* `--fallback copilot`: GitHub Copilot answers when Foundry rate-limits, fails or times out, then traffic moves back to Foundry

### Changes for everyone (Copilot mode too)

* Claude Code on GPT models: a ping every 15 s while the model thinks; an error Claude Code retries if no answer starts within 4 minutes; a stream that goes silent for 5 minutes is cut
* "Context too long" errors from GPT models now read "prompt is too long", so Claude Code compacts instead of failing
* 429s show in the terminal without `--verbose`, and `retry-after` is passed on to the client
* `count_tokens` uses GPT's own tokenizer for GPT models
* Images inside tool results now reach GPT models
* Memory-leak fixes for long-running proxies
