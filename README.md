# FileMaker Server for Docker

### Contents

- [Quickstart](#quickstart)
- [How the image works](#how-the-image-works)
- [Choose an image](#choose-an-image)
- [Docker run](#docker-run)
- [Docker Compose](#docker-compose)
- [Administrator environment variables](#administrator-environment-variables)
- [Storage and data retention](#storage-and-data-retention)
- [Bind mounts](#bind-mounts)
- [Backups and recovery](#backups-and-recovery)
- [Upgrade routes](#upgrade-routes)
- [Upgrade with Docker run](#upgrade-with-docker-run)
- [Upgrade with Compose](#upgrade-with-compose)
- [Troubleshooting](#troubleshooting)

## Quickstart

FileMaker Server 21, 22, and 26 on Ubuntu, with persistent storage and startup checks for upgrades. Images support AMD64 and ARM64; Docker selects the matching architecture.

**Image:** `ghcr.io/tstaehli84/filemaker-server`

### Before you start

First-start installation requires internet access. Configure your FileMaker Server license after setup.

Both examples deploy `26.0.3-ubuntu24` using the same persistent storage. Use either Docker run or Compose.

### Option 1: Docker run

```sh
docker run -d \
  --name fms \
  --hostname fms \
  --privileged \
  --restart unless-stopped \
  --stop-timeout 135 \
  -p 80:80 \
  -p 443:443 \
  -p 5003:5003 \
  -v "fms-data:/opt/FileMaker/FileMaker Server" \
  -v "fms-state:/var/lib/fms-docker" \
  ghcr.io/tstaehli84/filemaker-server:26.0.3-ubuntu24
```

See [Startup and access](#startup-and-access) for installation progress and connection details.

### Option 2: Docker Compose

Use [compose.yaml](compose.yaml):

```yaml
name: fms
services:
  fms:
    image: ghcr.io/tstaehli84/filemaker-server:26.0.3-ubuntu24
    container_name: fms
    hostname: fms
    privileged: true
    restart: unless-stopped
    stop_grace_period: 135s
    ports:
      - "80:80"
      - "443:443"
      - "5003:5003"
    volumes:
      - fms-data:/opt/FileMaker/FileMaker Server
      - fms-state:/var/lib/fms-docker
volumes:
  fms-data:
    name: fms-data
  fms-state:
    name: fms-state

```

```sh
docker compose up -d
docker compose logs -f fms
```

### Startup and access

**The first start takes several minutes:** the container installs FileMaker Server and its dependencies. A running container may still be installing.

Check the container’s **Logs** tab in Docker Desktop, or follow progress from the terminal:

```sh
# Docker run
docker logs -f fms

# Docker Compose
docker compose logs -f fms
```

Wait for the startup confirmation:

```text
[FMS] READY: FileMaker Server startup completed successfully; fmshelper.service is running.
```

- **Admin Console:** open <https://localhost/admin-console> when Docker is on your computer. For another computer, replace `localhost` with the Docker host’s IP address or DNS name.
- **Initial login:** username `admin`, password `1234`, PIN `0000` for a fresh installation unless overridden with [administrator environment variables](#administrator-environment-variables). Change the password in the admin console, then configure your license and TLS certificate. Upgraded installations retain their saved credentials.
- **FileMaker Pro:** add the same host address in Hosts. The connection uses port `5003`.

The examples publish ports on all host interfaces. Restrict access as appropriate and replace the initial credentials and certificate before exposing the server to untrusted networks.

Check status with `docker ps` or `docker compose ps`. `healthy` confirms startup and helper-service checks; also verify Admin Console login and your hosted databases.

### Storage and upgrades

The examples save the whole FileMaker Server folder in `fms-data` and startup history in `fms-state`. Keep **both** volumes when replacing or upgrading a container. Persistent storage is not a backup.

**Do not run `docker compose down -v` or delete these volumes unless you intend to erase the server’s saved files.**

For upgrades, select a specific release with the **same Ubuntu suffix**, back up both volumes while stopped, then recreate the container using the original mounts. Downgrades and in-place Ubuntu changes are blocked.

The sections below cover image tags, container management, upgrade routes, volumes and bind mounts, backups, rollback, and troubleshooting.

## How the image works

FileMaker Server is installed from the bundled package at container startup. Services run under systemd and require `privileged: true` in Compose or `--privileged` with Docker run.

First startup takes several minutes and requires internet access for package metadata and dependencies. Monitor it in Docker Desktop’s container **Logs** tab or from the terminal:

```sh
# Docker run:
docker logs -f fms
# Compose:
docker compose logs -f fms
```

Messages beginning with `[FMS]` describe installation, configuration restoration, upgrade checks, and errors. Wait for:

```text
[FMS] READY: FileMaker Server startup completed successfully; fmshelper.service is running.
```

The health status remains `starting` while installation runs. Successful startup and an active helper service produce `healthy`; startup failures produce `unhealthy`. Health does not prove that every database is open or that authentication works. Check Admin Console, FileMaker Pro, schedules, licensing, and web features you use.

Restarting the **same container** skips installation when the exact package is already installed. A **replacement container** has a new package database and installs the selected package using the saved configuration. Even replacing a container with the same image can therefore take time.

## Choose an image

All examples use GitHub Container Registry:

```text
ghcr.io/tstaehli84/filemaker-server:26.0.3-ubuntu24
```

The tag combines the FileMaker release and container OS: `26.0.3-ubuntu24` selects FileMaker Server 26.0.3 on Ubuntu 24.04. 

| Tag style | Meaning |
| --- | --- |
| `21.1.8-ubuntu22` | Specific FileMaker 21 release on Ubuntu 22.04 |
| `22.0.7-ubuntu22` | Specific FileMaker 22 release on Ubuntu 22.04 |
| `22.0.7-ubuntu24` | Specific FileMaker 22 release on Ubuntu 24.04 |
| `26.0.3-ubuntu22` | Specific FileMaker 26 release on Ubuntu 22.04 |
| `26.0.3-ubuntu24` | Specific FileMaker 26 release on Ubuntu 24.04 |
| `21`, `22`, `26` | Moving major-series aliases; Ubuntu 22 for 21, Ubuntu 24 for 22/26 |
| `22-ubuntu22`, `26-ubuntu22` | Moving major-series aliases with an explicit Ubuntu version |
| `latest` | Moving default alias for FileMaker 26 on Ubuntu 24 |

Tags omit the installer’s final build number: `26.0.3.309` becomes `26.0.3`. FileMaker 21 has Ubuntu 22 variants only; 22 and 26 can have Ubuntu 22 and 24 variants. Availability depends on which images have been published.

Pin the release and Ubuntu variant for production deployments, for example `26.0.3-ubuntu24`.

## Docker run

Use the [Docker run example](#option-1-docker-run) with both persistent mounts. The image requires privileged mode and uses a 135-second shutdown timeout for graceful FileMaker service shutdown.

| Port | Service |
| --- | --- |
| `80`, `443` | Web services and Admin Console |
| `5003` | FileMaker client connections |
| `2399` | Optional ODBC/JDBC; enable in FileMaker Server |
| `22` | Optional SSH for standby configurations; requires an image with SSH installed |

## Docker Compose

Use the supplied [compose.yaml](compose.yaml):

```sh
docker compose up -d fms
docker compose logs -f fms
```

For additional server instances, assign unique container names, volume names, hostnames, and host ports. Each instance must have its own FileMaker storage and startup state.

## Administrator environment variables

Set these optional variables before the first installation to choose the Admin Console credentials:

| Variable | Purpose | Default when empty or unset |
| --- | --- | --- |
| `ADMIN_USER` | Admin Console username | `admin` |
| `ADMIN_PWD` | Admin Console password | `1234` |
| `ADMIN_PIN` | Four-digit administrator PIN | `0000` |

Each supplied value replaces the matching field in a temporary assisted-install file; the bundled template is unchanged. You can set any combination of the variables. Values must be on a single line, and `ADMIN_PIN` must contain exactly four digits.

**These variables apply only to a fresh installation with no saved server configuration.** On restarts, replacement containers using existing configuration, and upgrades, existing credentials are preserved. Changing these variables does not reset an existing password, rename the administrator, or change its PIN.

### With Docker Compose

Add an `environment` mapping under the `fms` service in your Compose file, alongside `image`, `ports` and `volumes`:

```yaml
name: fms
services:
  fms:
    # Keep the other service settings from the Quickstart.
    environment:
      ADMIN_USER: "myadmin"
      ADMIN_PWD: "Choose-A-Strong-Password!42"
      ADMIN_PIN: "0123"
```

Quote the PIN to retain leading zeroes. For a literal `$` entered directly in Compose YAML, write `$$`.

Alternatively, reference a `.env` file beside `compose.yaml`:

```yaml
    environment:
      ADMIN_USER: ${ADMIN_USER:-}
      ADMIN_PWD: ${ADMIN_PWD:-}
      ADMIN_PIN: ${ADMIN_PIN:-}
```

Example `.env` contents:

```dotenv
ADMIN_USER=myadmin
ADMIN_PWD='Choose-A-Strong-Password!42'
ADMIN_PIN=0123
```

The `environment` mapping is required: a `.env` file alone does not pass these variables into the container. Then start the service with `docker compose up -d fms`.

### With Docker run

Add these options to the Quickstart's `docker run` command, before the image name:

```sh
  -e ADMIN_USER='myadmin' \
  -e ADMIN_PWD='Choose-A-Strong-Password!42' \
  -e ADMIN_PIN='0123' \
```

Choose your own values before starting the fresh server. Keep credential files out of version control; Docker environment inspection can reveal supplied values.

## Storage and data retention

Choose either of the following storage layouts. Both retain server data and startup history across container replacements; the difference is whether application files are retained alongside your configuration.

### Option 1: Two mounts for the whole server and startup state

This is the layout used in the Quickstart. It is convenient for backups and same-Ubuntu image upgrades because the entire FileMaker Server folder is retained.

| Volume | Container path | What it retains |
| --- | --- | --- |
| `fms-data` | `/opt/FileMaker/FileMaker Server` | The whole server folder, including databases, default backups, preferences, schedules, certificates, license files, logs, scripts, and application files stored there |
| `fms-state` | `/var/lib/fms-docker` | Last successful package version, original container Ubuntu version, and installation state |

### Option 2: Individual mounts for data and configuration

Mount the persistent folders separately, leaving FileMaker application binaries and other installer-managed files in the container. This gives more control over what is restored during a migration, particularly when moving to a different Ubuntu release: the destination installer creates its own application files, while selected data and compatible configuration can be restored separately.

| Example volume | Container path | What it retains |
| --- | --- | --- |
| `fms-server-data` | `/opt/FileMaker/FileMaker Server/Data` | Databases, default backups, server preferences, schedules, scripts and documents |
| `fms-cstore` | `/opt/FileMaker/FileMaker Server/CStore` | Certificates, keys and license-related files |
| `fms-wpeconf` | `/opt/FileMaker/FileMaker Server/Web Publishing/publishing-engine/conf` | Web publishing configuration |
| `fms-logs` | `/opt/FileMaker/FileMaker Server/Logs` | Server logs |
| `fms-state` | `/var/lib/fms-docker` | Package version history, original Ubuntu version and installation state |

For Compose, replace the Quickstart's volume mappings and top-level declarations with:

```yaml
name: fms
services:
  fms:
    # Keep the image, ports and other service settings from the Quickstart.
    volumes:
      - fms-server-data:/opt/FileMaker/FileMaker Server/Data
      - fms-cstore:/opt/FileMaker/FileMaker Server/CStore
      - fms-wpeconf:/opt/FileMaker/FileMaker Server/Web Publishing/publishing-engine/conf
      - fms-logs:/opt/FileMaker/FileMaker Server/Logs
      - fms-state:/var/lib/fms-docker
volumes:
  fms-server-data:
    name: fms-server-data
  fms-cstore:
    name: fms-cstore
  fms-wpeconf:
    name: fms-wpeconf
  fms-logs:
    name: fms-logs
  fms-state:
    name: fms-state
```

For Docker run, replace the two Quickstart mounts with:

```sh
  -v "fms-server-data:/opt/FileMaker/FileMaker Server/Data" \
  -v "fms-cstore:/opt/FileMaker/FileMaker Server/CStore" \
  -v "fms-wpeconf:/opt/FileMaker/FileMaker Server/Web Publishing/publishing-engine/conf" \
  -v "fms-logs:/opt/FileMaker/FileMaker Server/Logs" \
  -v "fms-state:/var/lib/fms-docker" \
```

Plug-ins, custom web files and other files outside these mounted folders need separate mounts or a documented reinstallation/restore procedure. In particular, check `Database Server/Extensions` and custom files under `HTTPServer/htdocs`.

**Individual mounts do not enable an in-place Ubuntu upgrade.** The startup OS check still blocks changing Ubuntu while reusing the existing installation and state. Use a separate migration into fresh destination storage, following the destination FileMaker version's supported restore procedures. Restore only compatible data/configuration and reinstall compatible plug-ins; do not remove or edit the original OS marker to bypass the check. See [Upgrade routes](#upgrade-routes).

### FileMaker folder contents

The following paths are relative to `/opt/FileMaker/FileMaker Server`. Option 1 covers all of them through the whole-server `fms-data` mount. With Option 2, the `Data`, `CStore`, web publishing configuration and `Logs` mounts cover the corresponding rows; plug-ins and custom web files need separate handling.

| Folder | Retained content and recovery considerations |
| --- | --- |
| `Data/Databases` | Hosted `.fmp12` files in the default database location. Retain the associated external container data as well as the database files. Additional database folders outside the server directory require separate persistence. |
| `Data/Databases/RC_Data_FMS` | External container data for databases using this default location. Preserve its directory structure and restore it together with the corresponding databases. Container data configured elsewhere must be included separately. |
| `Data/Backups` | Backups stored at the default backup destination, including retained backup sets. Custom backup and progressive-backup destinations are covered only if they are inside the mounted server folder. |
| `Data/Preferences` | Saved server configuration, including `dbs_config.xml`, administrator configuration, and schedule/settings information. Startup uses these preferences to retain the existing configuration during installation or upgrade. Database files alone do not restore these settings. |
| `Data/Scripts` | System-level scripts used by FileMaker Server schedules. Script dependencies outside the mounted folder need separate persistence. |
| `Data/Documents` | Files read or written by server-side FileMaker scripts in the server Documents folder. Include these when restoring solutions that depend on exported or imported files. |
| `CStore` | Certificate material, private keys, and license-related files. Retain the directory with the server configuration to preserve TLS and licensing information. Treat backups containing private keys as sensitive. |
| `Web Publishing/publishing-engine/conf` | Web publishing configuration. Retain it alongside server preferences when restoring WebDirect or other web publishing services. |
| `Logs` | FileMaker Server logs, including event, access, and statistics logs when enabled. Existing logs remain available after replacement; their rotation and retention follow the FileMaker configuration. |
| `Database Server/Extensions` | Server-side plug-ins installed in this location. Files persist, but plug-in compatibility and any external libraries or licensing dependencies must be checked for the destination FileMaker version and architecture. |
| `HTTPServer/htdocs` | Custom web files stored under this directory, including `httpsRoot`. Installer-managed web files may be replaced during an upgrade. |

The full-folder mount also retains the remaining application directories and custom files under the server root. **Persistence does not make installer-managed files immutable:** an upgrade can replace application files and update configuration. Keep a stopped snapshot for recovery of the pre-upgrade state.

### Startup state

`/var/lib/fms-docker` is retained separately in `fms-state`:

| File | Purpose |
| --- | --- |
| `package-version` | Last successfully installed package version; used to reject downgrades. |
| `os-version` | Original container Ubuntu release; used to reject an in-place OS change. |
| `installing-version` | Target version recorded during installation. Removed after success and retained for diagnosis after failure. |
| `install.lock` | Lock file used to prevent concurrent startup operations against the same state directory. |

For same-Ubuntu recovery, restore startup state from the same snapshot as the FileMaker folder or individual persistent folders. It contains no hosted databases, but is required to preserve installation history and upgrade checks.

## Bind mounts

To use host directories instead of named volumes, replace the service’s `volumes` section with:

```yaml
    volumes:
      - "/srv/fms/data:/opt/FileMaker/FileMaker Server"
      - "/srv/fms/state:/var/lib/fms-docker"
```

Remove the unused top-level volume declarations. Both directories must be writable by the container’s FileMaker services. Use empty directories for a fresh installation, or restore the server folder and matching startup state from the same stopped snapshot.

For Docker run, use:

```sh
  -v "/srv/fms/data:/opt/FileMaker/FileMaker Server" \
  -v "/srv/fms/state:/var/lib/fms-docker" \
```

## Backups and recovery

Use FileMaker Server’s scheduled backups for databases, with copies stored outside the Docker host. Before an upgrade or migration, also capture a complete stopped-server snapshot:

1. Run and verify a FileMaker backup.
2. Stop the server with `docker stop fms` or `docker compose stop fms`.
3. Snapshot or back up both persistent mounts and any custom storage, preserving ownership and permissions.
4. Save the image reference and deployment configuration with the snapshot.

The server folder and startup state must be backed up and restored together. Use FileMaker’s backup process for live databases; take filesystem snapshots for full-server recovery while the server is stopped.

### Rollback

Downgrade protection compares the target package with the last successful version recorded in startup state.

Stop the upgraded server. Restore the **entire matching pre-upgrade snapshot**, including startup state and custom paths, into separate volumes or host folders. Start the original image tag against that restored storage, with the original container Ubuntu version. Verify the restored server before switching clients back. Leave the upgraded storage available for investigation, and avoid two servers publishing the same host ports.

Retain the version and OS markers from the matching snapshot; do not modify them to force a downgrade.

## Upgrade routes

At startup, the image validates the target package against saved version and Ubuntu markers, then installs using the existing FileMaker preferences.

**In-place upgrades must retain the container’s Ubuntu release.**

| Example route | Image startup behavior |
| --- | --- |
| `26.0.2-ubuntu24` → `26.0.3-ubuntu24` | A newer package on the same Ubuntu release can proceed |
| `21.1.8-ubuntu22` → `22.0.4-ubuntu22` → `26.0.3-ubuntu22` | A newer package on the same Ubuntu release can proceed |
| `21.1.8-ubuntu22` → `26.0.3-ubuntu24` | Blocked: container Ubuntu release changes |
| `26.0.3-ubuntu22` → `26.0.3-ubuntu24` | Blocked: Ubuntu realease change not supported |
| `26.0.3-ubuntu24` → `26.0.2-ubuntu24` | Blocked: can't downgrade to an earlier package |

The image’s checks prevent certain unsafe operations; they do not validate every Claris-supported upgrade route. Consult the Claris upgrade instructions for your source and destination release, licensing, and client compatibility before choosing a route. Use intermediate versions where required.

To change Ubuntu versions, plan a separate migration into fresh storage using the destination version’s supported FileMaker backup/restore and configuration procedures. Reusing the existing volumes under a different Ubuntu tag is blocked. Do not remove the OS marker to bypass this protection.

## Upgrade with Docker run

The example below assumes your container is named `fms` and uses the two Quickstart volumes. Choose the destination tag with the **same Ubuntu suffix as your current server**. For an Ubuntu 24 server moving to 26.0.3:

1. Download the target image while the existing server is still available:

   ```sh
   docker pull ghcr.io/tstaehli84/filemaker-server:26.0.3-ubuntu24
   ```

2. Make a FileMaker backup, then stop the container:

   ```sh
   docker stop fms
   ```

3. Take a complete stopped-server snapshot, including both volumes and any custom mounts.
4. Remove the stopped container:

   ```sh
   docker rm fms
   ```

5. Run the [Quickstart Docker run command](#option-1-docker-run) with the destination image tag, keeping the **same two volume names**, hostname, ports, and any custom mounts/options from your original server.
6. Follow `docker logs -f fms`. Allow time for installation. After `READY`, verify Admin Console login, hosted databases, client read/write, backup schedules, license, certificates, and any web services or plug-ins you use.

## Upgrade with Compose

1. Confirm the current image tag in `compose.yaml`. Select the destination release on the **same Ubuntu version**.
2. Make a FileMaker backup and stop the service:

   ```sh
   docker compose stop
   ```

3. Take a complete stopped backup of both persistent volumes and all custom paths. Save the original Compose configuration with it.
4. Change only the `image:` tag in `compose.yaml` to the selected destination, for example:

   ```yaml
   image: ghcr.io/tstaehli84/filemaker-server:26.0.3-ubuntu24
   ```

5. Download the image and recreate the service:

   ```sh
   docker compose pull fms
   docker compose up -d fms
   docker compose logs -f fms
   ```

6. Wait for `READY` and check `docker compose ps`. Verify login, hosted databases, client read/write, schedules, backups, license, TLS, and your integrations.

Keep volume names and paths unchanged. If installation fails, inspect the logs before recovery; persistent files may already have been modified.

## Troubleshooting

### The container is running, but I cannot connect

Check the logs first. First startup or a replacement-container installation may still be running. Look for `READY` or an `ERROR` message. Check `docker ps` / `docker compose ps`, verify port mappings, and use the Docker host’s address rather than the container’s internal IP.

### Installation fails or health is unhealthy

Inspect recent logs:

```sh
docker logs --tail 200 fms
# Or:
docker compose logs --tail 200 fms
```

For detailed installation and service output:

```sh
# Docker run:
docker exec fms journalctl -u installcheck.service --no-pager
docker exec fms systemctl status fmshelper.service --no-pager
# Compose:
docker compose exec fms journalctl -u installcheck.service --no-pager
docker compose exec fms systemctl status fmshelper.service --no-pager
```

Check free disk space, network access, mount permissions, and the first error above the final failure summary. A failed health check does not by itself restart or stop the container; it stays available for investigation.

### “Refusing container OS change” or “Refusing downgrade”

You selected an incompatible tag for the saved storage. Select a compatible same-Ubuntu image, or restore the complete corresponding backup for rollback. Do not edit the stored markers to force startup.

### “Existing data found without server preferences”

Startup found saved data but no usable `Data/Preferences/dbs_config.xml`. This can indicate incomplete migration or missing storage. Stop and check that the correct full server folder and matching state are mounted; restore a complete backup rather than initializing over existing files.

### “Existing installation has no OS marker”

This can occur with installations created before the OS guard existed, or if their state was not preserved. Startup refuses to guess the original Ubuntu version.

First identify the original image and confirm its Ubuntu release, for example:

```sh
# Replace ORIGINAL_IMAGE with the original image reference.
docker run --rm --entrypoint cat ORIGINAL_IMAGE /etc/os-release
```

Stop the server and take a complete backup. For Compose, select an image matching that confirmed original Ubuntu release. Only for a confirmed Ubuntu 22.04 installation, initialize the missing marker with:

```sh
docker compose run --rm --no-deps --entrypoint /bin/bash fms -c '
  set -eu
  . /etc/os-release
  test "$ID:$VERSION_ID" = ubuntu:22.04
  marker=/var/lib/fms-docker/os-version
  test ! -e "$marker"
  umask 077
  printf "%s\n" ubuntu:22.04 > "$marker"
'
```

For a confirmed Ubuntu 24.04 installation, change **both** occurrences of `ubuntu:22.04` to `ubuntu:24.04`. This command uses the configured storage and refuses to overwrite an existing marker. Confirm the source OS independently; do not infer it from the destination image. If the rest of the startup state was lost, recover it from backup rather than treating this marker as a complete state restore.

With Docker run, perform the equivalent one-time operation in a temporary container using the original server’s exact mounts and a matching image, while the FileMaker container is stopped. Keep the same source-OS verification and no-overwrite checks.
