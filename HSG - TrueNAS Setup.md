# Homelab Setup Guide - TrueNAS Setup

A guide to how to install and configure TrueNAS Community Edition (formerly TrueNAS SCALE) as the base of your homelab.
This guide was written for **TrueNAS 25.10 "Goldeye" and newer**. Menu names can differ slightly between versions, but the steps stay the same.
Back to the overview: [HSG - Server Software](HSG%20-%20Server%20Software.md)

---

### Table of Contents

| No. | Topic                                                         | Software used | Description                                                 |
| --- | ------------------------------------------------------------- | ------------- | ----------------------------------------------------------- |
| 1   | [Prerequisites](#1-prerequisites)                             | –             | Hardware and things you need before starting                |
| 2   | [Installation](#2-installation)                               | TrueNAS       | Flashing the installer and installing TrueNAS               |
| 3   | [First login & basic settings](#3-first-login--basic-settings)| TrueNAS       | Network, time zone, updates, notifications                  |
| 4   | [Creating a pool](#4-creating-a-pool)                         | ZFS           | Choosing the right layout and creating your storage pool    |
| 5   | [Creating datasets](#5-creating-datasets)                     | ZFS           | `appdata` and `data` datasets with the right permissions    |
| 6   | [Users & SMB shares](#6-users--smb-shares)                    | SMB           | Accessing your files from Windows, macOS and Linux          |
| 7   | [Data protection](#7-data-protection)                         | ZFS           | Snapshots, scrubs and SMART tests                           |
| 8   | [Verification & Troubleshooting](#8-verification--troubleshooting) | –        | Check that everything works                                 |

---

## 1. Prerequisites

| Component      | Minimum                                   | Recommended                                                            |
| -------------- | ----------------------------------------- | ---------------------------------------------------------------------- |
| CPU            | 64-bit x86 (Intel / AMD)                  | Intel CPU with iGPU (Quick Sync) for Jellyfin / Immich transcoding     |
| RAM            | 8 GB                                      | 16 GB or more (ZFS uses free RAM as cache, more RAM = faster)          |
| Boot drive     | 16 GB SSD                                 | Small dedicated SATA / NVMe SSD (**not** a USB stick)                  |
| Data drives    | 1 drive                                   | 2+ identical HDDs (see [pool layouts](#4-creating-a-pool))             |
| Apps drive     | –                                         | 1–2 SSDs for a separate pool holding `appdata` and databases           |
| Network        | 1 GbE                                     | 2.5 GbE or faster                                                      |

You also need:

- A USB stick (≥ 8 GB) for the installer.
- A PC to download the ISO and flash the USB stick.
- A monitor and keyboard for the installation (only once).
- Access to your router to give the server a fixed IP address.

> ⚠️ **The boot drive is used exclusively for TrueNAS.** You can't store data on it. Everything on the drives you select during installation will be **erased**.

> **ECC RAM** is nice to have but not required for a homelab.

---

## 2. Installation

1. Download the latest stable ISO from the [TrueNAS download page](https://www.truenas.com/download-truenas-community-edition/).
2. Flash the ISO to the USB stick with [balenaEtcher](https://etcher.balena.io/) or [Rufus](https://rufus.ie/) (use **DD mode** if Rufus asks).
3. Plug the USB stick into the server and boot from it (boot menu is usually `F11`, `F12` or `DEL` depending on your mainboard).
4. In the installer choose **Install/Upgrade**.
5. Select the **boot drive** only (the small SSD). Do **not** select your data drives.
6. Confirm that the drive will be erased.
7. Choose **Administrative user (truenas_admin)** and set a strong password. Save it in your password manager.
8. Choose **UEFI** boot if asked (default on modern hardware).
9. Wait for the installation to finish, remove the USB stick and reboot.

After booting, the console shows the address of the web interface:

```
The web user interface is at:
http://192.168.xxx.xxx
```

---

## 3. First login & basic settings

Open the address shown on the console in your browser and log in with `truenas_admin` and your password.

### 3.1 Fixed IP address

Your server needs an IP address that never changes, otherwise all your bookmarks, SMB shares and Cloudflare Tunnel routes break.

**Recommended:** Create a **DHCP reservation** for the server in your router (sometimes called "static lease" or "always assign this IP address"). This way TrueNAS keeps using DHCP and you manage all IPs in one place.

Alternative: Set a static IP in TrueNAS under **Network → Interfaces → Edit**. Make sure the IP is outside the DHCP range of your router and set the gateway and DNS servers under **Network → Global Configuration**.

### 3.2 General settings

| Setting             | Where                                                            | Recommended value                         |
| ------------------- | ---------------------------------------------------------------- | ----------------------------------------- |
| Time zone           | **System → General Settings → Localization**                     | Your time zone, e.g. `Europe/Berlin`      |
| Hostname            | **Network → Global Configuration**                               | e.g. `truenas` or `homelab`               |
| Email notifications | **System → General Settings → Email** + **Alerts → Settings**    | Your email, so you get warned about failing drives |
| Updates             | **System → Update**                                              | Stay on the **stable** train              |

> **Test your email alerts.** A failing drive you don't know about is the most common reason for data loss on a NAS.

---

## 4. Creating a pool

A **pool** is the storage made from one or more drives. Inside a pool you create **datasets** (similar to folders, but with their own settings, permissions and snapshots).

### 4.1 Choosing a layout

| Layout     | Drives | Usable capacity     | Drives that can fail | Good for                                       |
| ---------- | ------ | ------------------- | -------------------- | ---------------------------------------------- |
| Stripe     | 1+     | 100 %               | **0**                | Nothing important. Don't use it for your data. |
| Mirror     | 2      | 50 %                | 1                    | Small setups, apps pools on SSDs               |
| RAIDZ1     | 3–5    | (n − 1) drives      | 1                    | Small drives (≤ 8 TB), budget setups           |
| RAIDZ2     | 4–12   | (n − 2) drives      | 2                    | **Recommended** for large drives and 5+ disks  |

> **Rule of thumb:** 2 drives → **Mirror**. 3–4 drives → **RAIDZ1** (or RAIDZ2 with 4 large drives). 5+ drives → **RAIDZ2**.

> **RAID is not a backup!** It protects against a dead drive, not against accidental deletion, ransomware or a fire. See [Data protection](#7-data-protection).

### 4.2 Create the data pool

1. Go to **Storage → Create Pool**.
2. Name: `tank` (or any name you like; this guide uses `tank` as placeholder).
3. Click through the wizard:
   - **Data:** select the layout (e.g. `RAIDZ2`), the drive size and the number of drives.
   - **Log / Spare / Cache / Metadata / Dedup:** leave empty. These are for special use cases and not needed in a homelab.
4. Review and click **Create Pool**. Confirm that the drives will be erased.

### 4.3 Optional: SSD pool for apps

If you have SSDs, create a second pool (e.g. `fast`) as a **Mirror** of two SSDs (or a single SSD stripe if you have regular backups of it). Put `appdata` on this pool. Databases (Immich, Nextcloud) and app configs run a lot faster on SSDs.

> If you create an SSD pool, replace `/mnt/tank/appdata` with `/mnt/fast/appdata` in all following guides.

---

## 5. Creating datasets

This guide follows the [recommended folder structure](HSG%20-%20Server%20Software.md#4-recommended-folder-structure):

```
tank (pool)
├── appdata        dataset, preset "Generic", owner apps:apps
└── data           dataset, preset "SMB"
    ├── torrents   normal folder (!)
    ├── media      normal folder (!)
    │   ├── movies
    │   └── tv
    ├── immich-lib normal folder
    └── nextcloud  child dataset, preset "Generic", owner www-data:www-data
```

### 5.1 `appdata` dataset

1. Go to **Datasets**, select the pool `tank` (or `fast`) and click **Add Dataset**.
2. Name: `appdata`, **Dataset Preset:** `Generic`.
3. Save.
4. Select the new dataset → **Permissions → Edit**:
   - **User:** `apps`, **Group:** `apps`
   - Tick **Apply User**, **Apply Group** and **Apply permissions recursively**.
   - Permissions: owner `Read | Write | Execute`, group `Read | Write | Execute`, other `none`.
5. Save.

> **Why "Generic" and not "SMB" for appdata?** Databases like PostgreSQL need to change file permissions (`chmod`) on their data folder. On datasets with an SMB/NFSv4 ACL this fails and the database won't start. `appdata` is never shared via SMB anyway.

### 5.2 `data` dataset

1. **Add Dataset** on the pool `tank`, name `data`, **Dataset Preset:** `SMB`.
2. Leave **Create SMB Share** ticked, share name `data`.
3. Save.
4. Select the dataset → **Permissions → Edit** (ACL editor) and add these entries:
   - **User** `apps` → Permissions `Modify`
   - **User** `<your user>` → Permissions `Modify` (after you created your user in [step 6](#6-users--smb-shares))
5. Tick **Apply permissions recursively** and save.

### 5.3 Folders inside `data`

Open **System → Shell** and create the folders:

```bash
mkdir -p /mnt/tank/data/{torrents,media/movies,media/tv,immich-lib}
```

> ⚠️ **`torrents` and `media` must be normal folders, not datasets.** Hardlinks don't work across datasets. If they are separate datasets, Sonarr and Radarr have to copy every file instead of creating an instant hardlink, which doubles the used space.

### 5.4 `nextcloud` dataset (only if you use Nextcloud)

The official Nextcloud image runs as the user `www-data` (UID `33`), so its data folder needs different permissions:

1. Select `data` → **Add Dataset**, name `nextcloud`, **Dataset Preset:** `Generic`.
2. **Permissions → Edit:** User `www-data`, Group `www-data`, tick **Apply User / Apply Group**, permissions `770`.

---

## 6. Users & SMB shares

### 6.1 Create your user

Never use `truenas_admin` for everyday file access.

1. Go to **Credentials → Users → Add**.
2. Set **Username**, **Full Name** and **Password**.
3. Enable **SMB Access** (named "SMB User" in some versions).
4. Save.
5. Go back to the `data` dataset permissions and add the user as described in [5.2](#52-data-dataset).

### 6.2 SMB service & connecting

1. Go to **System → Services** and make sure **SMB** is **running** and set to **Start Automatically**.
2. Connect from your client:

| OS      | How to connect                                                       |
| ------- | -------------------------------------------------------------------- |
| Windows | Explorer → address bar → `\\192.168.xxx.xxx\data` (or map a network drive) |
| macOS   | Finder → **Go → Connect to Server** → `smb://192.168.xxx.xxx/data`   |
| Linux   | File manager → `smb://192.168.xxx.xxx/data`                          |

Log in with the user you just created.

---

## 7. Data protection

### 7.1 Snapshots

Snapshots are read-only copies of a dataset at a point in time. They cost almost no space and protect you against accidental deletion and ransomware.

1. Go to **Data Protection → Periodic Snapshot Tasks → Add**.
2. Recommended tasks:

| Dataset        | Schedule | Keep for |
| -------------- | -------- | -------- |
| `tank/data`    | Daily    | 2 weeks  |
| `tank/appdata` | Daily    | 1 week   |

3. Tick **Recursive** so child datasets (e.g. `nextcloud`) are included.

Restoring a single file on Windows: right-click the folder in the SMB share → **Properties → Previous Versions**.

### 7.2 Scrubs & SMART tests

- **Scrub tasks** (**Data Protection → Scrub Tasks**) read all data and repair silent corruption. TrueNAS creates one per pool automatically; make sure it runs at least **monthly**.
- **SMART tests** (**Data Protection → S.M.A.R.T. Tests** in most versions) detect dying drives early. Recommended: a **short** test weekly, a **long** test monthly.

### 7.3 Backups (3-2-1 rule)

Snapshots live on the same drives as your data. A real backup needs another copy on another device, ideally offsite:

- **3** copies of your data,
- on **2** different media,
- **1** of them offsite.

Options built into TrueNAS: **Replication Tasks** to a second TrueNAS, or **Cloud Sync Tasks** to Backblaze B2, Storj, S3 etc. under **Data Protection**.

---

## 8. Verification & Troubleshooting

**Verify:**

- [ ] The web UI is reachable at the same IP after a reboot.
- [ ] The pool shows as **Online** and healthy under **Storage**.
- [ ] You can open `\\192.168.xxx.xxx\data` and create/delete a test file with your user.
- [ ] The folders `torrents`, `media/movies`, `media/tv`, `immich-lib` exist.
- [ ] A test email alert arrives.
- [ ] Snapshot, scrub and SMART tasks are scheduled.

**Common problems:**

| Problem                                          | Likely cause / fix                                                                                              |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| Web UI not reachable                             | Server got a new IP from DHCP. Check the console or your router and set a DHCP reservation.                     |
| "Access denied" on the SMB share                 | User has no ACL entry on `data`, or **SMB Access** isn't enabled for the user.                                  |
| Windows keeps asking for a password              | Windows cached old credentials. Remove them in **Credential Manager** and reconnect.                            |
| App can't write to a folder                      | `apps` user is missing in the ACL of `data`, or `appdata` isn't owned by `apps:apps`.                           |
| PostgreSQL container fails with `chmod` errors   | The database folder is on a dataset with SMB preset / NFSv4 ACL. Put it on the `Generic` `appdata` dataset.     |
| Drive shown with errors / pool **Degraded**      | Replace the drive: **Storage → Manage Devices → select disk → Replace**.                                        |

---

**Next step:** Set up Docker → [Docker on TrueNAS](HSG%20-%20Docker%20Setup.md#1-docker-on-truenas)
