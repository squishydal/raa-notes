# arma.tora setup on Debian trixie (devops-165)

Compiled from the working setup. Steps marked **(not seen)** were never shown in the conversation, so check them against the VM.

Stack: `bivent` (backend API) + `dugu` (login/authenticator) + `diga` (nginx auth gateway) + nginx + Postgres in Docker.

> **Secrets:** a Postgres password, `client_secret` values and a `gogen` password were pasted in chat. Regenerate them if the VM stays up. Placeholders below are `<...>`.

---

## 1. BFI apt repo and packages

```bash
curl -sL https://pkg.bfi.io/key -o /etc/apt/trusted.gpg.d/bfi-repo.gpg
# repo entry: /etc/apt/sources.list.d/bfi.sources   (contents not seen)
apt update
apt install bivent dugu diga
```

The `bfi-repo.gpg ... unsupported filetype` warning is cosmetic (armored key with a `.gpg` name). It was ignored.

## 2. Docker + Postgres

Use the **Debian** guide (the Ubuntu URL fails on trixie).

```bash
apt install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

`/opt/pg-bivent/` holds `docker-compose.yml` and `.env`:
- Postgres on `127.0.0.1:15432`, database `arma_db`
- user and password must match `bivent.yaml`
- start with `cd /opt/pg-bivent && docker compose up -d`

Compose file on the VM: **(not seen)**.

## 3. nginx and vhost

```bash
apt install nginx rsync apache2-utils
```

`/etc/nginx/sites-available/arma.tora`:

```nginx
server {
	server_name arma.tora;
	listen 80;
	listen [::]:80;
	access_log /var/log/nginx/arma_bfit_access.log;
	error_log  /var/log/nginx/arma_bfit_error.log;
	root /var/www/vhosts/arma.tora/htdocs;
	index index.html index.htm index.nginx-debian.html;
	include /etc/nginx/sites-available/include-arma-bfit/*.conf;
}
```

```bash
ln -s /etc/nginx/sites-available/arma.tora /etc/nginx/sites-enabled/arma.tora
```

`include-arma-bfit/` (copied from the laptop with rsync) contains `diga-arma.conf` and `dugu-arma.conf`. `diga nginx` prints the sample.

## 4. Settings inside `include-arma-bfit/`

| File | Setting | Value |
|---|---|---|
| `dugu-arma.conf` | `/dugu/` proxy_pass | `http://127.5.32.21:53221` |
| `diga-arma.conf` | `/diga-auth` and `/diga/acc/` proxy_pass | `http://127.42.9.87:42987` |
| `diga-arma.conf` | `/api/` proxy_pass | `http://127.0.0.1:33891` |
| `diga-arma.conf` | frontend alias | `/var/www/vhosts/arma.tora/arma/` (it had a `vhost` typo) |
| `diga-arma.conf` | `X-Server-Key` | must equal `header_server_val` in `bivent.yaml` (`header-test-arma`) |

The `X-Server-Key` line was found as `header-tora-arma` while `bivent.yaml` said `header-test-arma`. Final value on the VM: **(not seen)**.

## 5. Frontend

```bash
# from the laptop:  rsync -avzP arma bfi-devops-165:~/
mkdir -p /var/www/vhosts/arma.tora/htdocs
mv ~/arma /var/www/vhosts/arma.tora/arma
echo 'document.logoutURL = "http://arma.tora/diga/acc/logout";' \
  > /var/www/vhosts/arma.tora/htdocs/script.js
```

## 6. bivent

```bash
mkdir -p /opt/bivent/etc /opt/bivent/logs /opt/bivent/var/lib/{attns,i18n,idents,userdata}
bivent config > /opt/bivent/etc/bivent.yaml
echo '["tora@arma.tora"]' > /opt/bivent/var/lib/userdata/users.json
```

Edit `bivent.yaml`:
- `database`: host `127.0.0.1`, port `15432`, user/password/`dbname: arma_db` from `.env`
- `web.addr`: port `33891`
- `web.cors`: `http://arma.tora`
- `web.auth`: `user_fname` → `users.json`, `header_username: x-username`, `header_server_key: x-server-key`, `header_server_val: header-test-arma`
- `logfname` and `fname` paths must exist under `/opt/bivent/logs/` (otherwise: `cannot open logger writer`)

Unit command: `/usr/bin/bivent -c bivent.yaml web`. Unit file contents: **(not seen)**.

## 7. dugu (htaccess login)

```bash
mkdir -p /var/lib/dugu/arma-tora-htaccess/otp /var/log/dugu /var/log/diga
echo 'path: /etc/nginx/_dugu-arma-htaccess' > /var/lib/dugu/arma-tora-htaccess/bfile.yaml
htpasswd -c -B /etc/nginx/_dugu-arma-htaccess tora@arma.tora   # -c only for the first user
cd /etc/dugu && cp source.yaml source.yaml.bak
dugu generate -name arma-tora-htaccess >> source.yaml          # >> appends, > overwrites
```

Also create `otp/arma-tora-htaccess.yaml` (a tiny 3-byte file, copied from the laptop setup).

Edit the generated entry in `/etc/dugu/source.yaml`:

```yaml
arma-tora-htaccess:
  client_id: <generated>
  client_secret: <generated>
  redirect_uri: http://arma.tora/diga/acc/login/ok   # 2-space indent, no port
  bname: htaccess
  bfile: /var/lib/dugu/arma-tora-htaccess/bfile.yaml
  otp_file: /var/lib/dugu/arma-tora-htaccess/otp/arma-tora-htaccess.yaml
```

YAML indentation matters: an unindented `redirect_uri` line broke the file earlier.

## 8. diga

`/etc/diga/diga.yaml`:

```yaml
dugu_url: http://arma.tora/dugu
auth_name: arma-tora-htaccess        # name of the source.yaml entry
client_id: <same as source.yaml>     # must be identical
client_secret: <same as source.yaml> # must be identical
addr: 127.42.9.87:42987
cookie: arma-cookie
expire: 3600
redirect_ok: http://arma.tora
weblog: /var/log/diga/diga.log
```

Service commands:
- `dugu web -addr 127.5.32.21:53221 -source /etc/dugu/source.yaml -weblog /var/log/dugu/dugu.log`
- `diga web -config /etc/diga/diga.yaml`

## 9. The fix that unblocked the 503

diga calls `dugu_url` from the VM itself. On the VM, `arma.tora` resolved through DNS to a different server, which answered 404, so diga returned 503. Pin the name to the VM:

```bash
echo '127.0.0.3  arma.tora' >> /etc/hosts
getent hosts arma.tora                    # must print 127.0.0.3
curl -i http://arma.tora/dugu/style.css   # must return 200 with CSS
```

## 10. Start everything

```bash
systemctl enable --now bivent
systemctl restart dugu diga
nginx -t && systemctl reload nginx
systemctl is-active bivent dugu diga
```

## 11. Laptop side

```bash
# /etc/hosts:   127.0.0.1  arma.tora
sudo systemctl stop nginx                              # only if the laptop's nginx holds port 80
sudo sysctl net.ipv4.ip_unprivileged_port_start=80     # lost at reboot
ssh -N -L 127.0.0.1:80:localhost:80 bfi-devops-165     # leave running
```

Open `http://arma.tora` in a private window and log in as `tora@arma.tora`. Keep the URL **without a port**, because every redirect in the config has no port. A direct alternative, with no tunnel: put `192.168.1.165  arma.tora` in the laptop's hosts file.

---

## Debugging cheatsheet

| Symptom | Where to look |
|---|---|
| Log line `401 /auth "Token not found."` | Normal: first step of login for a visitor with no cookie |
| 503 on `/diga/acc/login`, diga log `Invalid status code from backend: 404` | diga can't reach `dugu_url` from the VM (see step 9) |
| "Unable to connect" in the browser after a redirect | Redirect URL has a different port or host than the address bar |
| bivent restart loop, `EConfigDBPingFailed` | Postgres container down, or wrong port/credentials in `bivent.yaml` |
| dugu crash loop | Check `journalctl -u dugu -n 30`: YAML indent, missing `bfile`/`otp` paths |
| Logs to follow | `tail -F /var/log/{diga/diga.log,dugu/dugu.log,nginx/arma_bfit_access.log,nginx/arma_bfit_error.log}` |

## Dead ends you can skip

- The Ubuntu Docker guide, and the `bfi-repo.gpg` key conversion.
- `:8080` tunnels, `absolute_redirect off`, and the `127.0.11.11` / `127.0.0.14` IP ideas.
- Leftover `arma.test`, `127.0.2.2` and `arma-test` values from the laptop setup. Find them with `grep -rn "arma.test\|127.0.2.2" /etc /opt`.
----------------
FLOW 
### Siapa yang menerima pertama

**1. Browser memutuskan tujuan.** Kamu mengetik `http://arma.tora`. Browser mencari IP-nya di `/etc/hosts` laptop, dapat `127.0.0.1`, lalu connect ke `127.0.0.1:80` di laptop sendiri.

**2. Penerima pertama di laptop: `ssh -N -L 127.0.0.1:80:localhost:80`.** Proses ssh ini mendengarkan port 80 laptop. Tugasnya hanya membungkus koneksi itu dan mengirimnya lewat koneksi SSH terenkripsi ke VM. `sshd` di VM lalu membuka koneksi baru ke `localhost:80` di VM.

Ini titik yang tadi bermasalah. Kalau nginx laptop yang memegang port 80, browser berhenti di nginx laptop dan terowongan tidak pernah dipakai.

**3. Penerima pertama di VM: nginx port 80.** Ia membaca header `Host: arma.tora`, mencocokkannya dengan `server_name arma.tora`, lalu memilih route berdasarkan path. nginx tidak memutuskan siapa boleh masuk. Ia bertanya ke diga.

### Alur login langkah demi langkah

|#|Yang terjadi|Siapa|
|---|---|---|
|1|Browser minta `/`. nginx menjalankan `auth_request /diga-auth`, yaitu pertanyaan internal ke diga: “orang ini sudah login?”|nginx → diga `/auth`|
|2|diga mencari cookie `arma-cookie`, tidak ada, jadi jawab 401. Log: `Token not found.`|diga|
|3|`error_page 401` di nginx mengubahnya jadi redirect `302 → /diga/acc/landing`|nginx|
|4|`/diga/acc/landing` diteruskan nginx ke diga (`/acc/landing`). diga menampilkan halaman “sending you to login”, yang pindah otomatis setelah 3 detik|diga|
|5|Browser minta `/diga/acc/login`. diga tidak langsung menampilkan form: ia **sendiri** memanggil dugu lewat `dugu_url` (`POST /init`, membawa `client_id` dan `client_secret`). dugu membalas dengan ID sesi|diga → dugu|
|6|diga membalas browser dengan redirect ke `http://arma.tora/dugu/sess/<id>`|diga|
|7|Browser minta `/dugu/sess/<id>`. nginx meneruskan `/dugu/` ke dugu. dugu menampilkan form login dan `style.css`|nginx → dugu|
|8|Kamu mengisi `tora@arma.tora` dan password. dugu memakai backend `htaccess`: membaca `bfile.yaml`, yang menunjuk ke `/etc/nginx/_dugu-arma-htaccess`, lalu mencocokkan hash password-nya|dugu|
|9|Kalau cocok, dugu redirect ke `redirect_uri`, yaitu `/diga/acc/login/ok?code=...`|dugu|
|10|diga menerima kode itu, memverifikasinya ke dugu (kemungkinan lewat `dugu_url` lagi, ini perkiraanku), memasang cookie `arma-cookie` di browser, lalu redirect ke `redirect_ok` (`http://arma.tora`)|diga|
|11|Browser minta `/` lagi, sekarang membawa cookie. `auth_request` ke diga → 200, dan diga menyertakan username di header. nginx menyajikan frontend dari `/var/www/vhosts/arma.tora/arma/` dan menyisipkan `<script src=w/script.js>`|nginx + diga|
|12|Aplikasi di browser memanggil `/api/...`. nginx bertanya ke diga lagi, lalu menambahkan header `X-Username` dan `X-Server-Key`, menulis ulang `/api/x` menjadi `/x`, dan meneruskannya ke bivent|nginx → bivent|
|13|bivent memeriksa `X-Server-Key` sama dengan `header_server_val`, memeriksa username ada di `users.json`, lalu query ke Postgres (Docker, `127.0.0.1:15432`) dan mengembalikan JSON|bivent → Postgres|

### Siapa tahu apa

|Komponen|Tugasnya|Tidak tahu apa-apa soal|
|---|---|---|
|**nginx**|Pintu masuk dan router. Mengarahkan path, menyajikan file statis, menanyakan izin ke diga|Isi password dan siapa user yang sah|
|**diga**|Penjaga: mengecek cookie dan menerbitkannya setelah login|Password pengguna|
|**dugu**|Pengecek identitas: memegang form login dan mencocokkan password|Cookie dan isi aplikasi|
|**bivent**|Aplikasi dan datanya. Percaya header yang dikirim nginx|Cara login berlangsung|
|**Postgres**|Menyimpan data|Semua yang lain|

Karena bivent hanya percaya header dari nginx, `X-Server-Key` ada untuk memastikan request itu memang lewat nginx dan bukan orang yang langsung menembak port 33891.

makanya bisa masuk karena di ambil dari cookies jadi intinya bisa masuk tuh  karena ambil cookies dari diga dugu cookies, jadi intinya nanti si bivent nge sync nama cookies nya sama yang di kasih dugu dan diga oleh karena itu bisa login, dan juga kenapa di tulis rewrite karena biar bisa di access oleh diga dan dugu ini kalau ga di access ya nnti hanya nama domain nya doang gada yang di lempar

oleh karena itu ada client secret sama secret id itu tuh pengenal biar di diga dan dugu sync dan tau mereka akan tandem 