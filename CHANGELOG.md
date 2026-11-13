# Changelog

User-facing changes to [`copilot-api-node20`](https://www.npmjs.com/package/copilot-api-node20).
Full list: [Releases](https://github.com/johnib/copilot-api-issues/releases).

## 0.48.4 (2026-11-13)

### Bug Fixes

* set the Foundry fallback headers cap by request type

## 0.48.3 (2026-11-13)

### Bug Fixes

* cancel the upstream error body read on a client disconnect and record a client cancel as a client abort
* carry tool search results and tool changes through the translators
* forward the client prompt_cache_key on Copilot /responses
* let short commands exit by unref of the completion-log cleanup timer
* print the package version for --version and exit with code 0
* stop false offline probes and needless or hung Copilot token exchanges

## 0.48.2 (2026-11-13)

### Bug Fixes

* give the local --rate-limit rejection its own error type
* keep CLI argument values out of process.command_args telemetry
* record stream failure type, forward gaps, pings and reasoning tokens on passthrough streams
* record the inner transport cause and the real exception type
* stop the root span export for cached /v1/models answers
* tag direct 4xx answers with copilot_api.error_type

## 0.48.1 (2026-11-12)

### Bug Fixes

* give api_error to an upstream 5xx with a flat message body

## 0.48.0 (2026-11-12)

### Bug Fixes

* keep the Foundry backoff until a full quiet window
* mark repository test telemetry explicitly

### Features

* answer Responses input_tokens with a local token count

## 0.47.11 (2026-11-11)

### Bug Fixes

* distinguish HTTP/2 peer faults from local closes
* finalize Responses streams at their terminal event
* preserve accepted HTTP/2 streams after graceful GOAWAY
* preserve chat cache usage in Messages responses
* stop filing [auto] issues for known upstream faults

## 0.47.10 (2026-11-10)

### Bug Fixes

* abort the self-test request when its timeout fires
* answer 503 images_unavailable when the image budget cannot hold any request
* bound online connectivity probe telemetry
* cancel upstream when the client disconnects on the Copilot paths
* classify HTTP error logs by status
* enforce the body size limit on chunked and understated bodies
* move GPT token counts to a bounded worker
* normalize Copilot embedding string inputs
* read package versions from filesystem paths
* report unavailable self-test candidates as inconclusive
* return JSON errors for unsupported routes
* skip cgroup directories that a nested or stacked mount covers

## 0.47.9 (2026-11-09)

### Bug Fixes

* bound bridged Copilot user identifiers
* reject declared oversized bodies before logger capture
* reject malformed Responses tool shapes
* retry Foundry 507 once through Copilot

## 0.47.8 (2026-11-09)

### Bug Fixes

* cut image uploads that pass a total body deadline
* record a late upload failure on the image attempt span

## 0.47.7 (2026-11-09)

### Bug Fixes

* bound held multipart chunks, keep epilogues, abort rate-limit wait
* do not keep the request Context alive while a stream is open
* follow same-origin upstream redirects for image calls
* pick the cgroup mount that holds the process membership

## 0.47.6 (2026-11-08)

### Bug Fixes

* complete h2 error mocks in Copilot service tests
* support bulk attributes in shared span mock

## 0.47.5 (2026-11-07)

### Bug Fixes

* classify local thread discovery rate limits as local in telemetry
* remove the approval abort listener when the prompt answers first

## 0.47.4 (2026-11-07)

### Bug Fixes

* honour Retry-After 0 and reject unterminated multipart params on images
* return 503 with Retry-After for HTTP/2 NGHTTP2_ENHANCE_YOUR_CALM stream errors

## 0.47.3 (2026-11-07)

### Bug Fixes

* take an images answer of exactly the spool limit and keep one deadline across 429 retries

## 0.47.2 (2026-11-07)

### Bug Fixes

* a complete JSON image body claims no drain slot on an error reply
* name the domain filter and value when Exa rejects a domain
* offer only chat-type models to pickers and self-test
* treat Bun fetch connection errors as transient transport errors
* wrap non-object upstream error bodies on OpenAI routes

## 0.47.1 (2026-11-06)

### Bug Fixes

* request full replay for unsupported message threads

## 0.47.0 (2026-11-06)

### Features

* support OpenAI Decisions API for gpt-6-luna-decisions

## 0.46.0 (2026-11-03)

### Features

* serve 30 concurrent image requests in a 1.5 GB container

## 0.45.1 (2026-11-02)

### Bug Fixes

* self-test skips models the account cannot use

## 0.45.0 (2026-11-01)

### Features

* add OpenAI Images API passthrough for gpt-image models

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
