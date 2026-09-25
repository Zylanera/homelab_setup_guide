# Homelab Setup Guide - Docker Containers

A guide to how to set up the individual services of your homelab as Docker containers, each with a ready-to-use Docker Compose file.
This guide assumes Docker is already running: [HSG - Docker Setup](HSG%20-%20Docker%20Setup.md).
Back to the overview: [HSG - Server Software](HSG%20-%20Server%20Software.md)

---

### Table of Contents

| No. | Container                                                           | Software used                         | Description                                                   |
| --- | ------------------------------------------------------------------- | ------------------------------------- | ------------------------------------------------------------- |
| –   | [How to use the compose files](#how-to-use-the-compose-files)       | –                                     | Paths and IDs for TrueNAS and Unraid                          |
| 1   | [Immich](#1-immich)                                                 | Immich, PostgreSQL, Valkey            | Self-hosted Google Photos alternative with mobile backup      |
| 2   | [Jellyfin](#2-jellyfin)                                             | Jellyfin                              | Media server for movies & TV shows                            |
| 3   | [Nextcloud](#3-nextcloud)                                           | Nextcloud, PostgreSQL, Valkey         | Self-hosted cloud storage, calendar & contacts                |
| 4   | [Sonarr](#4-sonarr)                                                 | Sonarr                                | Automatically finds and organizes TV shows                    |
| 5   | [Radarr](#5-radarr)                                                 | Radarr                                | Automatically finds and organizes movies                      |
| 6   | [Jackett](#6-jackett)                                               | Jackett                               | Indexer proxy for Sonarr & Radarr                             |
| 7   | [Prowlarr](#7-prowlarr) *(optional)*                                | Prowlarr                              | Modern alternative to Jackett                                 |
| 8   | [Deluge](#8-deluge)                                                 | Deluge                                | Download client used by Sonarr & Radarr                       |
| 9   | [Media stack in one compose file](#9-media-stack-in-one-compose-file-optional) *(optional)* | Sonarr, Radarr, Jackett/Prowlarr, Deluge | All media automation containers as one app |
| 10  | [Verification & Troubleshooting](#10-verification--troubleshooting) | –                                     | Check that everything works                                   |
| 11  | [Security Checklist](#11-security-checklist)                        | –                                     | Things to double-check before you're done                     |

---

## How to use the compose files

All compose files are written with **TrueNAS paths**. On **Unraid**, replace the values like this before deploying:

| TrueNAS                                  | Unraid                                   | Note                                          |
| ---------------------------------------- | ---------------------------------------- | --------------------------------------------- |
| `/mnt/tank/appdata/<app>`                | `/mnt/user/appdata/<app>`                | App configs                                   |
| `/mnt/tank/appdata/<app>/postgres` (or `/db`) | `/mnt/cache/appdata/<app>/postgres` (or `/db`) | **Databases always on the direct pool path** |
| `/mnt/tank/data`                         | `/mnt/user/data`                         | Your files                                    |
| `PUID=568` / `PGID=568`                  | `PUID=99` / `PGID=100`                   | linuxserver images                            |
| `user: 568:568`                          | `user: 99:100`                           | Official images (Jellyfin)                    |
| `sudo chown -R 568:568 ...`              | `chown -R 99:100 ...`                    | Folder preparation commands                   |

> If you created an SSD pool on TrueNAS (e.g. `fast`), use `/mnt/fast/appdata` instead of `/mnt/tank/appdata`.

**Deploying:** TrueNAS → **Apps → Discover Apps → ⋮ → Install via YAML** ([details](HSG%20-%20Docker%20Setup.md#13-install-a-custom-app-via-yaml)). Unraid → **Docker → Compose → Add New Stack** ([details](HSG%20-%20Docker%20Setup.md#23-docker-compose-manager)).

**Placeholders:** Replace everything in `<ANGLE_BRACKETS>`. Generate passwords with a password manager, **only letters and numbers** for database passwords (special characters break some connection strings).

**Commands:** TrueNAS → **System → Shell**. Unraid → terminal icon (**>_**) top right (you are root there, so no `sudo` needed).

---

## 1. Immich

Immich backs up the photos and videos from your phone automatically, recognizes faces and objects, and looks and feels like Google Photos. It consists of four containers:

| Container                 | Purpose                                                        |
| ------------------------- | -------------------------------------------------------------- |
| `immich_server`           | Web UI, API and background jobs                                |
| `immich_machine_learning` | Face recognition, smart search (CLIP)                          |
| `immich_redis`            | Job queue (Valkey, a Redis-compatible fork)                    |
| `immich_postgres`         | Database (special PostgreSQL image with vector extensions)     |

### 1.1 Prepare folders

```bash
sudo mkdir -p /mnt/tank/appdata/immich/{postgres,model-cache} /mnt/tank/data/immich-lib
```

### 1.2 Compose file

```yaml
services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:v3
    volumes:
      - /mnt/tank/data/immich-lib:/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - DB_PASSWORD=<CHANGE_ME_DB_PASSWORD>   # same as POSTGRES_PASSWORD below
      - DB_USERNAME=postgres
      - DB_DATABASE_NAME=immich
      - TZ=Europe/Berlin
    ports:
      - 2283:2283
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:v3
    volumes:
      - /mnt/tank/appdata/immich/model-cache:/cache
    environment:
      - TZ=Europe/Berlin
    restart: always
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/valkey/valkey:9
    healthcheck:
      test: redis-cli ping | grep -q PONG || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
    environment:
      POSTGRES_PASSWORD: <CHANGE_ME_DB_PASSWORD>
      POSTGRES_USER: postgres
      POSTGRES_DB: immich
      POSTGRES_INITDB_ARGS: '--data-checksums'
      # DB_STORAGE_TYPE: 'HDD'   # uncomment if the database is NOT on an SSD
    volumes:
      - /mnt/tank/appdata/immich/postgres:/var/lib/postgresql/data
    shm_size: 128mb
    restart: always
    healthcheck:
      disable: false
```

> **Based on the official Immich compose file.** The database image changes between Immich releases. Before a new install or a major update, compare the `database` and `redis` images with the [latest official compose file](https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml) and read the [release notes](https://github.com/immich-app/immich/releases).

> **Service names matter:** Immich finds its database and Redis by the hostnames `database` and `redis`. Don't rename these services.

### 1.3 First setup

1. Open `http://192.168.xxx.xxx:2283` and click **Getting Started**.
2. Create the **admin account** (the first user is always admin).
3. **Administration → Users:** create accounts for your family. Optional: set a storage quota per user.
4. **Administration → Settings → Storage Template:** enable it to store files in a readable structure like `2026/2026-09-25/IMG_1234.jpg` instead of random IDs.
5. Install the **Immich app** ([iOS](https://apps.apple.com/app/immich/id1613945652) / [Android](https://play.google.com/store/apps/details?id=app.alextran.immich)), enter the server URL `http://192.168.xxx.xxx:2283` and log in.
6. In the app, tap the cloud icon → select the albums to back up → **enable backup**. Enable **background backup** so it runs without opening the app.

### 1.4 Optional settings

- **Hardware acceleration:** Immich can use your GPU for video transcoding and machine learning (e.g. image tag `v3-openvino` for Intel). See the official docs for [transcoding](https://docs.immich.app/features/hardware-transcoding) and [machine learning](https://docs.immich.app/features/ml-hardware-acceleration).
- **Existing photo folders:** Use **External Libraries** (**Administration → External Libraries**) to show photos that already exist on your NAS without copying them. Mount the folder read-only into `immich-server`, e.g. `- /mnt/tank/data/photos-old:/mnt/photos-old:ro`.
- **Remote access:** Use Tailscale or Cloudflare Tunnel ([HSG - Remote Access](HSG%20-%20Remote%20Access.md)). In the app you can set a **local** and an **external** URL so it switches automatically.

---

## 2. Jellyfin

Jellyfin streams your movies and TV shows to your TV, phone, browser and more. Free, open source, no account required.

### 2.1 Prepare folders

```bash
sudo mkdir -p /mnt/tank/appdata/jellyfin/{config,cache}
sudo chown -R 568:568 /mnt/tank/appdata/jellyfin
```

### 2.2 Compose file

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: 568:568
    # group_add:                # uncomment for hardware transcoding, see 2.4
    #   - "<RENDER_GROUP_ID>"
    # devices:
    #   - /dev/dri:/dev/dri
    environment:
      - TZ=Europe/Berlin
    volumes:
      - /mnt/tank/appdata/jellyfin/config:/config
      - /mnt/tank/appdata/jellyfin/cache:/cache
      - /mnt/tank/data/media:/data/media:ro
    ports:
      - 8096:8096
    restart: unless-stopped
```

> The media folder is mounted **read-only** (`:ro`). Jellyfin only needs to read your files; Sonarr and Radarr are the ones who write.

### 2.3 First setup

1. Open `http://192.168.xxx.xxx:8096` and follow the setup wizard.
2. Create your admin user.
3. **Add Media Library:**

| Content type | Display name | Folder               |
| ------------ | ------------ | -------------------- |
| Movies       | Movies       | `/data/media/movies` |
| Shows        | TV Shows     | `/data/media/tv`     |

4. Set your preferred metadata language and country.
5. Finish the wizard and log in.
6. **Dashboard → Users:** create a user for every family member (don't give them admin rights).

### 2.4 Hardware transcoding (recommended)

Transcoding converts videos on the fly when a device can't play the original format. Without a GPU this is done by the CPU and can max it out.

**Intel iGPU (Quick Sync) / AMD:**

1. Find the group ID of the GPU device:
   ```bash
   stat -c '%g' /dev/dri/renderD128
   ```
   (Unraid: install the **Intel GPU TOP** plugin first.)
2. Uncomment `group_add` and `devices` in the compose file and enter the ID, redeploy.
3. In Jellyfin: **Dashboard → Playback → Transcoding**:
   - **Hardware acceleration:** `Intel QuickSync (QSV)` (Intel) or `Video Acceleration API (VAAPI)` (AMD)
   - Enable hardware decoding for the listed codecs your CPU supports, save.

**Nvidia GPU:** Install the driver ([TrueNAS](HSG%20-%20Docker%20Setup.md#11-enable-apps) / [Unraid](HSG%20-%20Docker%20Setup.md#21-enable-docker)), then add this to the `jellyfin` service instead of `devices`/`group_add` and choose **NVIDIA NVENC** in Jellyfin:

```yaml
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

**Test:** play a movie in the browser, click the gear icon → **Quality** → pick a low bitrate. **Dashboard → Activity** should show the stream as transcoding, with low CPU usage.

---

## 3. Nextcloud

Nextcloud is your own Dropbox / Google Drive with calendar, contacts, photos and office apps. This guide uses the **official image** with PostgreSQL and Valkey (Redis), plus a small container that runs Nextcloud's background jobs.

| Container          | Purpose                                         |
| ------------------ | ----------------------------------------------- |
| `nextcloud`        | Web UI and apps                                 |
| `nextcloud_db`     | PostgreSQL database                             |
| `nextcloud_redis`  | Cache and file locking (Valkey)                 |
| `nextcloud_cron`   | Runs background jobs every 5 minutes            |

### 3.1 Prepare folders

The official image runs as `www-data` (UID `33`). The data folder must belong to this user.

```bash
sudo mkdir -p /mnt/tank/appdata/nextcloud/{html,db}
# TrueNAS: /mnt/tank/data/nextcloud is the child dataset from the TrueNAS guide (owner www-data)
# Unraid:
chown -R 33:33 /mnt/user/data/nextcloud && chmod 770 /mnt/user/data/nextcloud
```

### 3.2 Compose file

```yaml
services:
  db:
    image: postgres:16-alpine
    container_name: nextcloud_db
    environment:
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=<CHANGE_ME_DB_PASSWORD>
    volumes:
      - /mnt/tank/appdata/nextcloud/db:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: valkey/valkey:8-alpine
    container_name: nextcloud_redis
    restart: unless-stopped

  app:
    image: nextcloud:stable
    container_name: nextcloud
    depends_on:
      - db
      - redis
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=<CHANGE_ME_DB_PASSWORD>
      - REDIS_HOST=redis
      - NEXTCLOUD_ADMIN_USER=<ADMIN_USERNAME>
      - NEXTCLOUD_ADMIN_PASSWORD=<CHANGE_ME_ADMIN_PASSWORD>
      - NEXTCLOUD_TRUSTED_DOMAINS=192.168.xxx.xxx cloud.example.com
      - TRUSTED_PROXIES=<IP_OF_CLOUDFLARED_HOST>   # only if you use Cloudflare Tunnel
      - OVERWRITECLIURL=https://cloud.example.com  # only if you use Cloudflare Tunnel
      - PHP_UPLOAD_LIMIT=16G
      - PHP_MEMORY_LIMIT=1G
    volumes:
      - /mnt/tank/appdata/nextcloud/html:/var/www/html
      - /mnt/tank/data/nextcloud:/var/www/html/data
    ports:
      - 8080:80
    restart: unless-stopped

  cron:
    image: nextcloud:stable
    container_name: nextcloud_cron
    entrypoint: /cron.sh
    depends_on:
      - app
    volumes:
      - /mnt/tank/appdata/nextcloud/html:/var/www/html
      - /mnt/tank/data/nextcloud:/var/www/html/data
    restart: unless-stopped
```

> **Environment variables are only read on the very first start.** Changing `NEXTCLOUD_TRUSTED_DOMAINS`, passwords etc. later has no effect. Use the `occ` commands in [3.4](#34-useful-occ-commands) instead.

> **Updating:** Nextcloud can only be updated **one major version at a time** (e.g. 31 → 32 → 33, never 31 → 33). `nextcloud:stable` handles this for you if you update regularly. If you skipped a lot of updates, pin the next major version (`nextcloud:32`) first.

### 3.3 First setup

1. Wait 1–2 minutes after the first start (the installation runs in the background), then open `http://192.168.xxx.xxx:8080`.
2. Log in with the admin user from the compose file.
3. **Administration settings → Basic settings → Background jobs:** select **Cron** (the `nextcloud_cron` container does the work).
4. **Administration settings → Overview:** check the security & setup warnings. Most of them are fixed with the commands below.
5. **Users:** create accounts for your family.
6. Install the [desktop and mobile apps](https://nextcloud.com/install/) and log in with `http://192.168.xxx.xxx:8080` (or your domain).

### 3.4 Useful `occ` commands

`occ` is Nextcloud's command line tool. Run it inside the container as `www-data`:

```bash
# Fix common warnings after installation
docker exec -u www-data nextcloud php occ db:add-missing-indices
docker exec -u www-data nextcloud php occ maintenance:repair --include-expensive
docker exec -u www-data nextcloud php occ config:system:set default_phone_region --value="DE"
docker exec -u www-data nextcloud php occ config:system:set maintenance_window_start --type=integer --value=1

# Add a trusted domain later (index 2 = third entry)
docker exec -u www-data nextcloud php occ config:system:set trusted_domains 2 --value=cloud.example.com

# Rescan files you copied directly into the data folder
docker exec -u www-data nextcloud php occ files:scan --all
```

### 3.5 Nextcloud behind Cloudflare Tunnel

See [HSG - Remote Access](HSG%20-%20Remote%20Access.md) for setting up the tunnel. Point the public hostname to `http://192.168.xxx.xxx:8080`, then:

- `TRUSTED_PROXIES` = IP of the host running `cloudflared`, so Nextcloud trusts the forwarded HTTPS headers. Local access via `http://192.168.xxx.xxx:8080` keeps working.
- `cloud.example.com` must be in the trusted domains.
- **Upload limit:** Cloudflare's free plan limits a single request to **100 MB**. The web UI and apps upload large files in chunks, so this is usually not a problem. If the desktop client fails on large files, add `maxChunkSize=50000000` under `[General]` in its `nextcloud.cfg`.

---

## 4. Sonarr

Sonarr watches for new episodes of your TV shows, searches them via Jackett/Prowlarr, sends them to Deluge and moves them into your library with clean names.

### 4.1 Prepare folders

```bash
sudo mkdir -p /mnt/tank/appdata/sonarr
sudo chown -R 568:568 /mnt/tank/appdata/sonarr
```

### 4.2 Compose file

```yaml
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=568
      - PGID=568
      - UMASK=002
      - TZ=Europe/Berlin
    volumes:
      - /mnt/tank/appdata/sonarr:/config
      - /mnt/tank/data:/data
    ports:
      - 8989:8989
    restart: unless-stopped
```

> **Mount `/data` as a whole**, not `/data/torrents` and `/data/media/tv` separately. Only then can Sonarr create hardlinks, see [recommended folder structure](HSG%20-%20Server%20Software.md#4-recommended-folder-structure).

### 4.3 First setup

1. Open `http://192.168.xxx.xxx:8989`.
2. **Authentication:** choose `Forms (Login Page)`, **Authentication Required:** `Disabled for Local Addresses`, set username and password.
3. **Settings → Media Management:**
   - **Rename Episodes:** `Yes`
   - **Show Advanced** → **Use Hardlinks instead of Copy:** `Yes` (default)
   - **Root Folders → Add Root Folder:** `/data/media/tv`
4. **Settings → Download Clients → + → Deluge:**

| Field    | Value                              |
| -------- | ---------------------------------- |
| Name     | `Deluge`                           |
| Host     | `192.168.xxx.xxx`                  |
| Port     | `8112`                             |
| Password | Your Deluge web UI password        |
| Category | `tv-sonarr`                        |

   Click **Test** → **Save**. (The category requires the **Label** plugin in Deluge, see [8.3](#83-first-setup).)

5. **Indexers:** either add them via [Jackett](#63-connect-jackett-to-sonarr--radarr) or let [Prowlarr](#73-first-setup) sync them automatically.
6. **Series → Add New:** search a show, select root folder `/data/media/tv`, **Monitor:** `All Episodes` or `Future Episodes`, click **Add**.

**Settings → General → API Key:** copy it, you'll need it for Prowlarr.

---

## 5. Radarr

Radarr does the same as Sonarr, but for movies.

### 5.1 Prepare folders

```bash
sudo mkdir -p /mnt/tank/appdata/radarr
sudo chown -R 568:568 /mnt/tank/appdata/radarr
```

### 5.2 Compose file

```yaml
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=568
      - PGID=568
      - UMASK=002
      - TZ=Europe/Berlin
    volumes:
      - /mnt/tank/appdata/radarr:/config
      - /mnt/tank/data:/data
    ports:
      - 7878:7878
    restart: unless-stopped
```

### 5.3 First setup

Identical to [Sonarr](#43-first-setup) with these differences:

| Setting                   | Value                 |
| ------------------------- | --------------------- |
| URL                       | `http://192.168.xxx.xxx:7878` |
| Root folder               | `/data/media/movies`  |
| **Rename Movies**         | `Yes`                 |
| Deluge category           | `radarr`              |

**Movies → Add New:** search a movie, select root folder `/data/media/movies`, **Add Movie**.

---

## 6. Jackett

Jackett translates the search requests of Sonarr and Radarr into the formats of hundreds of torrent trackers (indexers).

> **Jackett or Prowlarr?** You only need **one** of them. [Prowlarr](#7-prowlarr) is newer and syncs indexers to Sonarr/Radarr automatically. Jackett requires adding each indexer manually in both apps but is very stable. For new setups, Prowlarr is usually the easier choice.

### 6.1 Prepare folders

```bash
sudo mkdir -p /mnt/tank/appdata/jackett
sudo chown -R 568:568 /mnt/tank/appdata/jackett
```

### 6.2 Compose file

```yaml
services:
  jackett:
    image: lscr.io/linuxserver/jackett:latest
    container_name: jackett
    environment:
      - PUID=568
      - PGID=568
      - TZ=Europe/Berlin
      - AUTO_UPDATE=true
    volumes:
      - /mnt/tank/appdata/jackett:/config
    ports:
      - 9117:9117
    restart: unless-stopped
```

### 6.3 Connect Jackett to Sonarr & Radarr

1. Open `http://192.168.xxx.xxx:9117`.
2. Scroll down, set an **Admin password** and click **Set Password**.
3. Click **+ Add indexer**, search your tracker(s), click **+** and enter your credentials if it's a private tracker.
4. For each indexer, click **Copy Torznab Feed**.
5. In Sonarr / Radarr: **Settings → Indexers → + → Torznab**:

| Field     | Value                                                                        |
| --------- | ---------------------------------------------------------------------------- |
| Name      | Name of the indexer                                                          |
| URL       | The copied Torznab feed, with `localhost` replaced by `192.168.xxx.xxx`      |
| API Key   | **API Key** shown at the top of the Jackett dashboard                        |
| Categories| Sonarr: TV categories, Radarr: movie categories (pre-filled in most cases)   |

6. **Test** → **Save**. Repeat for every indexer in both apps.

---

## 7. Prowlarr

*(optional, alternative to Jackett)*

### 7.1 Prepare folders

```bash
sudo mkdir -p /mnt/tank/appdata/prowlarr
sudo chown -R 568:568 /mnt/tank/appdata/prowlarr
```

### 7.2 Compose file

```yaml
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=568
      - PGID=568
      - TZ=Europe/Berlin
    volumes:
      - /mnt/tank/appdata/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped
```

### 7.3 First setup

1. Open `http://192.168.xxx.xxx:9696` and set up authentication (same as Sonarr).
2. **Indexers → Add Indexer:** search your tracker(s), enter credentials if needed, **Test** → **Save**.
3. **Settings → Apps → + → Sonarr:**

| Field            | Value                                  |
| ---------------- | -------------------------------------- |
| Prowlarr Server  | `http://192.168.xxx.xxx:9696`          |
| Sonarr Server    | `http://192.168.xxx.xxx:8989`          |
| API Key          | Sonarr → **Settings → General → API Key** |

4. Repeat with **Radarr** (`http://192.168.xxx.xxx:7878`, Radarr's API key).
5. Click **Sync App Indexers**. The indexers now appear automatically in Sonarr and Radarr under **Settings → Indexers**.

> Some trackers are protected by Cloudflare's anti-bot check. For those, add a **FlareSolverr** container and configure it in Prowlarr under **Settings → Indexers → + → FlareSolverr**.

---

## 8. Deluge

Deluge downloads the torrents Sonarr and Radarr send to it.

### 8.1 Prepare folders

```bash
sudo mkdir -p /mnt/tank/appdata/deluge
sudo chown -R 568:568 /mnt/tank/appdata/deluge
```

### 8.2 Compose file

```yaml
services:
  deluge:
    image: lscr.io/linuxserver/deluge:latest
    container_name: deluge
    environment:
      - PUID=568
      - PGID=568
      - UMASK=002
      - TZ=Europe/Berlin
    volumes:
      - /mnt/tank/appdata/deluge:/config
      - /mnt/tank/data/torrents:/data/torrents
    ports:
      - 8112:8112        # web UI
      - 6881:6881        # incoming torrent traffic
      - 6881:6881/udp
    restart: unless-stopped
```

> **Same path inside every container:** Deluge sees the downloads as `/data/torrents`, and Sonarr/Radarr see them as `/data/torrents` too. Because the paths match, no "Remote Path Mapping" is needed.

### 8.3 First setup

1. Open `http://192.168.xxx.xxx:8112` and log in with the default password `deluge`.
2. Change the password when asked (or **Preferences → Interface → Password**).
3. In the **Connection Manager**, select the local daemon and click **Connect**.
4. **Preferences → Downloads:**
   - **Download to:** `/data/torrents`
   - **Move completed to:** disabled (Sonarr/Radarr handle this)
5. **Preferences → Network:**
   - Untick **Use Random Port**, set the incoming port to `6881`.
6. **Preferences → Plugins:** enable **Label** (needed for the Sonarr/Radarr categories). Reload the page afterwards.
7. **Preferences → Queue:** set seeding limits if you like, e.g. **Share Ratio** `2.0` → **Pause torrent**.
8. Click **Apply**.

> **Port forwarding is optional.** Downloads work without it, but forwarding port `6881` (TCP/UDP) to your server on the router can improve speeds. This is the only port in this repo that may be forwarded; never forward the web UI port `8112`.

> Only download content you are legally allowed to download in your country.

---

## 9. Media stack in one compose file (optional)

Instead of deploying Sonarr, Radarr, Jackett/Prowlarr and Deluge as separate apps, you can deploy them as **one** app. Advantages: one place to manage and update, and the containers reach each other by **service name** instead of IP.

```yaml
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment: &env
      - PUID=568
      - PGID=568
      - UMASK=002
      - TZ=Europe/Berlin
    volumes:
      - /mnt/tank/appdata/sonarr:/config
      - /mnt/tank/data:/data
    ports:
      - 8989:8989
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment: *env
    volumes:
      - /mnt/tank/appdata/radarr:/config
      - /mnt/tank/data:/data
    ports:
      - 7878:7878
    restart: unless-stopped

  prowlarr:                       # or replace with the jackett service from section 6
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment: *env
    volumes:
      - /mnt/tank/appdata/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped

  deluge:
    image: lscr.io/linuxserver/deluge:latest
    container_name: deluge
    environment: *env
    volumes:
      - /mnt/tank/appdata/deluge:/config
      - /mnt/tank/data/torrents:/data/torrents
    ports:
      - 8112:8112
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
```

The setup inside the apps is the same as in sections 4–8, except that you use the **service names** as hosts:

| Where                               | Host value (instead of the IP) |
| ----------------------------------- | ------------------------------ |
| Sonarr / Radarr → Download client   | `deluge`                       |
| Prowlarr → Apps → Prowlarr Server   | `http://prowlarr:9696`         |
| Prowlarr → Apps → Sonarr Server     | `http://sonarr:8989`           |
| Prowlarr → Apps → Radarr Server     | `http://radarr:7878`           |
| Jackett Torznab feed (if used)      | `http://jackett:9117/...`      |

---

## 10. Verification & Troubleshooting

**Verify:**

- [ ] Every web UI loads: Immich `:2283`, Jellyfin `:8096`, Nextcloud `:8080`, Sonarr `:8989`, Radarr `:7878`, Jackett `:9117` / Prowlarr `:9696`, Deluge `:8112`.
- [ ] The Immich app uploads a test photo, and it appears in `/data/immich-lib`.
- [ ] Nextcloud **Administration settings → Overview** shows no critical errors.
- [ ] Sonarr/Radarr: **System → Status** shows no errors, **Test** succeeds for the download client and all indexers.
- [ ] A test download ends up in `/data/torrents`, gets imported into `/data/media/...`, and appears in Jellyfin.
- [ ] Hardlinks work (see command below).
- [ ] After a reboot, all containers start again by themselves.

**Check whether a file is hardlinked** (a link count of `2` or more means it's a hardlink):

```bash
ls -l /mnt/tank/data/media/movies/<Movie folder>/
#  -rw-rw-r-- 2 apps apps 4.2G ...   ← "2" = hardlinked, no extra space used
```

**Common problems:**

| Problem                                                       | Likely cause / fix                                                                                                          |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Immich: `database` container restarts / `chmod` errors        | Postgres folder is on a dataset with SMB/NFSv4 ACL (TrueNAS) or on `/mnt/user` (Unraid). Use the Generic `appdata` dataset / `/mnt/cache/...`. |
| Immich app can't connect                                      | Wrong URL. It must include `http://` and the port: `http://192.168.xxx.xxx:2283`.                                           |
| Jellyfin: `permission denied` on `/config`                    | Folder not owned by the `user:` of the container. Run the `chown` command from 2.1.                                        |
| Jellyfin doesn't show new files                               | Library scan hasn't run yet. **Dashboard → Libraries → Scan All Libraries**. Sonarr/Radarr can trigger scans automatically via **Settings → Connect → Emby / Jellyfin**. |
| Jellyfin transcoding fails with hardware acceleration enabled | `/dev/dri` not passed through, wrong group ID, or a codec is enabled that your GPU doesn't support.                         |
| Nextcloud: "Access through untrusted domain"                  | Add the address with the `occ config:system:set trusted_domains` command from [3.4](#34-useful-occ-commands).              |
| Nextcloud: "Your data directory is readable by other users"   | Data folder permissions. Set owner `33:33` and mode `770`.                                                                  |
| Sonarr/Radarr: "Unable to connect to Deluge"                  | Wrong password, Deluge web UI not connected to the daemon (Connection Manager), or wrong host/port.                        |
| Sonarr/Radarr: category error with Deluge                     | **Label** plugin not enabled in Deluge.                                                                                    |
| Sonarr/Radarr: "Import failed, path does not exist"           | Different paths in the containers. Deluge and Sonarr/Radarr must both see downloads under `/data/torrents`.                 |
| Files are copied instead of hardlinked                        | `torrents` and `media` are on different datasets/shares, or Sonarr/Radarr mount them separately instead of `/data`.         |
| Indexer test fails                                            | Tracker down, wrong API key, or Cloudflare protection (use FlareSolverr).                                                  |

---

## 11. Security Checklist

- [ ] All default passwords are changed (Deluge `deluge`, database passwords, Nextcloud admin).
- [ ] Sonarr, Radarr, Prowlarr and Jackett have authentication enabled.
- [ ] **Only** Immich, Jellyfin and Nextcloud are published via Cloudflare Tunnel, if at all ([HSG - Remote Access](HSG%20-%20Remote%20Access.md)).
- [ ] Sonarr, Radarr, Jackett/Prowlarr and Deluge are **never** published via the tunnel. Reach them via Tailscale only.
- [ ] Compose files with real passwords are not committed to a public Git repository.
- [ ] `appdata` and the compose files are included in your backups.
- [ ] You read the release notes before updating Immich and Nextcloud.
