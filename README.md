# Echo Grid

Self-hosted chat and voice. Desktop, Android and iOS clients, one server you run yourself.

- Website: https://echogrid.app
- Server image: `ghcr.io/cgtheodorou/echogrid-server`

## Download

Desktop builds are attached to every [release](https://github.com/cgtheodorou/echogrid-releases/releases):
Windows installer, macOS Apple Silicon, Linux AppImage / DEB / RPM.
The desktop app updates itself, so you only need to download once.

Android ships on [Google Play](https://play.google.com/apps/testing/app.echogrid), and the APK is attached to each release for sideloading.
iOS is in beta via [TestFlight](https://testflight.apple.com/join/BCX4pBCE).

## Run your own server

See [docs/self-hosting.md](docs/self-hosting.md). The short version:

```bash
curl -O https://raw.githubusercontent.com/cgtheodorou/echogrid-releases/main/deploy/docker-compose.yml
curl -O https://raw.githubusercontent.com/cgtheodorou/echogrid-releases/main/deploy/Caddyfile
curl -O https://raw.githubusercontent.com/cgtheodorou/echogrid-releases/main/deploy/livekit.yaml
docker compose up -d
```

## Protocol

`protocol/schema/` documents the JSON wire format spoken over the single `/ws` WebSocket.
It is reference documentation, not generated code.

## About this repository

This repo carries downloads, deployment files and documentation.
Every file in it is generated and pushed by CI from a private source repository, so pull requests here cannot be merged and will be closed with a pointer back to Issues.

Bug reports and feature requests are very welcome in [Issues](https://github.com/cgtheodorou/echogrid-releases/issues).
