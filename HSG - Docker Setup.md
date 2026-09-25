# Homelab Setup Guide - Docker Setup

A guide to how to get Docker running on TrueNAS and Unraid, plus the Docker basics you need to understand the container guides.
This guide assumes your OS is already installed: [TrueNAS Setup](HSG%20-%20TrueNAS%20Setup.md) or [Unraid Setup](HSG%20-%20Unraid%20Setup.md).
Back to the overview: [HSG - Server Software](HSG%20-%20Server%20Software.md)

---

### Table of Contents

| No. | Topic                                                                 | Software used                    | Description                                                      |
| --- | --------------------------------------------------------------------- | -------------------------------- | ---------------------------------------------------------------- |
| 1   | [Docker on TrueNAS](#1-docker-on-truenas)                             | TrueNAS Apps, Docker Compose     | Enable apps, install catalog apps and custom apps via YAML       |
| 1.1 | [Enable apps](#11-enable-apps)                                        | TrueNAS Apps                     | Choose the apps pool                                             |
| 1.2 | [Catalog apps vs. custom apps](#12-catalog-apps-vs-custom-apps)       | TrueNAS Apps                     | Which install method to use                                      |
| 1.3 | [Install a custom app via YAML](#13-install-a-custom-app-via-yaml)    | Docker Compose                   | Deploy the compose files from the container guide                |
| 1.4 | [Updating & managing apps](#14-updating--managing-apps)               | TrueNAS Apps                     | Logs, shell, updates                                             |
| 2   | [Docker on Unraid](#2-docker-on-unraid)                               | Docker, Community Applications   | Enable Docker, install templates and compose stacks              |
| 2.1 | [Enable Docker](#21-enable-docker)                                    | Docker                           | Docker settings                                                  |
| 2.2 | [Community Applications templates](#22-community-applications-templates) | Community Applications        | Installing containers the "Unraid way"                           |
| 2.3 | [Docker Compose Manager](#23-docker-compose-manager)                  | Docker Compose Manager           | Deploy the compose files from the container guide                |
| 2.4 | [Updating & managing containers](#24-updating--managing-containers)   | Docker                           | Logs, console, updates                                           |
| 3   | [Docker basics](#3-docker-basics)                                     | Docker, Docker Compose           | Images, volumes, ports, networks, environment variables          |
| 4   | [Verification & Troubleshooting](#4-verification--troubleshooting)    | –                                | Check that everything works                                      |

---

## 1. Docker on TrueNAS

Since TrueNAS 24.10, apps run on plain **Docker** (earlier versions used Kubernetes). Every app is a Docker Compose project under the hood.

### 1.1 Enable apps

1. Go to **Apps**.
2. TrueNAS asks you to **Choose a pool** for apps. Select your SSD pool (e.g. `fast`) if you have one, otherwise `tank`.
3. TrueNAS creates a hidden dataset `ix-apps` on this pool for Docker images and internal data. Don't touch it.

**Optional – Nvidia GPU:** **Apps → Configure → Settings** → tick **Install NVIDIA Drivers**. Intel iGPUs (Quick Sync) work out of the box.

### 1.2 Catalog apps vs. custom apps

|                        | Catalog app (**Discover Apps**)                         | Custom app via YAML                                           |
| ---------------------- | ------------------------------------------------------- | ------------------------------------------------------------- |
| **How**                | Form with fields for paths, ports, users                | Paste a Docker Compose file                                   |
| **Pros**               | Easy, maintained by iX / community, update button       | Full control, same file works on Unraid and any Linux host    |
| **Cons**               | Only options the form offers, TrueNAS-specific          | You maintain the compose file yourself                        |

This repo uses **custom apps via YAML** so every guide works on both systems. If you prefer the catalog version of an app, use the paths and settings from the container guide in the form.

### 1.3 Install a custom app via YAML

1. Go to **Apps → Discover Apps**.
2. Click the **⋮** menu (top right) → **Install via YAML**.
3. **Name:** lowercase name of the app, e.g. `jellyfin`.
4. **Custom Config:** paste the compose file from [HSG - Docker Containers](HSG%20-%20Docker%20Containers.md).
5. Replace all placeholders (`<CHANGE_ME>`, paths, IDs) with your own values.
6. Click **Save**. TrueNAS pulls the images and starts the app.
7. The app appears under **Apps → Installed**. The status should change to **Running**.

> **Host paths must exist before you deploy.** TrueNAS won't create missing folders under `/mnt/...` for you. Create them first via **System → Shell**, e.g. `mkdir -p /mnt/tank/appdata/jellyfin/{config,cache}`.

> **No `.env` files:** TrueNAS custom apps don't support `env_file:` with relative paths. All compose files in this repo therefore write environment variables directly into the file.

### 1.4 Updating & managing apps

Select the app under **Apps → Installed**:

| Task             | How                                                                                         |
| ---------------- | ------------------------------------------------------------------------------------------- |
| View logs        | **Workloads → Containers → Logs icon** next to the container                                |
| Open a shell     | **Workloads → Containers → Shell icon**                                                     |
| Edit compose     | **Edit** → change YAML → **Save** (the app is redeployed)                                   |
| Update           | Click **Update** if TrueNAS shows an update, or change the image tag in **Edit** and save   |
| Stop / start     | **Stop** / **Start** buttons at the top                                                     |

Console alternative (**System → Shell**):

```bash
sudo docker ps                     # all running containers
sudo docker logs -f <container>    # follow the logs of a container
sudo docker exec -it <container> sh  # open a shell inside a container
```

> Don't create containers manually with `docker run` or `docker compose` in the shell. TrueNAS doesn't know about them and they can disappear after updates. Always use the **Apps** UI.

---

## 2. Docker on Unraid

### 2.1 Enable Docker

1. Go to **Settings → Docker**.
2. Recommended settings (the array must be **stopped** to change some of them):

| Setting                          | Value                                                           |
| -------------------------------- | --------------------------------------------------------------- |
| Enable Docker                    | `Yes`                                                           |
| Docker data-root                 | `directory` (more flexible than a vDisk image)                  |
| Docker directory                 | `/mnt/cache/system/docker/docker`                               |
| Default appdata storage location | `/mnt/user/appdata/`                                            |
| Docker custom network type       | `ipvlan` (default, fine for most setups)                        |
| Host access to custom networks   | `Enabled` if containers in custom networks must reach the host  |

3. **Apply**. A **Docker** tab appears in the top menu.

**Optional – GPU for transcoding:** install **Intel GPU TOP** (Intel iGPU) or **Nvidia Driver** (Nvidia GPU) from the **Apps** tab and reboot.

### 2.2 Community Applications templates

The "Unraid way" to install containers is via **templates** in the **Apps** tab:

1. Go to **Apps**, search for the container (e.g. `jellyfin`).
2. Prefer templates from **linuxserver** or the **official** image of the project.
3. Click **Install**. A form opens with ports, paths and variables.
4. Adjust the paths to the [recommended folder structure](HSG%20-%20Server%20Software.md#4-recommended-folder-structure), e.g. `/data` → `/mnt/user/data/`.
5. Click **Apply**.

Templates are great for single containers. For apps consisting of multiple containers (Immich, Nextcloud) use **Docker Compose Manager**.

### 2.3 Docker Compose Manager

1. **Apps** → search **Docker Compose Manager** → **Install**.
2. Go to the **Docker** tab. At the bottom there's a new **Compose** section.
3. Click **Add New Stack**, enter a name (e.g. `immich`).
4. Click the gear icon of the stack → **Edit Stack → Compose File**.
5. Paste the compose file from [HSG - Docker Containers](HSG%20-%20Docker%20Containers.md), adjust paths and IDs (see the [Unraid path table](HSG%20-%20Docker%20Containers.md#how-to-use-the-compose-files)), **Save Changes**.
6. Click **Compose Up**. A window shows the progress.
7. Optional: enable **Autostart** for the stack.

> Compose Manager also supports `.env` files (**Edit Stack → ENV File**), but the compose files in this repo work without them.

### 2.4 Updating & managing containers

| Task             | Template containers                                        | Compose stacks                                        |
| ---------------- | ---------------------------------------------------------- | ----------------------------------------------------- |
| View logs        | **Docker** tab → click container icon → **Logs**           | Container icon → **Logs**                             |
| Open a shell     | Container icon → **Console**                               | Container icon → **Console**                          |
| Update           | **Docker** tab → **Check for Updates** → **Apply Update**  | Gear icon of the stack → **Update Stack**             |
| Autostart        | Toggle in the **Autostart** column                         | **Autostart** toggle of the stack                     |

---

## 3. Docker basics

You don't need to be a Docker expert, but these terms appear in every container guide.

```
 ┌────────────── Host (TrueNAS / Unraid) ──────────────┐
 │                                                     │
 │  /mnt/tank/appdata/jellyfin ◄──── volume ────┐      │
 │  /mnt/tank/data/media ◄────────── volume ──┐ │      │
 │                                            │ │      │
 │                   ┌─── container ──────────┴─┴──┐   │
 │  port 8096 ◄──────┤  jellyfin (image)           │   │
 │                   │  /data/media   /config      │   │
 │                   └─────────────────────────────┘   │
 └─────────────────────────────────────────────────────┘
```

| Term                  | Meaning                                                                                                          |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Image**             | The "installer" of an app, downloaded from a registry (Docker Hub, `ghcr.io`, `lscr.io`).                        |
| **Tag**               | Version of an image, e.g. `postgres:16-alpine` or `:latest`.                                              |
| **Container**         | A running instance of an image. Containers are disposable, everything important lives in volumes.               |
| **Volume / bind mount** | A folder on the host mapped into the container: `host_path:container_path`. Delete the container, the data stays. |
| **Port mapping**      | `host_port:container_port`. `8097:8096` makes the container's port `8096` reachable on port `8097` of the server. |
| **Environment variable** | Settings passed to the container, e.g. `TZ=Europe/Berlin`, `PUID=568`.                                        |
| **Network**           | Containers in the same compose file share a network and can reach each other by **service name** (e.g. `database:5432`). |
| **Restart policy**    | `unless-stopped` / `always`: container starts again after a crash or reboot.                                     |

### 3.1 Anatomy of a compose file

```yaml
services:
  jellyfin:                              # service name (= hostname inside the network)
    image: jellyfin/jellyfin:latest      # image:tag
    container_name: jellyfin             # name shown in the UI
    user: 568:568                        # run as this UID:GID (see PUID/PGID)
    environment:
      - TZ=Europe/Berlin                 # environment variables
    volumes:
      - /mnt/tank/appdata/jellyfin/config:/config     # host:container
      - /mnt/tank/data/media:/data/media:ro           # ":ro" = read-only
    ports:
      - 8096:8096                        # host:container
    restart: unless-stopped
```

### 3.2 `:latest` or a fixed version?

| Tag          | Pros                                   | Cons                                                  |
| ------------ | -------------------------------------- | ----------------------------------------------------- |
| `:latest`    | Always newest version, no maintenance  | Breaking changes can arrive unannounced               |
| `:1.2.3`     | Predictable, you decide when to update | You have to update the tag manually                   |

**Recommendation:** `:latest` is fine for Jellyfin, Sonarr, Radarr, Jackett, Prowlarr and Deluge. For **Immich** and **Nextcloud** stick to the major versions used in the guide and **read the release notes before updating**, because database migrations can break things.

### 3.3 Backing up containers

Everything needed to restore a container is in:

1. its **compose file** (save a copy in a Git repo or on your PC), and
2. its **appdata folder**.

Stop containers with databases before backing up their appdata folder, or use the built-in backup features (Immich and Nextcloud both document database dumps).

---

## 4. Verification & Troubleshooting

**Verify:**

- [ ] TrueNAS: **Apps** shows the apps pool as configured. / Unraid: **Docker** tab is visible.
- [ ] A test container (e.g. Jellyfin) starts and its web UI is reachable at `http://192.168.xxx.xxx:{port}`.
- [ ] After a reboot of the server, the container starts automatically.

**Common problems:**

| Problem                                            | Likely cause / fix                                                                                                    |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `port is already allocated`                        | Another container (or the TrueNAS UI on `80`/`443`) uses the port. Change the **host** port, e.g. `8081:8080`.         |
| `permission denied` in the container logs          | The folder isn't owned by the user the container runs as. TrueNAS: `apps` (`568`), Unraid: `nobody:users` (`99:100`). |
| `no such file or directory` for a volume (TrueNAS) | Host path doesn't exist yet. Create it with `mkdir -p` first.                                                          |
| Container restarts in a loop                       | Check the logs. Most often a wrong path, missing variable or a database that isn't ready.                            |
| Image pull fails with `toomanyrequests`            | Docker Hub rate limit. Wait a few hours, or use the `ghcr.io` / `lscr.io` image where available.                       |
| Containers can't reach each other by name          | They are in different compose files/apps. Use `192.168.xxx.xxx:{port}` instead, or put them into one compose file.   |

**Useful commands** (TrueNAS: **System → Shell**, Unraid: terminal icon):

```bash
docker ps -a                          # all containers incl. stopped ones
docker logs -f --tail 100 <container> # last 100 log lines, then follow
docker exec -it <container> sh        # shell inside the container
docker stats                          # CPU / RAM usage per container
docker image prune -a                 # delete unused images (frees space)
```

> On TrueNAS prefix the commands with `sudo` if you're not logged in as root.

---

**Next step:** Set up your containers → [HSG - Docker Containers](HSG%20-%20Docker%20Containers.md)
