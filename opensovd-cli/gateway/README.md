<!--
SPDX-FileCopyrightText: Copyright (c) 2026 Contributors to the Eclipse Foundation
SPDX-License-Identifier: Apache-2.0
-->

# OpenSOVD Gateway

> HTTP gateway server for OpenSOVD vehicle diagnostics.

Exposes [OpenSOVD](https://github.com/eclipse-opensovd/opensovd-core) diagnostic services over HTTP, implementing the [SOVD](https://www.iso.org/standard/86587.html) REST API.

## Usage

```bash
# Listen on localhost:7690 (default)
opensovd-gateway

# Listen on all interfaces
opensovd-gateway --url http://0.0.0.0:8080/sovd

# Listen on a Unix socket
opensovd-gateway --unix-socket /tmp/opensovd.sock

# Listen on an abstract Unix socket (Linux)
opensovd-gateway --unix-socket @opensovd

# Enable mock topology for testing
opensovd-gateway --mock
```

Mock data comes from the shared `opensovd-mocks` crate used across examples and tests.

## Options

| Option          | Description                                          |
|-----------------|------------------------------------------------------|
| `--url`         | Server URL with base URI path (default: `http://localhost:7690/sovd`) |
| `--unix-socket` | Unix socket path (`@` prefix for abstract sockets)   |
| `--mock`        | Enable mock entities for testing                     |
| `--serve-dir`   | Serve static files (`PATH:DIRECTORY`)                |

### CORS Options

| Option               | Description                        |
|----------------------|------------------------------------|
| `--cors-origin`      | Allowed origins (`*` for any)      |
| `--cors-method`      | Allowed methods (`*` for any)      |
| `--cors-header`      | Allowed headers (`*` for any)      |
| `--cors-credentials` | Allow credentials                  |
| `--cors-max-age`     | Preflight cache duration (seconds) |

## Deploying on the Jetson AGX Orin

Runs the gateway as a native systemd service (no container), fed by Robodog's Zenoh bus,
reachable from other devices on the network. The Zenoh discovery/data provider is already
built into the gateway (`opensovd-providers/src/zenoh.rs`) and connects unconditionally on
startup via `--zenoh-endpoint` / `ZENOH_ENDPOINT`.

Assumes the Robodog stack (`../robodog-digipro/docker`) is already running/autostarting on
the same Jetson, with `zenoh-router` publishing `7447/tcp` to the host.

**Install (run once on the Jetson, from the `opensovd-server` repo root):**

```bash
# 1. Install a Rust toolchain (aarch64 — this is a native build, no cross-compilation needed)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

# 2. Install build prerequisites (cmake/clang build aws-lc-rs, used by the default `tls` feature;
#    leave them out and add --no-default-features --features jsonschema,json-preserve-order,mock,http2
#    to the build command below if TLS isn't needed on a trusted LAN)
sudo apt update
sudo apt install -y build-essential pkg-config cmake clang

# 3. Build the gateway
cargo build --release -p opensovd-gateway

# 4. Install the binary
sudo cp target/release/opensovd-gateway /usr/local/bin/opensovd-gateway

# 5. Install the unit and set the account that should run it
sudo cp packaging/systemd/opensovd-gateway.service /etc/systemd/system/
sudo sed -i "s/^User=.*/User=$(whoami)/" /etc/systemd/system/opensovd-gateway.service

# 6. If zenoh-router isn't reachable at tcp/127.0.0.1:7447, adjust --zenoh-endpoint in the unit:
# sudo sed -i "s|--zenoh-endpoint tcp/127.0.0.1:7447|--zenoh-endpoint tcp/<host>:<port>|" \
#   /etc/systemd/system/opensovd-gateway.service

# 7. Enable and start it now
sudo systemctl daemon-reload
sudo systemctl enable --now opensovd-gateway

# 8. Check it's up (Restart=on-failure means it'll keep retrying if Zenoh isn't reachable yet)
sudo systemctl status opensovd-gateway
```

**Verify (from another device on the same network):**

```bash
curl http://<jetson-ip>:7690/sovd/v1/components
```

**Managing the service:**

```bash
sudo systemctl restart opensovd-gateway   # restart (e.g. after a rebuild)
sudo systemctl stop opensovd-gateway      # stop
sudo systemctl disable opensovd-gateway   # stop autostarting on boot
journalctl -u opensovd-gateway -f         # tail logs
```

> **Firewall:** if `ufw`/`nftables` is active on the Jetson, also allow inbound traffic:
> `sudo ufw allow 7690/tcp`.

> **Reboot test:** after installing, reboot the Jetson once and confirm the service comes
> back up unattended (`systemctl status opensovd-gateway`).

> **Security note:** without `--auth-jwt-secret` the gateway runs with no
> authentication (`NoAuth`/`AllowAll`) — anyone on the network can read the exposed
> diagnostic data. Fine for a trusted lab LAN; enable JWT auth (see `--auth-jwt-secret`
> below) before exposing this more broadly.

## Contributing

See [CONTRIBUTING.md](../../CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the Apache License 2.0.
