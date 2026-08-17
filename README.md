# Hermes Agent + Tailscale

This deployment runs Hermes Agent and Tailscale together. Hermes shares the Tailscale container's network namespace, so the Hermes dashboard is exposed through Tailscale Serve without publishing a Docker port on the host. Dashboard basic authentication is enabled through the environment variables in `.env`.

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

   Set the dashboard credentials too:

   ```text
   HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
   HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=your-strong-password
   HERMES_DASHBOARD_BASIC_AUTH_SECRET=your-random-session-secret
   ```

   The session secret can be generated with:

   ```bash
   openssl rand -base64 32
   ```

   Optional settings are `HERMES_DATA_DIR`, `HERMES_UID`, and `HERMES_GID`. The defaults are shown in `.env.example`.

3. Make sure the host has the TUN device:

   ```bash
   test -e /dev/net/tun
   ```

4. Validate, replace any existing stack, and start:

   ```bash
   docker compose config --quiet
   docker compose down
   docker compose pull
   docker compose up -d
   ```

   Follow the combined logs if needed:

   ```bash
   docker compose logs --tail=100 -f
   ```

5. Configure Tailscale Serve after the containers start:

   ```bash
   docker compose exec tailscale \
     tailscale serve --bg http://127.0.0.1:9119
   ```

6. Verify both services and the dashboard endpoint:

   ```bash
   docker compose ps
   docker compose logs --tail 100 tailscale
   docker compose logs --tail 100 hermes
   ```

   ```bash
   docker compose exec tailscale tailscale status
   docker compose exec tailscale tailscale serve status
   docker compose exec tailscale \
     wget -S -O /dev/null http://127.0.0.1:9119/api/status
   ```

7. Find the Tailscale hostname/IP:

   ```bash
   docker compose exec tailscale tailscale status
   ```

Open the HTTPS URL shown by `tailscale serve status`, for example:

```bash
https://hermes.tailxxxx.ts.net/
```

## Important security notes

- Tailscale authentication is kept in `.env`; the key is not embedded in the Compose file.
- Dashboard credentials and the session secret are kept in `.env`; do not commit them.
- Hermes state is kept in `hermes-data/`; back it up.
- This file does not mount `/var/run/docker.sock`, so Hermes cannot control the Docker host.
- Tailscale Serve is used instead of publishing `9119` to all host interfaces.
- The host Docker service must itself be enabled at boot for `restart: unless-stopped` to take effect after a reboot.

## Updating

```bash
docker compose pull
docker compose up -d
```

After an update, re-check Tailscale Serve if the proxy is not available:

```bash
docker compose exec tailscale tailscale serve status
```
