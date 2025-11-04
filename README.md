<p align="center">
  <img src="docs/assets/logo.svg" alt="aiproxy logo" width="240">
</p>
<p align="center">
  <a href="https://aiproxy.dev"><code>aiproxy.dev</code></a>
</p>

**aiproxy** is a proxy for LLM provider APIs. For now it acts as either a forward proxy or a reverse proxy, and very soon it will also support transparent proxying. It sits between agents and upstream providers, can intercept TLS to observe/modify traffic, and streams rich telemetry to a terminal UI.

## Installation

```bash
curl -fsSL https://aiproxy.dev/install | bash
```

## Features

aiproxy is free to use during the public beta; we'll announce paid plans on [aiproxy.dev](https://aiproxy.dev) ahead of launch.

- Monitor agent chats, token usage, billing, and latencies transparently.
- MITM-aware HTTP proxy that handles CONNECT tunnels, protocol detection, and per-provider request routing
- A reverse proxy for OpenAI and Anthropic traffic.
- Dynamic TLS interception with on-the-fly certificates signed by a configurable CA
- Supported agents: Claude Code, OpenAI Codex, GitHub Copilot, Cursor, and ChatGPT. More to come!
- A terminal user interface for monitoring the traffic.
- Coming soon: centralized platform for observing, monitoring, and analyzing AI model traffic.

## Supported Providers

| Provider | How it is routed | Base URL (if supported) |
| --- | --- | --- |
| Anthropic Claude | Conversations on `api.anthropic.com` | `http://127.0.0.1:8080/anthropic` |
| OpenAI API & Chat Completions | Conversations on OpenAI/Codex domains | `http://127.0.0.1:8080/openai` |
| GitHub Copilot (all plans) | Conversations on Copilot domains | Proxy-only |
| ChatGPT (browser) | Conversations on `chatgpt.com` | Proxy-only |
| Cursor AI | Conversations on `api2.cursor.sh`, with HTTP/1.1 compatibility mode. | Proxy-only |

### Running the Proxy

```bash
aiproxy
```

Key CLI flags:
- `--listen-address` / `AIPROXY_LISTEN_ADDRESS` – override the port the proxy listens on (default: `127.0.0.1:8080`)
- `--config` / `AIPROXY_CONFIG_PATH` – select the config file (defaults to `./aiproxy.toml`)
- `--tui` / `--no-tui` – force-enable or disable the Ratatui frontend
- `--log <level>` – adjust log level (`info`, `debug`, `trace`, …)

Environment variable `RUST_LOG` can be used to fine tune logging further (workspace crates are auto-configured when `--log` is used).

Stop the proxy by pressing `q` inside the TUI. The process performs a graceful shutdown to flush in-flight telemetry updates before exiting.

## TLS Interception Workflow

Interception relies on a CA certificate trusted by your clients.

1. **Automatic CA generation** – when no CA is configured, the proxy creates `~/.aiproxy/aiproxy-ca-cert.pem` and `~/.aiproxy/aiproxy-ca.pem`.
2. **Custom CA** – to supply your own CA, generate an RSA pair and put the paths in the config:
   ```bash
   openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out ca.key -outform PEM
   openssl req -x509 -new -nodes -key ca.key -sha256 -days 1024 \
     -out ca.crt -subj "/C=US/ST=New York/L=New York/O=aiproxy/OU=Security/CN=aiproxy Root CA"
   ```
   Then configure:
   ```toml
   [tls_interception.certificate_authority]
   cert = "/path/to/ca.crt"
   key = "/path/to/ca.key"
   ```

### Installing the CA Certificate

Import the certificate once so applications trust the MITM edge.

- **NixOS**
  ```nix
  {
    security.pki.certificates = [''<contents of ca.crt>''];
  }
  ```
- **Generic Linux**
  ```bash
  sudo trust anchor --store ~/.aiproxy/aiproxy-ca-cert.pem
  ```
- **macOS**
  ```bash
  sudo security add-trusted-cert -d -p ssl -p basic \
    -k /Library/Keychains/System.keychain ~/.aiproxy/aiproxy-ca-cert.pem
  ```
- **Windows**
  For WSL applications, follow the instructions for Linux. For native Windows applications, run PowerShell as administrator and use the following command:
  ```powershell
  Import-Certificate -FilePath C:\path\to\ca.crt -CertStoreLocation "Cert:\CurrentUser\Root"
  ```

### Verifying the Certificate

```bash
curl -v --proxy 127.0.0.1:8080 \
     --cacert ~/.aiproxy/aiproxy-ca-cert.pem https://api.anthropic.com

# If the CA is trusted system-wide, omit --cacert
curl -v --proxy 127.0.0.1:8080 https://api.anthropic.com
```

Both commands should complete successfully (Anthropic responds with a 404).

## Using the Proxy with Clients

The proxy can be consumed in two ways:

1. **HTTP proxy mode** – point the client at `http://127.0.0.1:8080` via environment variables or app settings. Recommended for applications that expect CONNECT tunnelling (Copilot, browsers, most editors).
2. **Base URL mode** – some SDKs allow overriding the upstream API URL. Use the base URLs described earlier to bypass CONNECT negotiation.

Export the proxy once to cover most CLI tools:
```bash
export HTTP_PROXY=http://127.0.0.1:8080
export HTTPS_PROXY=http://127.0.0.1:8080

# Optional: single variable for curl and codex
export ALL_PROXY=http://127.0.0.1:8080

# Ensure Node-based clients (mostly Claude Code) see the MITM certificate
export NODE_EXTRA_CA_CERTS="$HOME/.aiproxy/aiproxy-ca-cert.pem"
```

### Claude Code

- **Proxy mode**
  ```bash
  export HTTP_PROXY=http://127.0.0.1:8080
  export NODE_EXTRA_CA_CERTS=$HOME/.aiproxy/aiproxy-ca-cert.pem
  claude
  ```
- **Base URL mode** (aiproxy acts as a router, not a proxy)
  ```bash
  export ANTHROPIC_BASE_URL=http://127.0.0.1:8080/anthropic
  claude
  ```

### OpenAI / Codex Clients

- **Proxy mode**
  ```bash
  export ALL_PROXY=http://127.0.0.1:8080
  codex
  ```
- **Base URL mode** – most OpenAI SDKs accept either `OPENAI_BASE_URL` or `OPENAI_API_BASE`:
  ```bash
  export OPENAI_BASE_URL=http://127.0.0.1:8080/openai
  export OPENAI_API_KEY=<your-key>
  python your_script.py
  ```
  Curl example (base URL mode):
  ```bash
  curl -s http://127.0.0.1:8080/openai/v1/models \
       -H "Authorization: Bearer $OPENAI_API_KEY"
  ```
  Some SDKs insist on HTTPS even with a base URL override. When that happens, prefer proxy mode and let aiproxy perform TLS interception transparently.

### VS Code Copilot
1. **Trust the CA** – import `~/.aiproxy/aiproxy-ca-cert.pem` into your OS.
2. **Configure VS Code** – set `http.proxy` (or `http.proxySupport`) to `http://127.0.0.1:8080`. Copilot respects the global proxy settings.
3. **Flatpak builds** – grant the sandbox access to the certificate:
   ```bash
   install -D -m 0644 /path/to/aiproxy.crt \
     /home/$USER/.var/app/com.visualstudio.code/data/certs/aiproxy-ca.pem

   sudo flatpak override \
     --filesystem=/home/$USER/.var/app/com.visualstudio.code/data/certs/aiproxy-ca.pem:ro \
     --env=ELECTRON_EXTRA_CA_CERTS=/home/$USER/.var/app/com.visualstudio.code/data/certs/aiproxy-ca.pem \
     --env=NODE_EXTRA_CA_CERTS=/home/$USER/.var/app/com.visualstudio.code/data/certs/aiproxy-ca.pem \
     com.visualstudio.code

   flatpak run --command=sh com.visualstudio.code -c \
     'mkdir -p $HOME/.pki/nssdb && \
      certutil -d sql:$HOME/.pki/nssdb \
        -A -t "C,," -n "aiproxy-ca" \
        -i /home/$USER/.var/app/com.visualstudio.code/data/certs/aiproxy-ca.pem'

   # ensure that vscode is completely shut down, and then restart it
   flatpak kill com.visualstudio.code
   ```

Copilot does not expose a stable base URL override – keep the proxy setting enabled instead.

### Cursor

1. **Trust the CA** (same steps as other clients).
2. Enable HTTP/1.1 compatibility so Cursor proxies over plain HTTP:
   - Cursor → **Settings (⇧⌘J) or (S-C-J on Linux/Windows) ** → **Network** → enable `HTTP Compatibility Mode`.
3. Point VS Code (if applicable) to the proxy under **Settings (⌘,) or (C-, on Linux/Windows) ** → **Application → Proxy → Proxy** = `http://localhost:8080`.

### Browsers (ChatGPT)

1. Trust the CA at the OS/browser level.
2. Configure the system or browser proxy to `http://127.0.0.1:8080`.
3. Visit your target site as normal – the traffic now traverses aiproxy and is eligible for telemetry.

### General Tips

- Some CLI tools cache DNS aggressively. When switching between proxy and direct modes, restart the application to ensure new settings take effect.
- If you see TLS errors, double-check that the CA is trusted and that `NODE_EXTRA_CA_CERTS` (for Node/Electron apps) or platform-specific trust stores have been updated.
