# Homelab Setup Guide - Server Software

A guide to how to set up the software side of a homelab: the server operating system, Docker, and the containers running on top of it.
This section assumes your hardware is assembled and the drives are installed. Once you've [set up your services](HSG%20-%20Set%20up%20your%20services.md), go to [HSG - Remote Access](HSG%20-%20Remote%20Access.md) to set up remote access.

---

### Table of Contents

| No. | Topic                                                                  | Description                                                   |
| --- | ---------------------------------------------------------------------- | ------------------------------------------------------------- |
| 1   | [How this section is structured](#1-how-this-section-is-structured)    | Which guide to read in which order                            |
| 2   | [TrueNAS or Unraid?](#2-truenas-or-unraid)                             | Short comparison to help you pick an operating system         |
| 3   | [Guides](#3-guides)                                                    | Index of all Server Software guides                           |
| 3.1 | [Operating Systems](#31-operating-systems)                             | Installing and configuring TrueNAS / Unraid                   |
| 3.2 | [Docker](#32-docker)                                                   | Getting Docker running on TrueNAS / Unraid                    |
| 3.3 | [Docker Containers](#33-docker-containers)                             | Setting up the individual services                            |
| 4   | [Recommended folder structure](#4-recommended-folder-structure)        | One layout that works for every guide in this repo            |
| 5   | [Conventions used in these guides](#5-conventions-used-in-these-guides)| Placeholders, users, permissions, time zone                   |
| 6   | [Port reference](#6-port-reference)                                    | Default web UI ports of all covered services                  |

---

## 1. How this section is structured

The guides build on each other. Work through them in this order:

```
 1. Operating System          2. Docker                  3. Containers
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────────┐
│ TrueNAS  or      │  ──►  │ Apps / Docker    │  ──►  │ Immich, Jellyfin,    │
│ Unraid           │       │ on your OS       │       │ Nextcloud, *arr, ... │
└──────────────────┘       └──────────────────┘       └──────────────────────┘
                                                                 │
                                                                 ▼
                                                      4. Remote Access (optional)
```

1. Pick **one** operating system and follow its setup guide.
2. Follow the matching part of the Docker guide.
3. Set up the containers you want. Each container section is self-contained, but the media stack (Sonarr, Radarr, Jackett, download client, Jellyfin) works best when set up together.
4. Optional: make your services reachable from outside via [HSG - Remote Access](HSG%20-%20Remote%20Access.md).

---

## 2. TrueNAS or Unraid?

Both are excellent choices. The biggest difference is **how they store your data**.

|                           | TrueNAS Community Edition (formerly SCALE)                          | Unraid                                                                 |
| ------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| **Price**                 | Free & open source                                                  | Paid license (tiered by number of drives)                              |
| **Storage**               | ZFS pools (RAIDZ / mirrors)                                         | Unraid array with parity drive(s), optional ZFS pools                  |
| **Mixing drive sizes**    | Possible, but wastes capacity; matched drives recommended           | Yes, any size, drives can be added one at a time                       |
| **Performance**           | High, data is striped across drives                                 | Single-drive speed on the array, SSD cache pool helps a lot            |
| **Data integrity**        | Checksums + self-healing on every block (ZFS)                       | Parity protection; checksums only on ZFS/BTRFS pools                   |
| **Power usage**           | Drives of a pool usually spin together                              | Unused drives can spin down individually                               |
| **Docker**                | Apps catalog + custom apps via Docker Compose YAML                  | Community Applications (templates) + optional Compose plugin           |
| **Good for**              | Maximum data safety, matched drives, "set it and forget it"         | Growing a server over time with whatever drives you have, beginners    |

**Rule of thumb:** You buy all drives at once and data safety is your top priority → **TrueNAS**. You want to reuse old drives of different sizes and expand bit by bit → **Unraid**.

---

## 3. Guides

> **Status legend:** ✅ Available · 🚧 Work in progress · 📝 Planned

### 3.1 Operating Systems

| No. | Guide                                                       | Software used             | Description                                                              | Status |
| --- | ----------------------------------------------------------- | ------------------------- | ------------------------------------------------------------------------ | ------ |
| 1   | [TrueNAS Setup](HSG%20-%20TrueNAS%20Setup.md)               | TrueNAS Community Edition | Installation, pool & dataset creation, users, SMB shares, snapshots      | 📝     |
| 2   | [Unraid Setup](HSG%20-%20Unraid%20Setup.md)                 | Unraid                    | USB boot drive, array & parity, cache pool, shares, users, plugins       | 📝     |

### 3.2 Docker

| No. | Guide                                                                        | Software used                   | Description                                                        | Status |
| --- | ---------------------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------ | ------ |
| 1   | [Docker on TrueNAS](HSG%20-%20Docker%20Setup.md#1-docker-on-truenas)         | TrueNAS Apps, Docker Compose    | Apps pool, catalog apps vs. custom apps (YAML), host path volumes  | 📝     |
| 2   | [Docker on Unraid](HSG%20-%20Docker%20Setup.md#2-docker-on-unraid)           | Community Applications, Compose | Enabling Docker, appdata share, templates, Compose Manager plugin  | 📝     |
| 3   | [Docker basics](HSG%20-%20Docker%20Setup.md#3-docker-basics)                 | Docker, Docker Compose          | Images, volumes, networks, environment variables, updating         | 📝     |

### 3.3 Docker Containers

All container guides live in [HSG - Docker Containers](HSG%20-%20Docker%20Containers.md). Each section contains a ready-to-use Docker Compose file plus TrueNAS- and Unraid-specific notes.

| No. | Container                                                                | Category         | Additional containers           | Description                                                     | Status |
| --- | ------------------------------------------------------------------------ | ---------------- | ------------------------------- | --------------------------------------------------------------- | ------ |
| 1   | [Immich](HSG%20-%20Docker%20Containers.md#1-immich)                      | Photos           | PostgreSQL, Redis, ML           | Self-hosted Google Photos alternative with mobile backup        | 📝     |
| 2   | [Jellyfin](HSG%20-%20Docker%20Containers.md#2-jellyfin)                  | Media            | –                               | Media server for movies & TV shows, incl. hardware transcoding  | 📝     |
| 3   | [Nextcloud](HSG%20-%20Docker%20Containers.md#3-nextcloud)                | Cloud / Files    | Database, Redis                 | Self-hosted cloud storage, calendar & contacts                  | 📝     |
| 4   | [Sonarr](HSG%20-%20Docker%20Containers.md#4-sonarr)                      | Media automation | –                               | Automatically finds and organizes TV shows                      | 📝     |
| 5   | [Radarr](HSG%20-%20Docker%20Containers.md#5-radarr)                      | Media automation | –                               | Automatically finds and organizes movies                        | 📝     |
| 6   | [Jackett](HSG%20-%20Docker%20Containers.md#6-jackett)                    | Indexer          | –                               | Indexer proxy for Sonarr & Radarr                               | 📝     |
| 7   | [Prowlarr](HSG%20-%20Docker%20Containers.md#7-prowlarr) *(optional)*     | Indexer          | –                               | Modern alternative to Jackett, syncs indexers to the *arr apps  | 📝     |
| 8   | [Deluge](HSG%20-%20Docker%20Containers.md#8-deluge)                      | Download client  | –                               | Download client used by Sonarr & Radarr                         | 📝     |

**How the media stack works together:**

```
            ┌─────────────┐   search    ┌───────────────────┐
  You ────► │ Sonarr /    │ ──────────► │ Jackett /         │ ──► Indexers
  add show  │ Radarr      │ ◄────────── │ Prowlarr          │
            └──────┬──────┘   results   └───────────────────┘
                   │ sends download
                   ▼
            ┌─────────────┐  downloads  ┌───────────────────┐
            │   Deluge    │ ──────────► │ /data/torrents    │
            └─────────────┘             └─────────┬─────────┘
                                                  │ Sonarr / Radarr import
                                                  │ (hardlink, instant)
                                                  ▼
            ┌─────────────┐    reads    ┌───────────────────┐
  You ◄──── │ Jellyfin    │ ◄────────── │ /data/media       │
  watch     └─────────────┘             └───────────────────┘
```

---

## 4. Recommended folder structure

All guides in this repo use the same layout. Stick to it and you avoid 90% of the usual permission and path problems.

```
<pool / share root>
├── appdata                 # container configs & databases (ideally on SSD)
│   ├── immich
│   ├── jellyfin
│   ├── nextcloud
│   ├── sonarr
│   └── ...
└── data                    # your actual files
    ├── torrents            # download client writes here
    ├── media               # Jellyfin reads from here
    │   ├── movies
    │   └── tv
    ├── immich-lib              # Immich library
    └── nextcloud           # Nextcloud user files
```

| OS      | `appdata` path                    | `data` path                   |
| ------- | --------------------------------- | ----------------------------- |
| TrueNAS | `/mnt/tank/appdata/<app>`         | `/mnt/tank/data`              |
| Unraid  | `/mnt/user/appdata/<app>`         | `/mnt/user/data`              |

> **Why one single `data` folder?** Sonarr and Radarr can only create **hardlinks** (instant moves, no double disk usage, seeding keeps working) if the download folder and the media folder are on the **same dataset / share**. That's why every media-related container mounts `/data` as a whole instead of `/downloads` and `/movies` separately.

> **Databases belong on SSDs.** Keep `appdata` on your SSD pool (TrueNAS) or cache pool (Unraid). Databases like the ones of Immich and Nextcloud get very slow on spinning drives and can even corrupt on Unraid's network shares (`/mnt/user/...` → use `/mnt/cache/...` or the pool path for databases).

---

## 5. Conventions used in these guides

> **Placeholder values used in these guides** – replace them with your own:

| Placeholder        | Meaning                                  | Example            |
| ------------------ | ---------------------------------------- | ------------------ |
| `tank`             | Name of your TrueNAS pool                | `tank`             |
| `192.168.xxx.xxx`  | IP address of your server                | `192.168.1.10`     |
| `{port}`           | Web UI port of the service               | `8096`             |
| `<app>`            | Name of the container                    | `jellyfin`         |
| `PUID` / `PGID`    | User / group ID the container runs as    | see below          |
| `TZ`               | Your time zone                           | `Europe/Berlin`    |

**User & group IDs (`PUID` / `PGID`):** Containers should never run as root if they don't have to. Use the OS-specific default user unless you know what you're doing:

| OS      | User            | `PUID` | `PGID` |
| ------- | --------------- | ------ | ------ |
| TrueNAS | `apps`          | `568`  | `568`  |
| Unraid  | `nobody:users`  | `99`   | `100`  |

**Compose first:** Every container guide provides a Docker Compose file. It works on both systems (TrueNAS custom apps / Unraid Compose Manager) and makes your setup easy to back up and rebuild. Where a good catalog app or template exists, the guide mentions it as an alternative.

**Secrets:** Passwords and API keys shown in the guides are placeholders like `<CHANGE_ME>`. Never commit real credentials to Git; use a `.env` file instead.

---

## 6. Port reference

Default web UI ports of all services covered in this section. Two containers can't use the same host port, so change the **left** side of the port mapping (`8081:8080`) if you run into a conflict.

| Service     | Default port | URL                              |
| ----------- | ------------ | -------------------------------- |
| Immich      | `2283`       | `http://192.168.xxx.xxx:2283`    |
| Jellyfin    | `8096`       | `http://192.168.xxx.xxx:8096`    |
| Nextcloud   | depends on image, see guide | –                 |
| Radarr      | `7878`       | `http://192.168.xxx.xxx:7878`    |
| Sonarr      | `8989`       | `http://192.168.xxx.xxx:8989`    |
| Prowlarr    | `9696`       | `http://192.168.xxx.xxx:9696`    |
| Jackett     | `9117`       | `http://192.168.xxx.xxx:9117`    |
| Deluge      | `8112`       | `http://192.168.xxx.xxx:8112`    |

> **Note:** The TrueNAS web UI itself runs on ports `80`/`443`. Don't map any container to these ports on TrueNAS.

---

**Next step:** Choose your operating system → [TrueNAS Setup](HSG%20-%20TrueNAS%20Setup.md) or [Unraid Setup](HSG%20-%20Unraid%20Setup.md).
