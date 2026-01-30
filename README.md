# Project-Compose-Oracle

Run Ollama behind Tailscale with Docker Compose. This stack starts a Tailscale sidecar (`oracle-ts`) and an Ollama server (`oracle-app`) that shares the Tailscale network stack for secure tailnet access, while optionally exposing a local LAN port.

## Overview

- **Services**
  - `oracle-ts`: Tailscale container providing a Tailnet IP and secure connectivity.
  - `oracle-app`: Ollama server sharing the Tailscale network via `network_mode: service:oracle-ts`.
- **Access**
  - Preferred: Connect over Tailscale using the device name (`TS_HOSTNAME`) or Tailnet IP on port `11434`.
  - Optional: A host port (`PUBLIC_PORT`, default `11434`) is mapped on `oracle-ts` for local LAN access.
- **Persistence**
  - `oracle-ts-state`: Persists Tailscale state.
  - `oracle-data`: Persists Ollama models and data.

## Prerequisites

- Linux host with Docker and the Compose plugin.
- Kernel TUN device available (`/dev/net/tun`) and Docker permission to use it.
- A valid Tailscale auth key (`TS_AUTHKEY`). Ephemeral or pre-auth keys are recommended.
- Ability to advertise tag `tag:container` in your Tailnet (if using tags).

## Quick Start

1. Create a `.env` file next to `docker-compose.yml`:

```env
# Stack naming
STACK_NAME=oracle

# Tailscale
TS_HOSTNAME=oracle-ollama
TS_AUTHKEY=tskey-xxxxxxxxxxxxxxxx

# Optional configuration
PUBLIC_PORT=11434   # host port for LAN access (optional)
TIMEZONE=UTC        # container timezone (optional)
```

2. Start the stack:

```bash
docker compose up -d
```

3. Verify Tailscale is connected:

```bash
docker compose logs -f oracle-ts
```

4. Pull a model and test Ollama:

```bash
# List installed models
docker compose exec oracle-app ollama list

# Pull a model
docker compose exec oracle-app ollama pull llama3.2

# Run a quick prompt
docker compose exec oracle-app ollama run llama3.2 "Hello"
```

5. Access the API:

- Over Tailscale: `http://TS_HOSTNAME:11434/`
- Over LAN (if `PUBLIC_PORT` is set): `http://localhost:11434/`

Example request:

```bash
curl http://localhost:11434/api/tags
```

## Configuration

This project is configured via environment variables (set in `.env` or your shell):

- `STACK_NAME` (required): Base name for containers and volumes.
- `TS_HOSTNAME` (required): Tailscale device name for the container.
- `TS_AUTHKEY` (required): Tailscale auth key. Prefer ephemeral or pre-auth keys.
- `PUBLIC_PORT` (optional, default `11434`): Host port mapped to `11434` inside the stack to enable LAN access.
- `TIMEZONE` (optional, default `UTC`): Container timezone.

Advanced Tailscale flags (pre-configured in compose):

- `TS_STATE_DIR=/var/lib/tailscale` with a named volume (`oracle-ts-state`).
- `TS_USERSPACE=false` to use the kernel TUN device (required on Linux for full performance).
- `TS_EXTRA_ARGS=--advertise-tags=tag:container --reset` to advertise a tag and reset settings on boot.
  - Note: `--reset` will reapply configuration on start; consider removing it after initial stabilization if undesired.

## Networking Model

- `oracle-app` uses `network_mode: service:oracle-ts`, sharing the network stack with `oracle-ts`.
- Ollama listens on `0.0.0.0:11434` within the shared stack. Access is available via:
  - The Tailscale Tailnet address assigned to `oracle-ts`.
  - The optional host port mapping on `oracle-ts` for LAN (`PUBLIC_PORT`).

## Volumes

- `oracle-ts-state` → `/var/lib/tailscale` (Tailscale state)
- `oracle-data` → `/root/.ollama` (Ollama models and data)

Volumes persist across restarts and `docker compose down`. Use `-v` to remove them.

## Common Operations

- Start / stop:

```bash
docker compose up -d
docker compose down
```

- Restart a service:

```bash
docker compose restart oracle-ts
docker compose restart oracle-app
```

- View logs:

```bash
docker compose logs -f oracle-ts
docker compose logs -f oracle-app
```

- Update images and recreate:

```bash
docker compose pull
docker compose up -d
```

- Remove with volumes (CAUTION: deletes state and models):

```bash
docker compose down -v
```

## Troubleshooting

- TUN device errors (`/dev/net/tun`): Ensure the host has the TUN module loaded and Docker has access. The compose file mounts `/dev/net/tun` and grants `CAP_NET_ADMIN`.
- Tailscale not connecting: Verify `TS_AUTHKEY` validity (ephemeral keys expire), Tailnet policy allows `tag:container`, and check logs in `oracle-ts`.
- Port conflicts: Change `PUBLIC_PORT` in `.env` if `11434` is in use.
- `--reset` side effect: If Tailscale settings reset each start, remove `--reset` from `TS_EXTRA_ARGS` in the compose file once the device is stable.

## FAQ

- Why share the network stack? It ensures Ollama is directly reachable via the Tailnet IP managed by Tailscale, simplifying secure access.
- Can I disable LAN access? Yes. Omit or remove the `ports:` mapping (or `PUBLIC_PORT`) if you want Tailnet-only access.
- Can I run without Tailscale? Technically yes by removing `oracle-ts` and using a standard bridge network with ports, but this setup is optimized for secure Tailnet access.

## License

See `LICENSE` for details.
