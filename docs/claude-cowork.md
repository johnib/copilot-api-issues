# Claude Cowork Setup Guide

Use [Claude Cowork](https://claude.ai/desktop) with your GitHub Copilot
subscription — all inference goes through the `copilot-api` proxy, no separate
API keys needed.

This guide uses [Tailscale](https://tailscale.com) to provide a real Let's
Encrypt TLS certificate. Tailscale is free for personal use and takes ~2
minutes to set up. This is the recommended approach because it avoids all
the trust-store and certificate-management headaches of self-signed certs.

For Cowork's own configuration docs, see
[Claude Cowork 3P Installation](https://claude.com/docs/cowork/3p/installation).

---

## Prerequisites

1. **Node.js 20+** (or Bun) and an active **GitHub Copilot** subscription
2. **[Tailscale](https://tailscale.com)** — free for personal use. Install and sign in:

   | OS | Install | Guide |
   |---|---|---|
   | macOS | [Download](https://tailscale.com/download/mac) (App Store or .pkg — **not** `brew`) | [Docs](https://tailscale.com/docs/install/mac) |
   | Linux | `curl -fsSL https://tailscale.com/install.sh \| sh` then `sudo tailscale up` | [Docs](https://tailscale.com/docs/install/linux) |
   | Windows | [Download](https://tailscale.com/download/windows) | [Docs](https://tailscale.com/docs/install/windows) |

   On macOS, open the Tailscale app from Applications and sign in.
   On Linux, run `sudo tailscale up` after installing.

---

## Setup

### 1. Generate a TLS certificate

```bash
copilot-api cowork-setup install
```

This runs `tailscale cert` under the hood, requesting a real Let's Encrypt
certificate for your machine's Tailscale hostname (e.g.
`your-machine.tail12345.ts.net`). The cert is stored at
`~/.local/share/copilot-api/certs/tailscale/`.

### 2. Start the proxy

```bash
copilot-api start --claude-cowork
```

The `--claude-cowork` flag auto-detects the Tailscale cert and enables HTTPS.
The startup banner shows the URL to use.

### 3. Configure Cowork

Follow Anthropic's official setup guide:
**[Claude Cowork 3P Installation](https://claude.com/docs/cowork/3p/installation)**

When prompted for the gateway URL, use your Tailscale hostname + port:

```
https://your-machine.tail12345.ts.net:4141
```

Replace `your-machine.tail12345.ts.net` with your actual Tailscale hostname
(shown by `tailscale status --self` or in the proxy startup banner). The API
key can be any non-empty string — the proxy ignores it.

Then **restart Cowork** (Cmd+Q and reopen).

---

## How it works

```
Cowork (Electron + CLI + VM)
       │
       │  https://your-machine.tail12345.ts.net:4141
       │  (real Let's Encrypt cert — trusted everywhere)
       ▼
  copilot-api proxy
       │
       │  GitHub Copilot API
       ▼
  Chat Completions
```

**Why Tailscale?** Cowork requires HTTPS for inference gateways. The bundled
`claude` CLI (a Bun-compiled binary) has a [known
issue](https://github.com/oven-sh/bun/issues) where it ignores
`NODE_EXTRA_CA_CERTS`, `BUN_CA_BUNDLE_PATH`, `SSL_CERT_FILE`, and the system
keychain — so self-signed certs require disabling TLS verification globally
via `NODE_TLS_REJECT_UNAUTHORIZED=0`.

Tailscale sidesteps this entirely: `tailscale cert` issues a real Let's
Encrypt certificate for your Tailscale hostname. Every TLS client trusts
Let's Encrypt out of the box — no OS-level trust hacks, no env vars, no
per-platform setup. Tailscale's MagicDNS also resolves the hostname from
the Mac host, the inference VM, and any LAN device, solving the
"single URL for multiple actors" problem.

---

## Diagnostics

Run `cowork-setup doctor` to check that everything is in order:

```bash
copilot-api cowork-setup doctor
```

Example output:
```
  ─────────────────────────────────────────────────────────────
   Claude Cowork Setup — Diagnostic
  ─────────────────────────────────────────────────────────────

  ✅ tailscale-installed
  ✅ tailscale-running  your-machine.tail12345.ts.net
  ✅ tailscale-cert     ~/.local/share/copilot-api/certs/tailscale/cert.pem
  ✅ ready              https://your-machine.tail12345.ts.net:<port>

  All checks passed.
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `tailscale not found in PATH` | Install Tailscale — see Prerequisites above |
| `Run "tailscale up" to connect` | Tailscale is installed but not connected. Run `tailscale up` |
| `Tailscale cert not found` | Run `copilot-api cowork-setup install` |
| Cowork says "taking a while to connect" | Verify the hostname + port are correct. Try `curl https://<hostname>:<port>/v1/models` |
| Cert expired (LE certs last 90 days) | Re-run `copilot-api cowork-setup install` |
| Proxy says `http://` instead of `https://` | Make sure you're using `--claude-cowork` (not just `--https`) |

---

## Revert

```bash
copilot-api cowork-setup revert
```

This removes the generated cert files. Update Cowork's `inferenceGatewayBaseUrl`
back to the default.

---

## Self-signed fallback (without Tailscale)

<details>
<summary>For users who can't install Tailscale</summary>

You can use `--https` with a self-signed cert instead:

```bash
copilot-api start --claude-cowork --https
```

This auto-generates a self-signed cert, but Cowork's bundled CLI won't trust
it. You'll need to:

1. Install the cert into the macOS keychain:
   ```bash
   sudo security add-trusted-cert -d -r trustRoot \
     -k /Library/Keychains/System.keychain \
     ~/.local/share/copilot-api/certs/cert.pem
   ```

2. Disable TLS verification for the bundled CLI:
   ```bash
   launchctl setenv NODE_TLS_REJECT_UNAUTHORIZED 0
   ```

3. Restart Cowork.

**Caveats:**
- `NODE_TLS_REJECT_UNAUTHORIZED=0` disables TLS verification for *every*
  process launched by launchd, not just Cowork.
- `launchctl setenv` doesn't survive reboots.
- If your LAN IP changes, you need to regenerate the cert.

The Tailscale approach has none of these problems.

</details>
