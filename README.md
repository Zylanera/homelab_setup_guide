# Homelab Setup Guide
A Guide to how to setup a homelab and access it from remote. <br>
This documentation assumes that services are already running. A separate setup guide covering this specifically will be available [here](URL) soon.

---

### Table of Contents
| No. | Topic | Software used | Description |
| -------- | -------- | ------- | ------- |
| 1 | [Overview](#1-overview) | – | Which access method to use for what |
| 2 | [Prerequisites](#2-prerequisites) | – | What you need before starting |
| 3 | [Public access via Cloudflare Tunnel](#3-public-access-via-cloudflare-tunnel) | cloudflared, Cloudflare Zero Trust | Expose selected services to the internet without opening ports |
| 3.1 | [Securing public services](#31-securing-public-services-with-cloudflare-access) | Cloudflare Access | Put a login in front of exposed services |
| 4 | [Private access via VPN](#4-private-access-via-tailscale-vpn) | Tailscale | Reach your whole homelab from your own devices |
| 4.4 | [Exit Node](#44-exit-node) | Tailscale | Reach devices in your home network that can't run Tailscale themselves |
| 5 | [Verification & Troubleshooting](#5-verification--troubleshooting) | – | Check that everything works |
| 6 | [Security Checklist](#6-security-checklist) | – | Things to double-check before you're done |

---

## 1. Overview
 
There are two fundamentally different ways to reach your homelab from outside your home network. Both work **without port forwarding** on your router and **without a static public IP**, since both tools build an outbound connection from inside your network.
 
| | Cloudflare Tunnel | Tailscale VPN |
| -------- | ------- | ------- |
| **Who can reach it** | Anyone on the internet (optionally restricted via Cloudflare Access) | Only your own devices that are logged into your Tailnet |
| **Client software needed** | No, just a browser | Yes, Tailscale application on every client |
| **Protocols** | Mainly HTTP/HTTPS | Everything (SSH, SMB, RDP, databases, …) |
| **Needs a domain** | Yes, managed via Cloudflare DNS (gets done automaticly when setting up a tunnel) | No |
| **Good for** | Web apps you want to share or open from any device (e.g. Nextcloud, Jellyfin, a status page) | Admin interfaces, SSH, file shares, anything that should never be public |
 
**Recommended rule of thumb:**
- Everything is reachable via **Tailscale** by default.
- Only services that really need to be reachable without a VPN client are published via **Cloudflare Tunnel**, and ideally protected with **Cloudflare Access**.
- Admin panels (Proxmox, router, NAS admin, Portainer, …) are **never** published via the tunnel.
```
                 Internet
                    │
     ┌──────────────┴──────────────┐
     │                             │
 Cloudflare                    Tailscale
     │                             │
     │ outbound tunnel             │ encrypted connection
     ▼                             ▼
┌─────────────────── Homelab ───────────────────┐
│  cloudflared  ──►  web services (http://...)  │
│  tailscaled   ──►  all services / subnet      │
└───────────────────────────────────────────────┘
```
 
---

## 2. Prerequisites
 
- A Linux host in your homelab (VM, LXC, Raspberry Pi or bare metal) that is always on. The examples use Debian/Ubuntu.
- `sudo` rights on this host (aka. root access).
- Your services are running and reachable **inside** your home network (e.g. `http://192.168.xxx.xxx:{port}`).
- For Cloudflare Tunnel:
  - A free Cloudflare account. (You do **not** need a paid plan!)
  - A domain whose nameservers point to Cloudflare (added as a zone in the Cloudflare dashboard). For details look [here](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/) or ask AI (lol).
- For Tailscale:
  - A free [Tailscale](https://tailscale.com/) account (login via Google, Microsoft, GitHub, …).
> **Placeholder values used in this guide** – replace them with your own:
> `example.com` = your domain, `192.168.1.0/24` = your home subnet, `homelab` = name of the tunnel.
 
---
 
## 3. Public access via Cloudflare Tunnel
 
A Cloudflare Tunnel runs a small daemon (`cloudflared`) in your network. It opens an outbound connection to Cloudflare, and Cloudflare forwards requests for your (sub)domains through this connection to your internal services.

### Dashboard-managed tunnel (recommended)
 
1. Open the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/) and navigate to **Networks → Tunnels**.
2. Click **Create a tunnel**, choose **Cloudflared** as connector type and give it a name (e.g. `homelab`).
3. Cloudflare now shows install commands including a **token**. Choose your environment:
   **Linux service (Debian/Ubuntu):**
   Install `cloudflared` using the commands shown in the dashboard, then run:
```bash
   sudo cloudflared service install <YOUR_TUNNEL_TOKEN>
```
  You **could** also run cloudflared as docker, which I do **not** recommend:
```bash
  sudo docker run -d \
    --name cloudflared \
    --restart unless-stopped \
    -e TUNNEL_TOKEN=<YOUR_TUNNEL_TOKEN> \
    cloudflare/cloudflared:latest \
    tunnel --no-autoupdate run
```
 
4. Back in the dashboard the connector should show as **Healthy**. Click **Next**.
5. Add a **Public Hostname** for each service:
   | Field | Example |
   | -------- | ------- |
   | Subdomain | `{subdomain}` |
   | Domain | `example.com` |
   | Service type | `HTTP` |
   | URL | `192.168.xxx.xxx:{port}` |
6. Save. Cloudflare automatically creates the DNS record. After a few seconds `https://{subdomain}.example.com` should load your service, including a valid SSL/TLS certificate.
> **Service uses HTTPS with a self-signed certificate internally?**
> Set the service type to `HTTPS` and enable **Additional application settings → TLS → No TLS Verify**.

### 3.1 Securing public services with Cloudflare Access
 
Without further protection, every service published through the tunnel is reachable by anyone who knows the URL. Cloudflare Access adds a login page **in front of** your service, before traffic ever reaches your homelab.
 
1. In the Zero Trust dashboard go to **Access → Applications → Add an application → Self-hosted**.
2. Enter a name and the hostname (e.g. `{subdomain}.example.com`).
3. Create a **policy**, for example:
   - Action: `Allow`
   - Include: `Emails` → `you@example.com`
4. As login method, the built-in **One-time PIN** (code via email) works out of the box. Google, GitHub etc. can be added under **Settings → Authentication**.
5. Save. Opening the URL now shows a Cloudflare login first.
> **Note:** Native mobile/TV apps (e.g. the Jellyfin app) often can't handle the Access login page. For those services, either access them via Tailscale instead or rely on the service's own authentication.
 
---

## 4. Private access via Tailscale VPN
 
Tailscale builds a private, encrypted network (**Tailnet**) between all your devices based on WireGuard. Devices connect directly to each other where possible; no ports need to be opened.
 
### 4.1 Install Tailscale on the homelab host
 
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
 
Open the printed link and log in. The host now appears in the [Tailscale admin console](https://login.tailscale.com/admin/machines) with a `100.xxx.xxx.xxx` IP address.
 
```bash
tailscale ip -4      # shows the Tailscale IP of this host
tailscale status     # shows all devices in your Tailnet
```

### 4.2 Install Tailscale on your clients
 
Install the Tailscale app on your laptop, phone, etc. ([download](https://tailscale.com/download)) and log in with the **same account**.
 
You can now reach the homelab host from anywhere, e.g.:
```bash
ssh user@100.x.y.z
```
or open `http://100.x.y.z:8096` in the browser. Note, that you have to enable the vpn in the tailscale app manually!
 
### 4.3 Recommended settings in the admin console
 
- **MagicDNS** (*DNS* tab): enable it to use hostnames instead of IPs, e.g. `http://homelab:8096` or `ssh user@homelab`.
- **Disable key expiry** for servers (*Machines → ⋯ → Disable key expiry*). Otherwise the server drops out of the Tailnet after the key expires (default 180 days) and needs a new login – usually when you're not at home.
### 4.4 Exit Node
 
If you want to be able to connect to devices inside your home network, without having each of them run tailscale, you can just set up *advertise as exit node* on the server:
```bash
sudo tailscale up --advertise-exit-node
```
Approve it in the admin console and select it as exit node on your client.
 
---
 
## 5. Verification & Troubleshooting
 
**Verify:**
- [ ] `https://<service>.example.com` loads from mobile data (Wi-Fi turned off).
- [ ] Protected services show the Cloudflare Access login first.
- [ ] With Tailscale connected on your phone (mobile data), `http://homelab:<port>` or the internal IP works.
- [ ] With Tailscale **disconnected**, internal-only services are **not** reachable.
**Common problems:**
 
| Problem | Likely cause / fix |
| -------- | ------- |
| Cloudflare error **502 Bad Gateway** | `cloudflared` can't reach the service. Check the service URL/port and test with `curl` from the `cloudflared` host/container. |
| Cloudflare error **1033** | Tunnel is not running or not connected. Check `sudo systemctl status cloudflared` or `docker logs cloudflared`. |
| Redirect loop / "too many redirects" | Service redirects HTTP → HTTPS itself. Point the tunnel to the HTTPS port or disable the redirect in the service. |
| Tailscale device shows as **offline** | Check `sudo systemctl status tailscaled`, check whether the key has expired. |
| Home network not reachable via exit node | Exit node not approved in admin console or not selected on the client, or IP forwarding not enabled on the server. |
| MagicDNS names don't resolve | MagicDNS disabled, or client uses a custom DNS that overrides Tailscale's DNS. |
 
**Useful commands:**
```bash
# Cloudflare
sudo systemctl status cloudflared
sudo journalctl -u cloudflared -f
cloudflared tunnel list
 
# Tailscale
tailscale status
tailscale ping <device>
tailscale netcheck
```
 
---
 
## 6. Security Checklist
 
- [ ] No ports are forwarded on the router (neither tool needs it).
- [ ] Admin interfaces (Proxmox, router, NAS, Portainer, …) are **only** reachable via Tailscale.
- [ ] Every service published via Cloudflare Tunnel either has Cloudflare Access in front of it or strong own authentication (+ 2FA where possible).
- [ ] Tunnel token / credentials file is not committed to Git (use `.env` / secrets).
- [ ] Key expiry is disabled only for servers, not for laptops/phones.
- [ ] Unused devices are removed from the Tailnet.
- [ ] Optional: Tailscale **ACLs** restrict which devices may access which (e.g. the phone may reach Jellyfin, but not SSH).
- [ ] Your Cloudflare and Tailscale accounts are protected with 2FA.
