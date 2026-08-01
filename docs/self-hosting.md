# Self-hosting Echo Grid

The stack is three containers: Echo Grid itself, a LiveKit sidecar for voice and screen share, and Caddy for TLS.
Persistence is a single SQLite file, so there is no external database to run.

## Requirements

- A host with a public IP and a domain pointed at it
- Docker with Compose
- These ports open: `443/tcp` (Caddy), `7881/tcp` (LiveKit TCP fallback), `50000-50100/udp` (LiveKit media)

Port `7880` stays loopback-only. Caddy terminates TLS and reverse-proxies to it.

## Setup

Fetch the three deploy files from `deploy/` in this repository, then set the environment variables the stack reads.
Compose picks these up from a `.env` file next to `docker-compose.yml`, or from your orchestrator's stack environment.

| Variable | Required | Purpose |
|---|---|---|
| `ECHOGRID_DOMAIN` | yes | The hostname Caddy terminates TLS on, e.g. `chat.example.com` |
| `LIVEKIT_NODE_IP` | yes | Your host's public IP. LiveKit cannot auto-detect it under `network_mode: host` |
| `LIVEKIT_PUBLIC_URL` | for voice | `wss://$ECHOGRID_DOMAIN/livekit` |
| `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` | for voice | Shared between Echo Grid and the sidecar. Generate with `livekit-server generate-keys` |
| `KLIPY_API_KEY` | no | Enables the GIF picker. Free key from https://klipy.com/developers |
| `FCM_SA_PATH` | no | Host path to a Firebase service-account JSON, for Android push |
| `APNS_P8_PATH` | no | Host path to an Apple `.p8` auth key, for iOS push |
| `APNS_TEAM_ID` / `APNS_KEY_ID` / `APNS_BUNDLE_ID` | with APNs | Apple developer identifiers |

Every optional group is config-as-gate: leave it unset and the feature turns itself off cleanly rather than erroring.
Voice and screen share need `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` and `LIVEKIT_PUBLIC_URL` all set; with any of them blank, clients hide the voice UI.

The two required variables are not config-as-gate, and one of them fails confusingly.
If `ECHOGRID_DOMAIN` is unset, Caddy refuses to start with `unrecognized global option: handle_path`, which never mentions the variable: an empty site address makes Caddy parse the site block as global options.
If you see that error, set `ECHOGRID_DOMAIN`.

## Start it

```bash
docker compose up -d
docker compose logs -f echogrid
```

Then point a client at your domain. **The first identity to connect becomes the server owner**, so connect once yourself before sharing the address.

## Server settings

Config is read from environment variables, where a double underscore separates config sections:

| Variable | Default | Purpose |
|---|---|---|
| `EchoGrid__ServerName` | `Echo Grid` | Display name shown to clients |
| `EchoGrid__DatabasePath` | `/data/echogrid.db` | SQLite file |
| `EchoGrid__MediaPath` | `/data/media` | Uploaded file storage, content-addressed by SHA-256 |
| `EchoGrid__ChangeFeedRetentionDays` | `30` | How long resume-sync change rows are kept |

## Updating

```bash
docker compose pull && docker compose up -d
```

Image tags follow the server version; `latest` tracks the newest release.
