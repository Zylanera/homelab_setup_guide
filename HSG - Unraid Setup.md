# Homelab Setup Guide - Unraid Setup

A guide to how to install and configure Unraid as the base of your homelab.
This guide was written for **Unraid 7.x**. Menu names can differ slightly between versions, but the steps stay the same.
Back to the overview: [HSG - Server Software](HSG%20-%20Server%20Software.md)

---

### Table of Contents

| No. | Topic                                                               | Software used          | Description                                               |
| --- | ------------------------------------------------------------------- | ---------------------- | --------------------------------------------------------- |
| 1   | [Prerequisites](#1-prerequisites)                                   | –                      | Hardware and things you need before starting              |
| 2   | [Installation](#2-installation)                                     | Unraid USB Creator     | Creating the boot device and first boot                   |
| 3   | [First login & basic settings](#3-first-login--basic-settings)      | Unraid                 | Onboarding, license, network, time zone, notifications    |
| 4   | [Array & pools](#4-array--pools)                                    | Unraid array, ZFS/BTRFS| Assigning parity, data drives and a cache pool            |
| 5   | [Shares](#5-shares)                                                 | Unraid shares          | `appdata` and `data` shares with the right settings       |
| 6   | [Users & SMB access](#6-users--smb-access)                          | SMB                    | Accessing your files from Windows, macOS and Linux        |
| 7   | [Recommended plugins](#7-recommended-plugins)                       | Community Applications | Plugins every Unraid server should have                   |
| 8   | [Data protection](#8-data-protection)                               | –                      | Parity checks, backups                                    |
| 9   | [Verification & Troubleshooting](#9-verification--troubleshooting)  | –                      | Check that everything works                               |

---

## 1. Prerequisites

| Component      | Minimum                                   | Recommended                                                            |
| -------------- | ----------------------------------------- | ---------------------------------------------------------------------- |
| CPU            | 64-bit x86 (Intel / AMD)                  | Intel CPU with iGPU (Quick Sync) for Jellyfin / Immich transcoding     |
| RAM            | 4 GB                                      | 16 GB or more                                                          |
| Boot device    | USB stick, 4–32 GB, **with unique GUID**  | Quality USB 2.0/3.0 stick from a known brand, **or** internal boot (see below) |
| Data drives    | 1 drive                                   | Any mix of HDDs. The **largest** one (or two) become parity.           |
| Cache drive    | –                                         | 1–2 SSDs (NVMe or SATA) for `appdata`, Docker and fresh downloads      |
| Network        | 1 GbE                                     | 2.5 GbE or faster                                                      |

You also need:

- An Unraid **license**. There's a free trial (30 days) to test everything. Paid tiers differ in the number of storage devices you can attach.
- A PC to create the boot device.
- Access to your router to give the server a fixed IP address.

> **Internal boot:** Since Unraid 7.3 you can boot from an internal drive instead of a USB stick. This requires a mainboard with **TPM 2.0**, which the license is then bound to. The onboarding wizard offers this option. USB boot still works and is used in this guide.

> ⚠️ Unraid will **erase** all drives you add to the array or a pool.

---

## 2. Installation

1. Download the **Unraid USB Flash Creator** from the [Unraid download page](https://unraid.net/download) and run it.
2. Plug in the USB stick, select the latest **stable** Unraid version and your USB stick.
3. Optional in **Customize**: set a server name and a static IP (you can also do this later).
4. Click **Write**. This erases the stick.
5. Plug the stick into the server, preferably into a **USB 2.0** port or an internal USB header (more reliable than USB 3.0 ports for this).
6. In the BIOS/UEFI set the USB stick as the **first boot device** and boot.
7. Unraid starts. After a short time the console shows the IP address of the web interface.

---

## 3. First login & basic settings

Open `http://<server-ip>` (or `http://tower.local` if you didn't change the name) in your browser.

### 3.1 Onboarding

1. Set the password for the `root` user. Save it in your password manager.
2. Follow the **onboarding wizard**: language, time zone, theme, boot option and license (start the trial or enter your key).
3. The wizard also offers to install recommended plugins like **Community Applications** and **Fix Common Problems**. Accept.

### 3.2 Fixed IP address

**Recommended:** Create a **DHCP reservation** for the server in your router. This way Unraid keeps using DHCP and you manage all IPs in one place.

Alternative: **Settings → Network Settings** → set **IPv4 address assignment** to `Static` and enter IP, gateway and DNS.

### 3.3 General settings

| Setting             | Where                                                 | Recommended value                                 |
| ------------------- | ----------------------------------------------------- | ------------------------------------------------- |
| Time zone           | **Settings → Date and Time**                          | Your time zone, e.g. `Europe/Berlin`              |
| Server name         | **Settings → Identification**                         | e.g. `unraid` or `homelab`                        |
| Notifications       | **Settings → Notification Settings**                  | Enable email, Discord, Pushover, ... and test it  |
| Disk spin down      | **Settings → Disk Settings → Default spin down delay**| `30 minutes` or `1 hour`                          |
| Hardlink support    | **Settings → Global Share Settings → Tunable (support Hard Links)** | `Yes` (required for Sonarr/Radarr) |

> **Test your notifications.** A failing drive you don't know about is the most common reason for data loss on a NAS.

---

## 4. Array & pools

Unraid has two kinds of storage:

- **Array:** your HDDs. Each drive has its own file system, protected by one or two **parity** drives. Drives can have different sizes and spin down individually.
- **Pools:** usually SSDs (e.g. the `cache` pool). Fast, used for apps, Docker and as a landing zone for new files.

```
                ┌───────────── Unraid ─────────────┐
 new files ───► │  cache pool (SSD)                │
                │   appdata, system, data (new)    │
                │           │ Mover (nightly)      │
                │           ▼                      │
                │  array (HDD)                     │
                │   parity │ disk1 │ disk2 │ ...   │
                └──────────────────────────────────┘
```

### 4.1 Assign the array

1. Go to **Main**.
2. **Parity:** select your **largest** drive. No data drive may be larger than the parity drive.
   - With 5+ data drives, consider **Parity 2** (protects against two failed drives).
3. **Disk 1, Disk 2, ...:** select the remaining HDDs.
4. Leave the file system at the default (**XFS**) for array drives.

### 4.2 Create the cache pool

1. Below the array click **Add Pool**, name it `cache`, number of slots = number of SSDs.
2. Assign your SSD(s).
3. Click the pool name and set the file system:
   - 1 SSD → `xfs` or `zfs`
   - 2 SSDs → `zfs` (mirror) or `btrfs` (raid1). Two SSDs protect your appdata against a dead SSD.

### 4.3 Start the array

1. Scroll down, click **Start**.
2. New drives show as **Unmountable**. Tick **Yes, I want to do this** next to **Format** and click **Format**.
3. The first **parity sync** starts. This can take many hours. You can already use the server, it's just slower during the sync.

> **RAID is not a backup!** Parity protects against a dead drive, not against accidental deletion, ransomware or a fire. See [Data protection](#8-data-protection).

---

## 5. Shares

This guide follows the [recommended folder structure](HSG%20-%20Server%20Software.md#4-recommended-folder-structure). Unraid creates `appdata`, `system`, `domains` and `isos` automatically once Docker/VMs are enabled.

### 5.1 `appdata` share

Go to **Shares**, click `appdata` (create it if it doesn't exist yet):

| Setting             | Value                  |
| ------------------- | ---------------------- |
| Primary storage     | `cache`                |
| Secondary storage   | `None`                 |
| Export (SMB)        | `No`                   |

`appdata` should live **only** on the SSD. Moving databases to the HDD array makes them extremely slow.

### 5.2 `data` share

**Shares → Add Share:**

| Setting                  | Value                    | Why                                                     |
| ------------------------ | ------------------------ | ------------------------------------------------------- |
| Name                     | `data`                   |                                                         |
| Primary storage          | `cache`                  | New downloads/uploads land on the fast SSD first        |
| Secondary storage        | `Array`                  | The Mover moves them to the HDDs later                  |
| Mover action             | `Cache → Array`          |                                                         |
| Allocation method        | `High-water`             | Default, fills drives evenly                            |
| Split level              | `Automatically split any directory as required` | Simplest option              |

Then **SMB Security Settings** of the share: **Export** `Yes`, **Security** `Private`, give your user **Read/Write** (after [step 6](#6-users--smb-access)).

> **Only one `data` share!** Hardlinks only work inside the **same share**. If downloads and media are separate shares, Sonarr and Radarr have to copy every file, which doubles the used space.

> If your SSD is small, set **Primary storage** of `data` to `Array` and **Secondary storage** to `None` instead. Downloads then go directly to the HDDs.

### 5.3 Folders inside `data`

Open the terminal (**>_** icon top right) and create the folders:

```bash
mkdir -p /mnt/user/data/{torrents,media/movies,media/tv,immich-lib,nextcloud}
chown -R nobody:users /mnt/user/data
chmod -R 775 /mnt/user/data
```

### 5.4 Mover schedule

**Settings → Scheduler → Mover Settings:** run the Mover **daily at night** (e.g. 03:00).

> **`/mnt/user/...` vs. `/mnt/cache/...`:** `/mnt/user/` is a combined view of the cache and all array drives. For **databases** (Immich, Nextcloud) always use the direct pool path `/mnt/cache/appdata/...`. This bypasses the user share layer, which is faster and avoids database corruption.

---

## 6. Users & SMB access

### 6.1 Create your user

Never use `root` for file access.

1. **Users → Add User**.
2. Set **User name** and **Password**, click **Add**.
3. Go to **Shares → data → SMB Security Settings** and set your user to **Read/Write**.

### 6.2 Connecting

Make sure SMB is enabled under **Settings → SMB** (**Enable SMB:** `Yes`).

| OS      | How to connect                                                       |
| ------- | -------------------------------------------------------------------- |
| Windows | Explorer → address bar → `\\192.168.xxx.xxx\data` (or map a network drive) |
| macOS   | Finder → **Go → Connect to Server** → `smb://192.168.xxx.xxx/data`   |
| Linux   | File manager → `smb://192.168.xxx.xxx/data`                          |

---

## 7. Recommended plugins

All plugins are installed via the **Apps** tab (Community Applications).

| Plugin                             | Why                                                                          |
| ---------------------------------- | ---------------------------------------------------------------------------- |
| **Community Applications**         | The app store for plugins and Docker templates (the **Apps** tab)            |
| **Fix Common Problems**            | Scans your config for common mistakes and warns you                          |
| **Appdata Backup**                 | Stops containers, backs up `appdata` and the boot device on a schedule       |
| **Unassigned Devices**             | Mount drives that aren't part of the array/pools (e.g. USB backup drives)    |
| **Dynamix System Temperature**     | Shows CPU and mainboard temperatures on the dashboard                        |
| **Intel GPU TOP**                  | Loads the Intel iGPU driver for hardware transcoding (Intel CPUs)            |
| **Nvidia Driver**                  | Only if you use an Nvidia GPU for transcoding                                |
| **Docker Compose Manager**         | Run Docker Compose stacks (used in the [Docker guide](HSG%20-%20Docker%20Setup.md#2-docker-on-unraid)) |

---

## 8. Data protection

### 8.1 Parity checks

**Settings → Scheduler → Parity Check:** schedule a check **monthly** (it reads every drive and detects problems early). Leave **Write corrections to parity** enabled for scheduled checks.

### 8.2 Boot device backup

Your license and entire configuration live on the boot device. Back it up:

- Via **Main → Boot Device → Boot Device Backup** (named **Flash → Flash Backup** in older versions), or
- automatically via **Unraid Connect** or the **Appdata Backup** plugin.

### 8.3 Backups (3-2-1 rule)

Parity is not a backup. Keep:

- **3** copies of your data,
- on **2** different media,
- **1** of them offsite.

Popular options: **Appdata Backup** for configs, and a Docker container like **Duplicacy**, **Kopia** or **rclone** to back up important folders (photos, documents) to Backblaze B2, another NAS or a USB drive.

---

## 9. Verification & Troubleshooting

**Verify:**

- [ ] The web UI is reachable at the same IP after a reboot.
- [ ] Array shows all drives green, parity is **valid**.
- [ ] `appdata` shows **Primary storage: cache** and no files on the array.
- [ ] **Tunable (support Hard Links)** is set to `Yes`.
- [ ] You can open `\\192.168.xxx.xxx\data` and create/delete a test file.
- [ ] A test notification arrives.
- [ ] **Fix Common Problems** shows no errors.

**Common problems:**

| Problem                                          | Likely cause / fix                                                                                              |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| Server doesn't boot from the USB stick           | Wrong boot order, or try another USB port (USB 2.0). Some mainboards need **CSM/Legacy** boot for USB sticks.   |
| "Invalid GUID" / license error                   | The USB stick has no unique GUID. Use another stick from a known brand.                                         |
| New drive shows **Unmountable**                  | Normal for new drives. Format it (see [4.3](#43-start-the-array)).                                              |
| Files stay on the cache forever                  | Mover action is set wrong, or the Mover isn't scheduled. Check the share settings.                              |
| `appdata` files end up on the array              | Share settings of `appdata` were changed. Set **Secondary storage** to `None`, then set Mover action **Array → Cache** once and run the Mover. |
| Sonarr/Radarr copy files instead of hardlinking  | Downloads and media are in different shares, or **Tunable (support Hard Links)** is `No`.                       |
| Red X on a drive                                 | The drive was disabled after write errors. Check cables first, then replace the drive and let Unraid rebuild.   |

---

**Next step:** Set up Docker → [Docker on Unraid](HSG%20-%20Docker%20Setup.md#2-docker-on-unraid)
