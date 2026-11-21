# Host setup and Compose

## Prepare Windows

1. Install WSL 2 and Docker Desktop. Use the WSL 2 backend for Linux containers.
2. Enable **Start Docker Desktop when you sign in to your computer** in Docker Desktop. Plan for the configured Windows user to sign in after a reboot so the engine can start.
3. Set plugged-in sleep to **Never** and allow the display to turn off on a timer. Set the battery timeouts for portable use.
4. Check the runtime with `wsl --version`, `docker version`, and `docker compose version`. Run `docker run --rm hello-world` to verify that the engine can pull and start a container.

## Organize projects

Give each application its own directory under `compose/`, with a `compose.yaml` and an `.env.example` containing safe defaults or placeholders. Copy the example to `.env` and fill in local values when a project requires them. Run Compose commands from the project's directory.

Define persistent storage in each project's Compose file. Use named volumes for application-managed data and put bind-mounted application data under the project's `data/` directory. Keep local credentials in the ignored `.env` or under `secrets/` when a service supports reading secrets from files. The repository ignores these paths and local `backups/` directories.

## Test startup after a reboot

From the repository root in PowerShell, start the isolated test container. Its `always` restart policy brings it back when the engine starts, including after a graceful stop.

```powershell
Set-Location compose/smoke-test
docker compose --env-file .env.example config --quiet
docker compose --env-file .env.example up -d
docker compose --env-file .env.example ps
```

Reboot into Windows and sign in so Docker Desktop can start. From `compose/smoke-test`, check the existing container's status and start time. It should be running with a start time after the Windows reboot.

```powershell
docker compose --env-file .env.example ps
$containerId = docker compose --env-file .env.example ps -q probe
docker inspect --format '{{.State.StartedAt}}' $containerId
```

Remove the test container once verification is complete.

```powershell
docker compose --env-file .env.example down
```
