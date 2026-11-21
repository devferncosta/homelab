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
3. After a Windows reboot, sign in so Docker Desktop can start. The API, web container, tunnel connector, and optional MCP service each have an `always` restart policy. Check them with `docker compose ps` and confirm the HTTPS site responds again.

## Connect the AI Coach to a ChatGPT plan

Set `COACH_CHATGPT_PLAN=on` in the local `.env` and apply it with `docker compose --profile tunnel up -d --wait` for a tunneled deployment, or `docker compose up -d --wait` locally.

1. As the openGym admin, open **Settings → Account → Admin dashboard → AI Coach** and choose **ChatGPT plan (preview) → Continue with ChatGPT**.
2. Approve plan usage in the browser. The browser redirects to a local callback and may show a connection error. Copy the full callback address into the openGym sign-in form and select **Finish sign-in**.
3. Select **List models**, choose a model, and select **Test the Coach**.

Eligible usage counts against the connected ChatGPT plan. The authorization binds to the first profile that uses it. Check OpenAI's preview eligibility before offering plan usage to other users. [openGym Coach guide](https://github.com/DuarteSantos8/openGym/blob/v1.4.5/docs/AI_COACH.md#your-chatgpt-plan-preview) · [OpenAI Sign in with ChatGPT](https://developers.openai.com/siwc/token-sharing-open-source)

## Use openGym from ChatGPT

Set `MCP=on` and `COMPOSE_PROFILES=tunnel,mcp` in the live `.env`, then run `docker compose up -d --wait`. The existing HTTPS hostname serves the connector at `/mcp`. Check `/mcp/health` and open **Settings → Use your subscription** in openGym.

In ChatGPT, open **Plugins → Add custom MCP server**, enter the full HTTPS `/mcp` address, and choose **OAuth**. Sign in to openGym and authorize the requested access. Each authorization grants access to one profile. Routine changes and workout logging require MCP write support in ChatGPT. [openGym OAuth connector](https://github.com/DuarteSantos8/openGym/blob/v1.4.5/docs/MCP_REMOTE.md#claude-chatgpt-and-gemini-sign-in-with-opengym-oauth) · [ChatGPT custom MCP availability](https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt)
