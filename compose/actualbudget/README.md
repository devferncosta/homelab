# Actual Budget

## Local setup

Run these PowerShell commands from `compose/actualbudget`. Copy the example only on a fresh setup. It pins the official image to `26.11.0`.

```powershell
Copy-Item .env.example .env
docker compose config --quiet
docker compose up -d --wait
```

Open [localhost:5006](http://localhost:5006) to set the shared server password and save it in your password manager. Use the same password for each household device. Finish this step before publishing the service. [Actual Docker guide](https://actualbudget.org/docs/install/docker/)

Actual stores its server database and synced budget files under `/data` in the `homelab-actualbudget_actualbudget-data` Docker volume. Use `docker compose down` without `--volumes` to preserve it. Use Actual's budget export for independent recovery and keep exports outside the repository. [Backup guide](https://actualbudget.org/docs/backup-restore/backup/)

## Publish over HTTPS

The openGym Compose project creates `homelab-tunnel-origins` for the shared cloudflared connector. On Windows, add `COMPOSE_FILE=compose.yaml;compose.tunnel.yaml` to the local Actual `.env`. From `compose/actualbudget`, run `docker compose config --quiet` and `docker compose up -d --wait`. The Windows port stays bound to loopback.

Back up the live tunnel YAML. Add the Actual route above the final `http_status:404` rule, using your own hostname.

```yaml
  - hostname: budget.example.com
    service: http://actualbudget:5006
```

From `compose/opengym`, validate the tunnel config and confirm it matches the new route. Substitute your own hostname in the second command.

```powershell
docker compose --profile tunnel run --rm --no-deps cloudflared tunnel --config /etc/cloudflared/config.yml ingress validate
docker compose --profile tunnel run --rm --no-deps cloudflared tunnel --config /etc/cloudflared/config.yml ingress rule https://budget.example.com
docker compose --profile tunnel up -d --no-deps --force-recreate cloudflared
```

Add the hostname's proxied CNAME to the existing tunnel in Cloudflare DNS, or use `cloudflared tunnel route dns <tunnel-name-or-id> <hostname>` with the local Cloudflare account certificate. Open the new HTTPS address, sign in, and check that the other tunnel services still respond. Use the HTTPS address on phones and other devices. [Cloudflare tunnel commands](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/tunnel-useful-commands/) · [Actual browser requirements](https://actualbudget.org/docs/troubleshooting/shared-array-buffer/)

To roll back publication, remove the new DNS record and Actual ingress entry, recreate cloudflared, then remove the `COMPOSE_FILE` line from the local Actual `.env` and run `docker compose up -d --wait` from `compose/actualbudget`. The budget remains accessible on localhost.

## Restore and cut over

Sync the source devices and pause edits during cutover. Export the source budget from **Settings → Export data**. Store a restorable copy securely outside the repo. In the new Actual instance, select **Switch file → Import file → Actual** and choose the ZIP. Compare balances, transactions, categories, and budget amounts. Use a second device to open the imported budget from the server and verify an edit syncs. Test over cellular if you use remote access. If you use bank sync, check provider credentials and account links before running the first import. [Restore guide](https://actualbudget.org/docs/backup-restore/restore/) · [Sync guide](https://actualbudget.org/docs/getting-started/sync/)

After verifying the new copy, send all new edits to the homelab and keep the source available for rollback. If you need to return to the source after making new edits, pause edits, export the current homelab budget, and restore and verify it on the source before resuming there.

## Manual upgrades

Export the budget and read the [release notes](https://actualbudget.org/docs/releases/) before changing `ACTUAL_VERSION` in `.env`. Apply the pin with `docker compose pull` and `docker compose up -d --wait`. Verify sign-in and sync afterward. If an upgrade has migrated stored data, use the previous image with a pre-upgrade backup for rollback.
