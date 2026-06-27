# LDK Server Mint Production Compose

This deployment template runs `cdk-mintd` with the external `ldk-server`
backend while keeping the mint HTTP listener private to the host. Host Caddy is
the only public web entrypoint.

## Exposure Model

Public:

- `80/tcp` and `443/tcp`: host Caddy
- `9735/tcp`: Lightning P2P, only if the LDK Server node needs public peers
- `22/tcp`: SSH, preferably IP-restricted

Private:

- `127.0.0.1:3338`: `cdk-mintd`, reachable by host Caddy only
- `127.0.0.1:3536`: `ldk-server` API, reachable by the mint only
- `./data/cdk-mintd`: SQLite database, mint logs, and workdir state
- `./secrets`: mint seed, LDK Server API key, and pinned TLS certificate

The compose file intentionally has no `ports:` entry. It uses `network_mode:
host` so the mint can talk to a host-local `ldk-server` API and bind to host
loopback. Keep `CDK_MINTD_LISTEN_HOST=127.0.0.1`; changing it to `0.0.0.0`
would publish the mint directly on the host network.

## First-Time Setup

Run these commands from this directory:

```bash
cp .env.example .env
mkdir -p data/cdk-mintd secrets
chmod 700 secrets
```

Create or copy the mint seed:

```bash
install -m 0600 /path/to/cdk-mintd-seed secrets/cdk-mintd.seed
```

Copy the LDK Server TLS certificate:

```bash
install -m 0644 /path/to/ldk-server/tls.crt secrets/ldk-server-tls.crt
```

Store the LDK Server API key as the exact text expected by
`ldk-server-client`. If the source key is binary, hex-encode it:

```bash
python3 - <<'PY'
from pathlib import Path

source = Path("/path/to/ldk-server/api_key")
target = Path("secrets/ldk-server-api-key")
target.write_text(source.read_bytes().hex())
target.chmod(0o600)
PY
```

Set ownership for the non-root runtime user:

```bash
docker run --rm -v "$PWD/data:/data" debian:bookworm-slim \
  chown -R 10001:10001 /data/cdk-mintd
```

Edit `.env` if the LDK Server API is not on `127.0.0.1:3536`.

## Build And Start

```bash
docker compose build mint
docker compose up -d mint
docker compose ps
docker compose logs -f mint
```

Healthcheck:

```bash
curl -fsS http://127.0.0.1:3338/v1/info | jq .
```

## Caddy

Install the snippet in `caddy/mint.hedwig.sh.Caddyfile` into the host Caddyfile,
then force a reload after DNS is live:

```bash
caddy validate --config /etc/caddy/Caddyfile
caddy reload --force --config /etc/caddy/Caddyfile
```

Public check:

```bash
curl -fsS https://mint.hedwig.sh/v1/info | jq .
```

## Rollout Notes

- Pin `CDK_MINTD_IMAGE` to an immutable tag or digest once CI publishes images.
- Keep `LDK_SERVER_ADDRESS` private. If the LDK Server API must bind outside
  loopback, restrict it with firewall rules to the mint host only.
- Back up `data/cdk-mintd`, `secrets`, and the LDK Server data directory before
  upgrading.
- Restore-test backups before increasing public mint and melt limits.
