# Hermes Agent + Tailscale

This deployment runs Hermes Agent and Tailscale together. Hermes shares the Tailscale container's network namespace, so the Hermes dashboard is exposed through Tailscale Serve without publishing a Docker port on the host.

## Files and storage

- `hermes-data/` — persistent Hermes state, mounted at `/opt/data`.
- `tailscale-state` Docker volume — persistent Tailscale identity/state.
- The Hermes dashboard listens inside the shared network namespace on `127.0.0.1:9119`.
- Tailscale Serve publishes that dashboard on the tailnet over HTTPS.

Do not commit `.env` or an auth key.

## Setup

1. Copy the example environment file:

   ```bash
   cp .env.example .env
   ```

2. Put a short-lived or reusable Tailscale auth key in `.env`:

   ```text
   TS_AUTHKEY=tskey-auth-...
   ```

3. Make sure the host has the TUN device:

   ```bash
   test -e /dev/net/tun
   ```

4. Pull and start:

   ```bash
   docker compose up -d
   ```

5. Check both services:

   ```bash
   docker compose ps
   docker compose logs --tail 100 tailscale
   docker compose logs --tail 100 hermes
   ```

6. Find the Tailscale hostname/IP:

   ```bash
   docker compose exec tailscale tailscale status
   ```

Open the Tailscale HTTPS URL shown by `tailscale serve status`:

```bash
docker compose exec tailscale tailscale serve status
```

## Important security notes

- Tailscale authentication is kept in `.env`; the key is not embedded in the Compose file.
- Hermes state is kept in `hermes-data/`; back it up.
- This file does not mount `/var/run/docker.sock`, so Hermes cannot control the Docker host.
- Tailscale Serve is used instead of publishing `9119` to all host interfaces.
- The host Docker service must itself be enabled at boot for `restart: unless-stopped` to take effect after a reboot.

## Updating

```bash
docker compose pull
docker compose up -d
```
