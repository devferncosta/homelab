# openGym

## Local test

Run these PowerShell commands from `compose/opengym` on a fresh checkout. The example pins both upstream images to the same release.

```powershell
Copy-Item .env.example .env
docker compose up -d --wait
```

Open [localhost:8080](http://localhost:8080) on the host. Create a test profile first. The first profile becomes the instance admin. Check the API at [localhost:8080/api/health](http://localhost:8080/api/health).

The API stores profiles and workout data in `data/`. The local `.env` and `data/` directory are ignored by Git. Passkeys created here are tied to `localhost`. [Upstream self-hosting guide](https://github.com/DuarteSantos8/openGym/blob/v1.4.5/docs/SELF_HOSTING.md)

## Publish over HTTPS

For a fresh deployment, set `OPENGYM_DATA_DIR` in the local `.env` to a separate data directory outside the checkout. Create the first admin while the service is reachable only from the host. To bring that account to a different hostname, set `OPENGYM_PASSWORD_LOGIN=1` during creation and register it with a password.

After the admin exists, set `OPENGYM_INVITE_ONLY=1`, `OPENGYM_RP_ID` to the public hostname, and `OPENGYM_ORIGIN` to its full HTTPS origin. Apply these settings with `docker compose up -d --wait` before publishing the tunnel route. Sign in at the HTTPS address and register a passkey there. Verify it works before turning password sign-in off.

To run the Cloudflare connector with Compose, set `OPENGYM_TUNNEL_CONFIG` and `OPENGYM_TUNNEL_CREDENTIALS` in the local `.env` to the locations of the tunnel YAML and its credentials JSON. Keep both files outside the checkout. In the tunnel YAML, point `credentials-file` at `/etc/cloudflared/credentials.json`. Configure the app hostname to route to `http://web:80` and end the ingress list with `http_status:404`. Run `docker compose --profile tunnel up -d --wait` and verify the tunnel is connected before adding the DNS route. [Cloudflare tunnel guide](https://developers.cloudflare.com/tunnel/features/locally-managed-tunnels/create-local-tunnel/)

## Verify phone sync and restart

1. Sign in at the HTTPS address and select **Settings → Account → Pair the mobile app**. Confirm with a passkey or with your password while password sign-in is enabled.
2. In the native app, select **Connect to my server** and enter the HTTPS address and the one-time pairing code. With Wi-Fi off and cellular data on, log a sample workout and confirm it appears in the browser. This tests access and sync from outside the home network. [Upstream mobile guide](https://github.com/DuarteSantos8/openGym/blob/v1.4.5/docs/MOBILE.md#connecting-the-app-to-your-own-server)
3. After a Windows reboot, sign in so Docker Desktop can start. The API, web container, and tunnel connector each have an `always` restart policy. Check them with `docker compose --profile tunnel ps` and confirm the HTTPS site responds again.

## Connect Codex to two profiles

1. Sign in to each openGym profile and open **Settings → Integrations → API tokens → New token**. Create one token per profile. For routine updates, grant **Plans & routines → Change directly**. Also grant **Workouts → Change directly** if Codex should log workouts. Choose **See** for categories it only needs to read. [openGym token scopes](https://github.com/DuarteSantos8/openGym/blob/v1.4.5/docs/API_TOKENS.md)
2. Add one entry per profile to the local Codex `config.toml`. Copy the example for the second profile with a distinct server name and its own token. Keep the tokens in the local config.

```toml
[mcp_servers.opengym-first]
command = "docker"
args = ["run", "--rm", "--no-healthcheck", "-i", "-e", "OPENGYM_URL", "-e", "OPENGYM_TOKEN", "ghcr.io/duartesantos8/opengym-mcp:1.4.5", "node", "src/index.js"]
env = { OPENGYM_URL = "https://gym.example.com", OPENGYM_TOKEN = "<profile token>" }
```

3. Restart Codex and use `/mcp` to check that both servers loaded. Ask each connection to list routines first. Name the target profile explicitly when requesting a change. [Codex MCP configuration](https://learn.chatgpt.com/docs/extend/mcp?surface=cli)
