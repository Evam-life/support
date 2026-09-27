# Evam CAD Bridge

The Evam CAD Bridge is a small process that runs on your infrastructure and connects your CAD system to Evam Central Services. It maintains a permanent, authenticated WebSocket connection to Evam so your CAD only has to talk to something on its own local network — no inbound firewall rules, no public exposure of CAD systems.

Published artifacts:

- **Docker image** — [`ghcr.io/evam-life/cad-bridge`](https://github.com/evam-life/cad-bridge/pkgs/container/cad-bridge)
- **JAR** — attached to each [GitHub Release](https://github.com/evam-life/cad-bridge/releases) as `evam-cad-bridge-<version>-all.jar`

Latest version: `1.0.10`.

## What it does

```
                        ┌─────────────────────────┐
                        │   Evam Central Services  │
                        └────┬───────────────┬─────┘
                             │               │
               WebSocket (outbound, TLS)  ── same host, same auth ──
                             │               │
                     ┌───────┴───┐   ┌───────┴────────┐
                     │ CAD Bridge│   │  CAD Bridge     │
                     │  (TCP)    │   │  (HTTP proxy)   │
                     └───────┬───┘   └───────┬────────┘
                             │               │
                             ▼               ▼
                        Your CAD        Your internal
                        system          HTTP services
```

The bridge always initiates the connection outward to Evam — so you only need to make sure your firewall allows outbound connections towards Central Services, no inbound rule needed.

---

## Installation

### Prerequisites

- **Docker** (recommended) — any recent version.
- **JVM 21+** if you prefer running the JAR directly.
- A **configuration file** — copy the [sample below](#sample-configuration) into `config/application.yaml` and edit it. The values Evam provides during setup go into the `websocket` and `oauth2` sections.

### Option 1 — Docker (recommended)

Pull a specific release:

```bash
docker pull ghcr.io/evam-life/cad-bridge:<version>
```

Or the rolling `latest` tag:

```bash
docker pull ghcr.io/evam-life/cad-bridge:latest
```

Run it, mounting your config directory and passing sensitive values through environment variables:

```bash
docker run --rm \
  --name evam-cad-bridge \
  -v $(pwd)/config:/app/config:ro \
  -e BRIDGE_PASSWORD=secret \
  -p 10987:10987 \
  ghcr.io/evam-life/cad-bridge:<version>
```

Notes:
- The container reads its config from `/app/config/application.yaml` — mount your local `config/` directory there.
- Port `10987` is the default TCP port from the sample config; adjust `-p` if you change `tcp.port`, and drop it entirely if you use `tcp.mode: client`.

### Option 2 — JAR

Download the release asset from the [Releases page](https://github.com/evam-life/cad-bridge/releases) — or via `curl`:

```bash
VERSION=<version>
curl -L -o evam-cad-bridge.jar \
  https://github.com/evam-life/cad-bridge/releases/download/${VERSION}/evam-cad-bridge-${VERSION}-all.jar
```

Run it:

```bash
java -jar evam-cad-bridge.jar --config config/application.yaml
```

Pass secrets through the environment:

```bash
BRIDGE_PASSWORD=secret \
  java -jar evam-cad-bridge.jar --config config/application.yaml
```

### Running as a service (Linux)

For long-running deployments, wrap the JAR in a systemd unit:

```ini
# /etc/systemd/system/evam-cad-bridge.service
[Unit]
Description=Evam CAD Bridge
After=network-online.target

[Service]
User=cadbridge
WorkingDirectory=/opt/evam-cad-bridge
EnvironmentFile=/etc/evam-cad-bridge/env
ExecStart=/usr/bin/java -jar /opt/evam-cad-bridge/evam-cad-bridge.jar \
          --config /etc/evam-cad-bridge/application.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Put your secrets in `/etc/evam-cad-bridge/env` (mode `0600`), then `systemctl enable --now evam-cad-bridge`.

---

## Configuration

All configuration lives in a single YAML file. Pass its path with `--config <file>` — or, in Docker, mount it at `/app/config/application.yaml`.

### Sample configuration

Copy this into `config/application.yaml`, then edit the values marked *(edit)* for your deployment. Anything wrapped in `${VAR:default}` can be overridden by an environment variable at runtime.

```yaml
# evam-cad-bridge configuration
#
# Any string value can reference environment variables:
#   "${VAR}"          -> required, startup fails if unset
#   "${VAR:default}"  -> optional, falls back to `default`

# --- Evam Central Services connection -----------------------------------------
# The URLs and credentials below are provided by Evam during setup.
websocket:
  url: "${BRIDGE_WS_URL}"                # (Evam-provided)

oauth2:
  token-url: "${BRIDGE_TOKEN_URL}"       # (Evam-provided)
  client-id: "${BRIDGE_CLIENT_ID:web}"   # (Evam-provided)
  username: "${BRIDGE_USERNAME}"         # (Evam-provided)
  password: "${BRIDGE_PASSWORD}"         # (Evam-provided; keep as env var)

# --- Local CAD-facing side ----------------------------------------------------
# This is the part you configure to match your CAD system's networking.
tcp:
  mode: "server"                         # "server" or "client" — see below
  host: "0.0.0.0"                        # bind address (server) OR remote host (client)
  port: 10987
  tls:
    enabled: false
    # server-mode TLS:
    keystore-path: ""
    keystore-password: ""
    key-password: ""
    # client-mode TLS:
    truststore-path: ""
    truststore-password: ""

# --- Optional HTTP proxy ------------------------------------------------------
# Only enable if Evam has explicitly asked you to.
http-proxy:
  enabled: false
  websocket-url: "${BRIDGE_HTTP_PROXY_WS_URL:}"   # (Evam-provided if enabled)
```

### `websocket` and `oauth2` — Evam Central Services

Both sections describe how the bridge connects to Evam Central Services. The URLs, client ID, username, and password are **all provided by Evam during setup** — you don't invent them. Once configured, the bridge:

- keeps a permanent authenticated WebSocket open to Evam,
- refreshes its OAuth2 token automatically (using `refresh_token` when available, falling back to re-authenticating),
- reconnects on its own if the link drops.

You normally don't need to touch these fields beyond pasting in what Evam sent you.

The `websocket.reconnect` block has sensible defaults (`enabled: true`, `delay-ms: 5000`, `max-retries: -1` for unlimited) and can be omitted.

### `tcp` — your CAD side

This is the part **you** configure to match your CAD system's networking. The bridge relays raw bytes between this TCP endpoint and Evam Central Services in both directions.

#### `mode: server` vs `mode: client`

Which one to use depends on how your CAD system is set up to talk to a gateway:

| Aspect | `mode: server` | `mode: client` |
|---|---|---|
| Who initiates the TCP connection? | Your **CAD** connects **to** the bridge | The **bridge** connects **to** your CAD |
| `host` field | Address to **bind** to (`0.0.0.0` = all interfaces, `127.0.0.1` = localhost only) | Address of your **CAD system** |
| `port` field | Port the bridge listens on | Port your CAD is listening on |
| Docker `-p` flag | **Required** — expose the port to the host | Not needed |
| TLS material | `tls.keystore-path` (bridge presents a cert) | `tls.truststore-path` (bridge verifies the CAD's cert) |
| When to pick this | Most common. Your CAD is configured to "connect to a GD92 gateway." | Your CAD is running its own TCP listener and expects an inbound connection. |

Only one TCP connection is active at a time in either mode. If the connection drops, the bridge reopens it (`client` mode) or waits for the CAD to reconnect (`server` mode). Traffic in both directions is logged at INFO level with a hex dump so you can inspect what's flowing.

#### `tcp.tls`

Set `tls.enabled: true` if your CAD requires TLS on the TCP link.

- In `mode: server`, the bridge presents a certificate — configure `keystore-path`, `keystore-password`, and (if the key is separately protected) `key-password`. The keystore must be a JKS file.
- In `mode: client`, the bridge verifies the CAD's certificate against a trust store — configure `truststore-path` and `truststore-password`. JKS format.
- If your CAD trusts a public CA, leave the truststore fields empty and the bridge uses the JVM default trust store.

### `http-proxy` — optional

Only relevant if Evam has asked you to enable it. When enabled, the bridge opens a second connection to Evam so that specific backend services can make HTTP requests through the bridge into your local network (for example, to a downstream service that isn't publicly reachable).

| Field | Description | Default |
|---|---|---|
| `enabled` | Enable the HTTP proxy | `false` |
| `websocket-url` | WebSocket URL for the proxy connection (Evam-provided) | *required if enabled* |
| `timeout-ms` | Per-request HTTP timeout in milliseconds | `30000` |
| `max-concurrent-requests` | Maximum parallel in-flight HTTP requests | `16` |

### Environment variable substitution

Any string value in the YAML can reference environment variables:

```yaml
password: "${BRIDGE_PASSWORD}"           # required — startup fails if the var is unset
password: "${BRIDGE_PASSWORD:changeme}"  # with a default fallback
```

Comment lines are not processed, so `# ${SOME_VAR}` is safe. This is the recommended way to keep secrets out of the YAML file.

---

## Is it running?

After starting the program with a proper configuration, this will be printed to STDOUT:

```
13:53:05.157 [main] INFO  com.evam.bridge.Main - Loading configuration from config/application.yaml
13:53:05.211 [main] INFO  com.evam.bridge.Main - Configuration loaded — WS: wss://(...)/v2/ws/integration/cad/gd92, TCP: SERVER 0.0.0.0:10987
13:53:05.289 [main] INFO  com.evam.bridge.bridge.Bridge - Bridge starting
13:53:05.293 [main] INFO  c.e.bridge.auth.OAuth2TokenProvider - Requesting OAuth2 token from https://(...)/realms/cs/protocol/openid-connect/token (grant=password)
13:53:05.320 [main] INFO  com.evam.bridge.tcp.TcpServer - TCP server listening on 0.0.0.0:10987
13:53:05.320 [main] INFO  com.evam.bridge.tcp.TcpServer - Waiting for TCP client ...
13:53:05.580 [main] INFO  c.e.bridge.auth.OAuth2TokenProvider - OAuth2 token acquired, expires in 1800s (will refresh at 80%)
13:53:05.580 [main] INFO  c.e.bridge.websocket.WebSocketBridge - Connecting to WebSocket wss://(...)/v2/ws/integration/cad/gd92 (attempt 1)
13:53:05.613 [main] INFO  c.e.bridge.websocket.WebSocketBridge - WebSocket connected to wss://(...)/v2/ws/integration/cad/gd92
```

You should **not** see any error, notably no connection issue towards Central Services, such as:

```
c.e.bridge.websocket.WebSocketBridge - Reconnecting in 5000 ms ...
```

---

## Hardware requirements

The CAD bridge requires a minimum of 256MB of available memory to operate without HTTP proxy (`http-proxy.enabled: false`), 512MB if the HTTP proxy is enabled.

One vCPU is enough for a deployment without HTTP proxy, consider using 2 vCPUS if the HTTP proxy is enabled.

Ubuntu 24.04 LTS is the recommended Operating System for deployment.

---

## Updates

### Docker

Pull the new tag and restart the container:

```bash
docker pull ghcr.io/evam-life/cad-bridge:<new-version>

docker stop evam-cad-bridge && docker rm evam-cad-bridge
docker run --rm --name evam-cad-bridge \
  -v $(pwd)/config:/app/config:ro \
  -e BRIDGE_PASSWORD=secret \
  -p 10987:10987 \
  ghcr.io/evam-life/cad-bridge:<new-version>
```

If you're using Docker Compose or an orchestrator, update the image tag in your manifest and re-apply. Pin to a specific version in production — `latest` is convenient but reduces reproducibility.

### JAR

Download the new release asset, stop the running process, replace the JAR, and start again:

```bash
curl -L -o evam-cad-bridge.jar.new \
  https://github.com/evam-life/cad-bridge/releases/download/<new-version>/evam-cad-bridge-<new-version>-all.jar

systemctl stop evam-cad-bridge      # if you're using the systemd unit above
mv evam-cad-bridge.jar.new /opt/evam-cad-bridge/evam-cad-bridge.jar
systemctl start evam-cad-bridge
```

Config is loaded fresh on each startup — no migration is needed for the fields documented above. Breaking changes will be called out in the release notes on GitHub.
