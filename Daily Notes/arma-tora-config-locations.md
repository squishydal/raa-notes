# arma.tora: where everything lives

Reference for the VM `devops-165` (Debian trixie) and the laptop `raa-bft`. Compiled from the conversation. Anything marked **(not seen)** was never shown, so verify it with the command next to it.

> **Secrets** (Postgres password, `client_secret` values, htpasswd hashes) are intentionally not copied here. They live in the files listed below.

---

## 0. The big picture

```
browser ──► nginx :80 ──► /diga/acc/*  ──► diga  127.42.9.87:42987
                    │                         │  (calls dugu_url from the VM itself)
                    ├──► /dugu/*      ──► dugu  127.5.32.21:53221
                    ├──► /api/*       ──► bivent 127.0.0.1:33891   (needs login + X-Server-Key)
                    └──► /            ──► static frontend /var/www/vhosts/arma.tora/arma/
```

Login flow: `/` → diga says "no token" → `/diga/acc/landing` → `/diga/acc/login` → diga asks dugu (`POST /init`) → redirect to `/dugu/sess/...` → login form → `/diga/acc/login/ok` → cookie set → app.

---

## 1. Addresses and ports

| Service | Address | Defined in |
|---|---|---|
| nginx | `:80` (all addresses) | `/etc/nginx/sites-available/arma.tora` (`listen`) |
| diga | `127.42.9.87:42987` | `/etc/diga/diga.yaml` (`addr`) and nginx `proxy_pass` in `diga-arma.conf` |
| dugu | `127.5.32.21:53221` | `-addr` flag in `/etc/systemd/system/dugu.service` and `proxy_pass` in `dugu-arma.conf` |
| bivent | `:33891` | `web.addr` in `/opt/bivent/etc/bivent.yaml` and `proxy_pass` in `diga-arma.conf` |
| Postgres (Docker) | `127.0.0.1:15432` | `/opt/pg-bivent/.env` and `docker-compose.yml`, and `database.port` in `bivent.yaml` |

A change to any address must be made in **both** places shown in the last column.

---

## 2. Package and system layer (VM)

| What | Path |
|---|---|
| Binaries | `/usr/bin/bivent`, `/usr/bin/dugu`, `/usr/bin/diga` (from `apt install`; confirm with `which bivent dugu diga`) |
| BFI apt key | `/etc/apt/trusted.gpg.d/bfi-repo.gpg` (armored, so apt warns "unsupported filetype"; cosmetic) |
| BFI apt source | `/etc/apt/sources.list.d/bfi.sources` (contents **not seen**) |
| Docker apt key | `/etc/apt/keyrings/docker.asc` |
| Docker apt source | `/etc/apt/sources.list.d/docker.sources` |
| Systemd units | `/etc/systemd/system/dugu.service`, `diga.service`, `bivent.service` |
| Hosts file | `/etc/hosts` (VM needs `127.0.0.3  arma.tora`) |

Notes:
- `bivent.service`: on the laptop it is a symlink `/etc/systemd/system/bivent.service -> /opt/bivent/etc/bivent.service`. How it was installed on the VM is **not seen**; check with `ls -l /etc/systemd/system/bivent.service`.
- Command lines in the units: `dugu web -addr 127.5.32.21:53221 -source /etc/dugu/source.yaml -weblog /var/log/dugu/dugu.log`, `diga web -config /etc/diga/diga.yaml`, `bivent -c bivent.yaml web`.
- Enable on boot: `systemctl enable bivent dugu diga nginx docker` (bivent was still `disabled`; check with `systemctl is-enabled bivent dugu diga nginx docker`).

---

## 3. nginx

| What | Path |
|---|---|
| Main config | `/etc/nginx/nginx.conf` |
| Vhost | `/etc/nginx/sites-available/arma.tora` |
| Enabled link | `/etc/nginx/sites-enabled/arma.tora` (symlink to the file above; nginx only loads `sites-enabled`) |
| diga routes | `/etc/nginx/sites-available/include-arma-bfit/diga-arma.conf` |
| dugu route | `/etc/nginx/sites-available/include-arma-bfit/dugu-arma.conf` |
| Old/backup | `/etc/nginx/sites-available/include-arma-bfit/arma-w7.bak` (ignored: only `*.conf` is included) |
| Password file | `/etc/nginx/_dugu-arma-htaccess` (htpasswd, bcrypt; `bfile.yaml` of dugu points here) |
| Access log | `/var/log/nginx/arma_bfit_access.log` |
| Error log | `/var/log/nginx/arma_bfit_error.log` |

`diga-arma.conf` contains, in order:
1. `@diga401HTML` / `@diga500HTML` / `@diga401JSON` error handlers
2. `/diga-auth` (internal auth check to diga)
3. `/diga/acc/` (login pages, proxied to diga)
4. `/` (static frontend, **`alias /var/www/vhosts/arma.tora/arma/`**, injects `w/script.js`)
5. `/api/` (to bivent, sets **`X-Server-Key`** and `X-Username`)
6. `/api/diag/` (blocked), `/bivent-protected-attns/` (attachments, `alias /opt/bivent/var/lib/attns/`)
7. `/w/` (serves `htdocs/` for `script.js`)

Commands: `nginx -t` (test) then `systemctl reload nginx`. Show what is really loaded: `nginx -T | less`.

---

## 4. Frontend and `script.js`

| What | Path |
|---|---|
| Built frontend (Angular) | `/var/www/vhosts/arma.tora/arma/` (copied with `rsync -avzP arma bfi-devops-165:~/`) |
| Logout link script | `/var/www/vhosts/arma.tora/htdocs/script.js` |
| Vhost document root | `/var/www/vhosts/arma.tora/htdocs` (`root` in the vhost) |

`script.js` is one line: `document.logoutURL = "http://arma.tora/diga/acc/logout";` (no port). nginx injects it before `</body>` and serves it at `/w/script.js`. Test: `curl http://arma.tora/w/script.js`.

---

## 5. diga

| What | Path |
|---|---|
| Config | `/etc/diga/diga.yaml` (default template: `diga.yaml.default`) |
| Log | `/var/log/diga/diga.log` (root-only, so use `sudo`) |

Fields that must match other files:

| `diga.yaml` field | Must equal |
|---|---|
| `auth_name` | the entry name in `/etc/dugu/source.yaml` (`arma-tora-htaccess`) |
| `client_id`, `client_secret` | the same two values in that `source.yaml` entry |
| `dugu_url` | `http://arma.tora/dugu`, and **the VM must be able to reach it** |
| `redirect_ok` | `http://arma.tora` |
| `addr` | the `proxy_pass` address in `diga-arma.conf` |
| `cookie` | `arma-cookie` (bivent `web.cookie_ns` uses the same name) |

Sample nginx config printed by: `diga nginx`. Sample config: `diga config`.

---

## 6. dugu

| What | Path |
|---|---|
| Source list (authenticators) | `/etc/dugu/source.yaml` (also `.bak`, `.bak2`, `.default`) |
| Entry data | `/var/lib/dugu/arma-tora-htaccess/bfile.yaml` (`path: /etc/nginx/_dugu-arma-htaccess`) |
| One-time-password file | `/var/lib/dugu/arma-tora-htaccess/otp/arma-tora-htaccess.yaml` (tiny, 3 bytes) |
| Unused basic entry | `/var/lib/dugu/arma-tora/` (`bfile.yaml`, `otp/arma-tora.yaml`) |
| Log | `/var/log/dugu/dugu.log` |

Fields in `source.yaml` entry `arma-tora-htaccess`:
- `redirect_uri: http://arma.tora/diga/acc/login/ok`
- `bname: htaccess`
- `bfile` and `otp_file` as in the table above

YAML warning: every field is indented 2 spaces; one unindented line crashed dugu earlier.

Generate a new entry: `dugu generate -name <name> >> /etc/dugu/source.yaml` (`>>` appends, `>` overwrites the file).

---

## 7. bivent

| What | Path |
|---|---|
| Config | `/opt/bivent/etc/bivent.yaml` |
| Unit file source | `/opt/bivent/etc/bivent.service` (laptop; VM **not seen**) |
| Allowed users | `/opt/bivent/var/lib/userdata/users.json` (e.g. `["tora@arma.tora"]`) |
| Data directory | `/opt/bivent/var/lib/bivent` (`datadir`) |
| Attachments | `/opt/bivent/var/lib/attns/` (must match nginx `/bivent-protected-attns/` alias) |
| Other dirs | `/opt/bivent/var/lib/i18n`, `/opt/bivent/var/lib/idents` |
| Logs | `/opt/bivent/logs/bivent-web.log`, `/opt/bivent/logs/bivent.log` (the folder must exist, or "cannot open logger writer") |
| Service journal | `journalctl -u bivent -n 50 --no-pager` |

Fields that must match other files:

| `bivent.yaml` field | Must equal |
|---|---|
| `database.host/port/user/password/dbname` | `/opt/pg-bivent/.env` |
| `web.auth.header_server_val` | the `X-Server-Key` value in `diga-arma.conf` (`header-test-arma`) |
| `web.auth.header_server_key` | `x-server-key` |
| `web.auth.header_username` | `x-username` |
| `web.auth.user_fname` | path of `users.json` |
| `web.cors` | `http://arma.tora` |
| `web.cookie_ns` | `arma-cookie` |
| `xsendfile_location` | the `/bivent-protected-attns/` location in nginx |

---

## 8. Postgres (Docker)

| What | Path |
|---|---|
| Compose file | `/opt/pg-bivent/docker-compose.yml` |
| Credentials | `/opt/pg-bivent/.env` (`POSTGRES_ADDR`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`) |
| Data | `/opt/pg-bivent/data/` (seen on the laptop; VM **not seen**) |

Commands: `cd /opt/pg-bivent && docker compose up -d`, `docker compose ps`, `docker compose logs --tail 30`.

---

## 9. Name resolution (the thing that caused the 503)

| Machine | File | Line |
|---|---|---|
| VM | `/etc/hosts` | `127.0.0.3  arma.tora` (so diga's call to `dugu_url` reaches the VM's own nginx, not another server found through DNS) |
| Laptop | `/etc/hosts` | `127.0.0.1  arma.tora` (with the tunnel) or `192.168.1.165  arma.tora` (direct to the VM) |

Check on each machine: `getent hosts arma.tora`.

---

## 10. Laptop side

| What | Where |
|---|---|
| Tunnel | `ssh -N -L 127.0.0.1:80:localhost:80 bfi-devops-165` (leave running) |
| Port 80 permission | `sudo sysctl net.ipv4.ip_unprivileged_port_start=80` (lost at reboot) |
| ssh alias `bfi-devops-165` | `~/.ssh/config` (or a conf.d include) |
| Old `arma.test` setup (reference only) | `/etc/nginx/sites-available/arma.test`, `/etc/dugu/`, `/etc/diga/`, `/opt/bivent/`, `/var/www/vhosts/arma.test/` |
| Nginx config download | `~/Downloads/nginx-config/` |

Stop the laptop's own nginx (`sudo systemctl stop nginx`) if it holds port 80.

---

## 11. Logs, one place to look

```bash
tail -F /var/log/diga/diga.log /var/log/dugu/dugu.log \
        /var/log/nginx/arma_bfit_access.log /var/log/nginx/arma_bfit_error.log
journalctl -u bivent -u dugu -u diga -n 50 --no-pager
```

| Line | Meaning |
|---|---|
| diga `401 /auth "Token not found."` | Normal: visitor not logged in yet |
| diga `503 ... backend: 404` | diga cannot reach `dugu_url` (see section 9) |
| bivent `EConfigDBPingFailed` | Postgres down, or wrong port/credentials |
| dugu `POST /init` from `Go-http-client` | diga reached dugu (good) |

---

## 12. "Where is it?" commands

```bash
systemctl cat dugu diga bivent                      # real unit files and paths
nginx -T 2>/dev/null | grep -n "include\|alias\|root\|proxy_pass"
grep -rIn "arma.test\|arma-test\|127.0.2.2" /etc /opt /var/lib /var/www 2>/dev/null   # leftovers
find / -xdev -name 'script.js' -path '*htdocs*' 2>/dev/null
ss -tlnp | grep -E "nginx|diga|dugu|bivent|15432"   # who listens where
```

Restart after any config change: `systemctl restart bivent dugu diga` and `nginx -t && systemctl reload nginx`.

---

## 13. Quick consistency checklist

1. `diga.yaml` ↔ `source.yaml`: `auth_name` = entry name, `client_id` and `client_secret` identical.
2. All URLs (`dugu_url`, `redirect_ok`, `redirect_uri`, `script.js`) use the same host and port as the browser address bar.
3. `X-Server-Key` in nginx = `header_server_val` in `bivent.yaml`.
4. `users.json` contains the login username; the htpasswd file has the same username.
5. `getent hosts arma.tora` gives `127.0.0.3` on the VM.
6. Every `proxy_pass` matches a listening address in `ss -tlnp`.
