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
| `EchoGrid__PublicAddress` | *(required)* | The address(es) your users type into the connect form, comma-separated |
| `EchoGrid__RequireBoundAuth` | `true` | Accept only signatures bound to that address |
| `EchoGrid__ServerName` | `Echo Grid` | Display name shown to clients |
| `EchoGrid__DatabasePath` | `/data/echogrid.db` | SQLite file |
| `EchoGrid__MediaPath` | `/data/media` | Uploaded file storage, content-addressed by SHA-256 |
| `EchoGrid__ChangeFeedRetentionDays` | `30` | How long resume-sync change rows are kept |

### `PublicAddress` and why the server won't start without it

Set it to exactly what a user types into the connect form: `chat.example.com`, or `192.168.1.50:5162` on a LAN.
No scheme, no path, and include the port only if it isn't 443.

If the same server answers to more than one name, comma-separate them:

```
EchoGrid__PublicAddress=chat.example.com, www.chat.example.com
```

A signature bound to any listed name is accepted; one bound to a name you haven't listed is not.
This is the same idea as SAN entries on a certificate - you are naming the identities your server legitimately answers to, which is why it adds nothing an attacker can use.
Note that `example.com` and `example.com:443` are *different* entries, because a client treats a colon as "connect in plaintext" and so dials a different server.

Clients sign a challenge to log in, and they bind that signature to the address they dialed.
The server checks the binding against this value.
Without it, a signature proves only "this key signed some bytes" and names no particular server, so a hostile server you visit could forward its challenge from *your* server, collect your answer, and replay it to log in as you.
Because everyone reuses one identity keypair across every server they join, that is a realistic path rather than an exotic one.

The value cannot be inferred from the incoming request: an attacker mounting exactly that relay controls the `Host` header they send.
So the server has to be told its own name, and it fails to start rather than run the handshake in a mode whose security it cannot deliver.

Clients older than 2.9.0 do not bind their signatures and will be rejected with a message telling the user to update.
If you need a migration window, set `EchoGrid__RequireBoundAuth=false` to also accept the old form.
That re-opens the relay for as long as it is set, so treat it as a window and not a setting.

## Updating

```bash
docker compose pull && docker compose up -d
```

Image tags follow the server version; `latest` tracks the newest release.
