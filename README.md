# How I installed Opencloud on a Mini PC and have data on TrueNas

NOTE: This is my personal experince, and not a official guide!


# OpenCloud Installation & Configuration Guide (Bare‑metal, with TrueNAS + NPM)

This document captures a **clean, reproducible install** of OpenCloud on Ubuntu with:

- **All OpenCloud state & user data on a TrueNAS NFS share** (`/mnt/opencloud`) for easy recovery.
- **Systemd** service management.
- **Nginx Proxy Manager (NPM)** doing HTTPS + reverse proxy.
- Practical **troubleshooting** notes from a real setup.

> Tip: run commands **one at a time** and verify the result before moving on.


---

## 0) Prereqs (what you have)

- Ubuntu server (fresh or minimal).
- NAS export: `<your_nas_ip>:/mnt/tank_nvme/opencloud`.
- NPM at `<your_npm>` with a public/local DNS name: `<your_domain>`.
- A dedicated Linux user `opencloud`. (If not present, see §1).


---

## 1) Create service user and dirs (if not already)

**Why:** Run OpenCloud as a non-login service user for safety.

**Run:**
```bash
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/opencloud opencloud
sudo mkdir -p /var/lib/opencloud
sudo chown -R opencloud:opencloud /var/lib/opencloud
```

> If the user already exists, you’ll see errors—OK to ignore.


---

## 2) Mount the NAS at `/mnt/opencloud`

**Why:** We want *all* state + user files on the NAS so a host failure is painless.

### 2.1 Add to `/etc/fstab`

**Edit:**
```bash
sudo nano /etc/fstab
```

**Add line (one line):**
```
your_nas_ip:/mnt/tank_nvme/opencloud  /mnt/opencloud  nfs4  rw,noatime,nofail,_netdev,x-systemd.automount,x-systemd.device-timeout=10s  0  0
```

**Apply + start automount:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart mnt-opencloud.automount || true
```

**Trigger mount & verify:**
```bash
ls /mnt/opencloud
mount | grep /mnt/opencloud
```

You should see an NFS mount listed.


### 2.2 Confirm the service user can write

**Run:**
```bash
sudo -u opencloud bash -lc 'touch /mnt/opencloud/.oc-write && ls -l /mnt/opencloud/.oc-write && rm -f /mnt/opencloud/.oc-write'
```

If that fails, fix permissions (see §8 “NFS permissions”).


---

## 3) Install OpenCloud binary

**Why:** We’re doing a non-Docker install managed by systemd.

**Download & install (example: v3.4.0):**
```bash
cd /tmp
wget -q https://github.com/opencloud-eu/opencloud/releases/download/v3.4.0/opencloud-linux-amd64.zip
unzip -o opencloud-linux-amd64.zip
sudo mv -f opencloud /usr/local/bin/opencloud
/usr/local/bin/opencloud --version
```


---

## 4) Config directories & logs

**Why:** Keep config under `/etc/opencloud` and send logs to `/var/log/opencloud` (or use `journalctl`).

**Create:**
```bash
sudo mkdir -p /etc/opencloud /var/log/opencloud
sudo chown -R opencloud:opencloud /etc/opencloud /var/log/opencloud
```

> If you plan to log to a file, we’ll set `OC_LOG_FILE` and ensure the file exists (see §6).


---

## 5) Environment file (`/etc/opencloud/.env`)

**Why:** In a systemd setup, OpenCloud only sees env vars you give it. This file is loaded by the service.

**Create/edit:**
```bash
sudo nano /etc/opencloud/.env
```

**Recommended content:**
```dotenv
# Core paths
OC_CONFIG_DIR=/etc/opencloud
# Some builds use OC_BASE_DATA_PATH, others OC_DATA_DIR;
# set BOTH to be safe. They should point to the NAS base path.
OC_BASE_DATA_PATH=/mnt/opencloud
OC_DATA_DIR=/mnt/opencloud

# External URL & behavior
OC_URL=https://<your_url>
OC_INSECURE=true
PROXY_ENABLE_BASIC_AUTH=true

# Logging (file logging is optional; journal works too)
OC_LOG_LEVEL=warning
OC_LOG_FILE=/var/log/opencloud/opencloud.log

# Demo content
IDM_CREATE_DEMO_USERS=false
```

**Secure the env file:**
```bash
sudo chown root:root /etc/opencloud/.env
sudo chmod 600 /etc/opencloud/.env
```


---

## 6) YAML config (`/etc/opencloud/opencloud.yaml`)

**Why:** Make storage explicit; avoids surprises if env var names change.

**Create/edit:**
```bash
sudo nano /etc/opencloud/opencloud.yaml
```

**Minimal content:**
```yaml
storage:
  data: "/mnt/opencloud"
```

**Create expected initial directories (prevents first-run errors):**
```bash
sudo -u opencloud bash -lc 'mkdir -p /mnt/opencloud/{idm,idp,nats,proxy,search,storage}'
sudo -u opencloud bash -lc 'mkdir -p /mnt/opencloud/storage/users/{nodes,uploads,trash,indexes}'
```

**If using file logging, ensure log file exists & is writable:**
```bash
sudo install -o opencloud -g opencloud -m 750 -d /var/log/opencloud
sudo install -o opencloud -g opencloud -m 640 /dev/null /var/log/opencloud/opencloud.log
```


---

## 7) Systemd service

**Why:** Auto-start on boot, restart-on-failure, env injection.

**Create service file:**
```bash
sudo nano /etc/systemd/system/opencloud.service
```

**Content:**
```ini
[Unit]
Description=OpenCloud service
After=network-online.target mnt-opencloud.mount
Requires=network-online.target mnt-opencloud.mount

[Service]
User=opencloud
Group=opencloud
EnvironmentFile=/etc/opencloud/.env
# You can pass --config explicitly or rely on OC_CONFIG_DIR
ExecStart=/usr/local/bin/opencloud server
Restart=always
RestartSec=5s
# Working dir for any relative paths
WorkingDirectory=/var/lib/opencloud

[Install]
WantedBy=multi-user.target
```

**Enable & start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opencloud
```

**Check:**
```bash
sudo systemctl status opencloud --no-pager
```

You should see `active (running)`.


---

## 8) NFS permissions (important)

Uploads failing after moving data to NAS is almost always a UID/GID mismatch. Confirm ownership shown via `ls -l /mnt/opencloud`.

You have **two clean options**:

### Option A — Align Ubuntu `opencloud` user to NAS IDs
If NAS shows `3003:3002` as owner:

```bash
sudo systemctl stop opencloud
sudo usermod -u 3003 opencloud
sudo groupmod -g 3002 opencloud
sudo chown -R opencloud:opencloud /var/lib/opencloud /etc/opencloud /var/log/opencloud
sudo systemctl start opencloud
```

### Option B — Map via TrueNAS NFS export (mapall)
In the TrueNAS NFS share, set **mapall user/group** to `opencloud` so all remote accesses map to the NAS’ `opencloud` user. No UID changes on Ubuntu needed.

> Either method works; choose one.


---

## 9) Nginx Proxy Manager (NPM)

**Goal:** `https://<your_url>` → OpenCloud backend (`https://opencloud-host:9200`).

- In NPM, add a **Proxy Host**:
  - Domain: `<your domain>`
  - Scheme: `https`
  - Forward Hostname/IP: `<opencloud host IP>`
  - Forward Port: `9200`
  - SSL: **Request a new certificate** (Let’s Encrypt). Force SSL if you want.

- **Custom Nginx config** (Advanced tab) – helpful headers/timeouts:
  ```nginx
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-Forwarded-Host $host;

  # WebSocket support
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";

  # Large uploads
  client_max_body_size 10G;
  proxy_read_timeout 3600s;
  proxy_send_timeout 3600s;
  ```

**Test the chain:**  
```bash
curl -skI https://127.0.0.1:9200/ | head -n1          # backend → expect HTTP/1.1 200
curl -vkI https://<your_domain> | head -n1          # through NPM → expect HTTP/2 200
```

If your certificate lives on NPM (it does), SNI/SSL is handled there—backend may still be HTTPS; that’s fine.


---

## 10) Verification (uploads land on NAS)

**Upload a tiny file via the web UI**, then confirm it appears under NAS storage:

```bash
sudo find /mnt/opencloud/storage -type f -mmin -5 -printf '%TY-%Tm-%Td %TH:%TM:%TS %p\n' | tail -n 20
```

Expected user paths look like:
```
/mnt/opencloud/storage/users/users/<user-uuid>/...
```


---

## 11) Logs & rotation

If you set `OC_LOG_FILE=/var/log/opencloud/opencloud.log`:

- Tail logs:
  ```bash
  sudo tail -f /var/log/opencloud/opencloud.log
  ```

- Optional logrotate `/etc/logrotate.d/opencloud`:
  ```bash
  sudo tee /etc/logrotate.d/opencloud >/dev/null <<'ROT'
/var/log/opencloud/*.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    copytruncate
}
ROT
  ```

Alternatively, skip file logs and use:
```bash
journalctl -u opencloud -f
```


---

## 12) Troubleshooting (common)

- **502 via NPM:**  
  Backend not running or wrong upstream. Check:
  ```bash
  sudo systemctl status opencloud --no-pager
  curl -skI https://127.0.0.1:9200/ | head -n1
  ```

- **NATS/JetStream “no servers available” / “could not create storage directory”:**  
  Ensure the base path exists and is owned by `opencloud`:
  ```bash
  sudo mkdir -p /mnt/opencloud/nats
  sudo chown -R opencloud:opencloud /mnt/opencloud
  ```

- **Token expired (OIDC):**  
  Log out in the browser, clear cookies for the site, log in again. Verify system time:
  ```bash
  timedatectl status
  ```

- **Uploads fail after moving to NAS:**  
  UID/GID mismatch. Fix via §8.

- **Service fails at boot with dependency:**  
  Use `x-systemd.automount` in fstab and keep:
  ```ini
  After=network-online.target mnt-opencloud.mount
  Requires=network-online.target mnt-opencloud.mount
  ```


---

## 13) Backup & snapshots (outline)

- **NAS snapshots**: hourly (2 days), daily (14 days), weekly (8 weeks) is a solid rotation.
- **Proxmox Backup Server (PBS)**: replicate long-term copies (and optionally to Backblaze B2).
- Keep a copy of `/etc/opencloud` (config) in your backups.

> Recovery is simple: new host → mount NAS → install OpenCloud binary → restore `/etc/opencloud` → start service.


---

## 14) Quick health checklist

- Service enabled:
  ```bash
  systemctl is-enabled opencloud
  ```

- Process has the right env vars:
  ```bash
  sudo cat /proc/$(pidof opencloud)/environ | tr '\0' '\n' | grep -E '^OC_(BASE_DATA_PATH|DATA_DIR|CONFIG_DIR)='
  ```

- Storage root exists & owned by service user:
  ```bash
  ls -ld /mnt/opencloud
  ```

- UI reachable:
  ```bash
  curl -vkI https://<your_domanin> | head -n1
  ```


---

## 15) Appendix: Changing UIDs safely

If you decide to align Ubuntu `opencloud` to NAS’ `3003:3002` later:

```bash
sudo systemctl stop opencloud
sudo usermod -u 3003 opencloud
sudo groupmod -g 3002 opencloud
sudo chown -R opencloud:opencloud /var/lib/opencloud /etc/opencloud /var/log/opencloud
sudo systemctl start opencloud
```

Verify:
```bash
id opencloud
ls -ld /mnt/opencloud
```

---

**End of Guide** — Happy syncing!
