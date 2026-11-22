# OpenTodo

## Local setup

Run these PowerShell commands from `compose/opentodo` on a fresh checkout. The example pins the official image to version `1.6.0`. Reuse your existing `.env` if you already have one.

```powershell
Copy-Item .env.example .env
docker compose config --quiet
docker compose up -d --wait
```

Open [localhost:8081](http://localhost:8081) on the host and create the owner account with a password. Open signup is disabled once the owner exists. The port is bound to the host's loopback interface. [Upstream Docker guide](https://open-todo.app/docs/self-hosting/docker/)

OpenTodo stores its database and attachments under `/data` in the `homelab-opentodo_opentodo-data` Docker volume. Compose reuses this volume on restart or recreation. Keep the local `.env` and any backup archives out of Git, and use `docker compose down` without `--volumes` when stopping the project.

Run `docker compose restart opentodo` followed by `docker compose up -d --wait`. Check [local health](http://localhost:8081/api/v1/health), then sign in again and check any sample task you created.

## Test separate accounts and a shared task

Run this check on the host or after publishing HTTPS if the second person will sign up from a phone.

1. From the owner account, create a single-use invitation under **Settings → People → Users and invitations**. Have the second person open the link on their phone after publication, or in a private browser window on the host for a local test. [Invitation guide](https://open-todo.app/docs/invite-people/)
2. Create a sample project, open **Share**, and add the second account as an editor. Create a sample task from one account, then complete it from the other and confirm the first account sees the update. [Sharing guide](https://open-todo.app/docs/sharing/)
3. Restart OpenTodo and confirm both accounts can still sign in and the task is still complete.

## Publish over HTTPS

Create the owner account locally before publishing. The openGym Compose project creates the shared `homelab-tunnel-origins` Docker network for its cloudflared service. Start the connector from `compose/opengym` if the network has not been created yet.

```powershell
docker compose --profile tunnel up -d --no-deps cloudflared
```

In the local OpenTodo `.env`, set `OPENTODO_BASE_URL` to the permanent public HTTPS address and add `COMPOSE_FILE=compose.yaml;compose.tunnel.yaml` on Windows. Run `docker compose up -d --wait` from `compose/opentodo`. Use the HTTPS hostname for browser sign-in after this change. Accounts keep their passwords. [Base URL and proxy settings](https://open-todo.app/docs/self-hosting/docker/#behind-a-reverse-proxy)

From `compose/opengym`, verify that the connector can resolve the service name and reach OpenTodo from its network namespace.

```powershell
$connector = docker compose --profile tunnel ps -q cloudflared
docker run --rm --network "container:$connector" busybox:1.37.0 wget -qO- http://opentodo:8080/api/v1/health
```

Back up the local tunnel YAML, then add the OpenTodo ingress entry above the final `http_status:404` rule. Keep the existing openGym entries in place.

```yaml
  - hostname: todo.example.com
    service: http://opentodo:8080
```

From `compose/opengym`, validate the updated YAML and check that the HTTPS hostname matches the new origin. Substitute your own hostname in the second command.

```powershell
docker compose --profile tunnel run --rm --no-deps cloudflared tunnel --config /etc/cloudflared/config.yml ingress validate
docker compose --profile tunnel run --rm --no-deps cloudflared tunnel --config /etc/cloudflared/config.yml ingress rule https://todo.example.com
```

Recreate the connector with `docker compose --profile tunnel up -d --no-deps --force-recreate cloudflared`. Check that openGym still responds. Then add the proxied CNAME for OpenTodo with `cloudflared tunnel route dns <tunnel-name-or-id> <hostname>` using the local Cloudflare account certificate, or add that record in Cloudflare DNS. Check `/api/v1/health` at the HTTPS address and confirm both accounts can sign in. [Cloudflare tunnel routing](https://developers.cloudflare.com/tunnel/concepts/routing/)

To roll back publication, remove the OpenTodo DNS record and ingress entry, then recreate the connector. Remove `COMPOSE_FILE` and `OPENTODO_BASE_URL` from the local OpenTodo `.env` and run `docker compose up -d --wait` to restore localhost sign-in.

## Verify phones and host startup

1. Turn off Wi-Fi and use cellular data. In the native client, enter the public HTTPS address and sign in with each person's own account. [iOS app](https://open-todo.app/docs/ios-app/) · [Android app](https://open-todo.app/docs/android-app/) · [Install from the browser](https://open-todo.app/docs/install-phone/)
2. Add a task to the shared project from one phone and confirm it appears on the other phone or in a signed-in browser. Complete it on the second device and confirm the change reaches the first phone.
3. Open the shared project, put the phone in airplane mode, and edit a sample task. Reconnect over cellular and check that the edit reaches the other account. First launch requires a connection. The installed web app supports a full offline launch on iOS. [Offline and sync](https://open-todo.app/docs/offline-and-sync/) · [iOS offline behavior](https://open-todo.app/docs/ios-app/#offline)
4. After a Windows reboot, sign in so Docker Desktop starts. Check `docker compose ps` from `compose/opentodo` and verify HTTPS health and sync again. OpenTodo uses an `always` restart policy.

## Manual upgrades

Read the release notes and create an OpenTodo backup before changing `OPENTODO_VERSION` in the local `.env`. Keep a copy of the archive outside the Docker volume. Apply the new version with `docker compose pull` and `docker compose up -d --wait`, then repeat the sign-in and sync checks. For rollback after a schema migration, use the previous image pin with the pre-upgrade backup. [Release notes](https://github.com/OpenTodoApp/OpenTodo/releases) · [Backup and restore](https://open-todo.app/docs/backups/)
