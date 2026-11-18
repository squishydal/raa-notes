# Ringkasan: Hook Lint di Gogs, SSH, dan Jenkins

Tanggal sesi: 17 November 2026.

## Tujuan

1. Setiap `git push` ke Gogs dicek otomatis. Kalau kodenya tidak lolos aturan lint, push **ditolak**. Berlaku untuk semua repo.
2. Push yang bersih memicu build di Jenkins lewat webhook.

## Gambaran Singkat

```
laptop ──ssh (port 52222)──▶ Gogs (rxgogs)
                               │ hook pre-receive jalan (bghook)
                               ├─ cek email committer dan ukuran file
                               └─ kirim kode ke hoverlint ──▶ dicek linter
                               ▼
                       push diterima / ditolak
                               │ (hanya kalau diterima)
                               └─ webhook ──▶ Jenkins (laptop, port 8080)
```

| Komponen | Fungsi |
|---|---|
| `rxgogs` | Gogs. Sudah ada `bghook` di dalamnya. |
| `rxpg` | Database Postgres untuk Gogs. |
| `hoverlint` | Server linter. Semua linter ada di sini. |
| `rxgogsnet` | Network Docker yang dipakai ketiga container. |
| Jenkins | Service di laptop, port 8080. nginx juga membukanya sebagai `jenkins.test`. |

`10.254.254.254` adalah alamat di laptop kamu. Docker membuka Gogs di alamat itu: web port 3000, SSH port 52222.

Lokasi file di host (dari history): project compose Gogs ada di `~/Devel/001-bficx/sysop/rxgogs/compose/gogs`, dan folder `gogsdata/` di dalamnya adalah isi `/data` di container (folder `git/`, `gogs/`, `ssh/` cocok dengan susunan `/data`). Jadi file ini bisa diedit dari host:

- `gogsdata/gogs/conf/app.ini` (buat backup dulu: `cp app.ini{,.bak}`)
- `gogsdata/git/.gitconfig`
- `gogsdata/git/git-templates/custom_hooks/pre-receive`

Setelah ubah `app.ini`, stack dijalankan ulang dari folder compose dengan `docker compose down` lalu `docker compose up`. Perubahan port dan `extra_hosts` memang butuh langkah ini.

## Yang Kita Lakukan

### 1. Jalankan hoverlint

- `docker compose up -d` dari compose file hoverlint, dengan network `rxgogsnet`.
- Cek dari Gogs: `curl http://hoverlint:19435/` menjawab `405`. Itu normal, karena hoverlint hanya menerima POST.
- Tes langsung dengan file Go yang jelek: hasilnya `gofmt ... failed` plus diff-nya, tanpa password.

### 2. Atur Gogs supaya pakai hoverlint

Semua perintah dijalankan sebagai user `git` (`docker exec -u git rxgogs ...`).

File `/data/git/.config/bfi-ghooks/lint.json` diisi supaya semua bahasa diarahkan ke hoverlint:

```json
{
  "urls": {
    "bash":   "http://hoverlint:19435/",
    "c":      "http://hoverlint:19435/",
    "cpp":    "http://hoverlint:19435/",
    "go":     "http://hoverlint:19435/",
    "java":   "http://hoverlint:19435/",
    "js":     "http://hoverlint:19435/",
    "php":    "http://hoverlint:19435/",
    "python": "http://hoverlint:19435/",
    "kotlin": "http://hoverlint:19435/"
  }
}
```

Hati-hati: `bghookctl reset-lint -help` **tidak** menampilkan help. Perintah itu mereset file ini ke pengaturan lokal.

### 3. Hook untuk repo baru

Template ada di `/data/git/git-templates/custom_hooks/pre-receive`:

```bash
#!/bin/bash
/usr/local/bin/bghook pre-receive
```

`init.templatedir` di `/data/git/.gitconfig` menunjuk ke folder itu, jadi setiap repo baru otomatis dapat hook ini. Repo yang dibuat sebelum template harus diinstall dengan `bghookctl rewrite-all`.

Aturan bawaan: email committer harus berakhiran `bandungfe.net`, `bfinews.com`, atau `bfi.io`, dan ukuran file maksimal 2 MB.

### 4. Setup SSH

- Compose Gogs: `"10.254.254.254:52222:22"` dan `"10.254.254.254:3000:3000"`.
- `app.ini`: `DOMAIN = 10.254.254.254`, `EXTERNAL_URL = http://10.254.254.254:3000/`, `SSH_PORT = 52222`. `SSH_PORT` hanya mengatur port yang tampil di URL clone, jadi harus sama dengan port yang dibuka Docker.
- Setelah ubah compose file, jalankan `docker compose up -d`. `docker restart` saja tidak cukup.
- Tambahkan public key (`~/.ssh/id_ed25519.pub`) di Gogs: Settings → SSH Keys. Jangan pernah copy private key.
- `~/.ssh/config` di laptop:

```
Host gogs
    HostName 10.254.254.254
    Port 52222
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

- Tes: `nc -zv 10.254.254.254 52222`, lalu `ssh -T gogs`. Pesan "authenticated, but Gogs does not provide shell access" artinya berhasil.
- Kalau git masih minta username dan password, remote-nya masih HTTP. Ganti: `git remote set-url origin gogs:USER/REPO.git`.
- `~/.ssh/conf.d/bfi-systems.conf` diedit berkali-kali di sela push, dan `chmod 600` dijalankan tepat sebelum push mulai berhasil. OpenSSH menolak file config yang izinnya terlalu longgar, jadi simpan di `600`. Ini hanya dugaan dari history, karena pesan errornya tidak ada di sana.

## Hasil Tes untuk Go

| Kasus | Hasil |
|---|---|
| Push `README.md` | Diterima (hook jalan) |
| Push `bad.go` yang formatnya jelek | **Ditolak**: `gofmt ... failed` |
| hoverlint dimatikan, push file kode | **Ditolak**: `REMOTE LINTER: server down` |
| `gofmt -w`, `git commit --amend`, push | **Diterima** |
| File jelek kedua `bad2.go` | **Ditolak**: `gofmt ... failed` |

Catatan:

- `Gogs: Internal error` saat push ditolak itu normal. Itu cara Gogs menampilkan penolakan dari hook.
- Kalau lint gagal, perbaiki file, lalu `git commit --amend` atau `git reset --soft HEAD~1`, baru push lagi.
- Kalau hoverlint mati atau lebih dari 4 detik tidak menjawab, push file kode ditolak (aman, tidak ada kode yang lolos tanpa dicek). Cukup nyalakan hoverlint dan push lagi, commit tidak perlu diubah.
- Variabel yang tidak dipakai adalah error compiler, bukan aturan lint, jadi tidak ketahuan.

## Masalah Terbuka: Hanya Go yang Dicek

Push Python (`bad.py` di repo `test-hooks-py`) **diterima** tanpa ada hasil linter sama sekali. Tes langsung ke hoverlint untuk file Python, shell, JS, PHP, dan C semuanya menjawab `null`. Hanya Go yang ada hasilnya.

Yang sudah dicek:

- `DisablePython` bernilai `false`.
- `lint.json` milik hoverlint lengkap: semua path linter terisi.
- Semua linter ada di image. Kalau dijalankan manual di dalam hoverlint, `bfi-flake8`, `bfi-pycodestyle3`, dan `shellcheck` melaporkan masalah dengan benar.
- `repo.json` di container punya `DisableTS`, yang tidak ada di source lokal `bfi-ghooks`. Artinya image dibuat dari source yang berbeda dengan checkout kamu.

Penyebabnya belum ketemu. Langkah berikutnya:

1. Baca source server di folder `~/Devel/001-bficx/workflow/bfi-ghooks`: `cmd/bgrun/connector.go` dan `lint_python.go`. Jalankan dari folder itu, bukan dari repo tes.
2. Cek `git log -3` dan `git status` di sana, apakah checkout sama dengan source yang membuat image.
3. Ganti `GHOOKS_LOGLEVEL` di compose hoverlint dengan level yang valid (`debug` tidak valid, log penuh `invalid log level`). Coba `trace`, lalu lihat `hostdata/bghook.log`.
4. Kalau image dibuat dari source lain, build ulang dari source terbaru.

Sampai ini selesai, hanya Go yang terlindungi. Bahasa lain lolos tanpa dicek.

## Hubungkan Gogs ke Jenkins

### Jaringan

- Jenkins di laptop, port 8080, semua interface. nginx di port 80: `jenkins.test` diteruskan ke `127.0.0.1:8080`.
- Di dalam container Gogs, `127.0.0.1` adalah container itu sendiri, dan `jenkins.test` tidak dikenali. Container bisa mencapai Jenkins lewat `10.254.254.254`:

```sh
docker exec rxgogs curl -sI --max-time 5 http://10.254.254.254:8080/login | head -1
# HTTP/1.1 200 OK
```

Jadi webhook memakai alamat langsung. Jalur lewat nginx juga bisa, tapi butuh `extra_hosts: ["jenkins.test:10.254.254.254"]` di service `rxgogs` dan `docker compose up -d`.

- URL di payload webhook (`html_url`, `clone_url`) memang `10.254.254.254` karena diambil dari `EXTERNAL_URL`. Tidak perlu diubah: plugin ini hanya memakai token untuk memulai scan, dan scan melakukan clone dari Branch Source URL di job Jenkins.

### Webhook

- Plugin: **Multibranch Scan Webhook Trigger**. Di job multibranch aktifkan **Scan by webhook** dan isi token (misalnya `openssl rand -hex 16`).
- Di Gogs: Repository → Settings → Webhooks, tipe Gogs, content type `application/json`, event push.
- Payload URL (Jenkins ada di **8080**):

```
http://10.254.254.254:8080/multibranch-webhook-trigger/invoke?token=TOKEN_BARU
```

- Kesalahan yang terjadi: URL sempat memakai port **3000**, itu Gogs sendiri. Gogs tidak punya path itu, jadi menampilkan halaman 404 dan gagal merendernya (`template: base/footer ... PageStartTime: invalid value`). Itu hanya akibat. Perbaikan: pakai port 8080.
- Kalau ada pesan "address not allowed", tambahkan `LOCAL_NETWORK_ALLOWLIST = 10.254.254.254` di bagian `[security]` `app.ini`, lalu `docker restart rxgogs` (belum pasti didukung versi Gogs kamu).
- Cek dengan Test Delivery, lalu Recent Deliveries → tab Response. Status `200` berarti berhasil.
- Token lama sudah terlanjur muncul di chat, jadi buat token baru dan ganti di Gogs.

### Jenkins clone lewat SSH

Jenkins berjalan sebagai user `jenkins`, jadi butuh key sendiri:

1. `ssh-keygen -t ed25519 -C "jenkins-gogs" -f ~/.ssh/jenkins_gogs -N ""`
2. Di Gogs, repo `tora/repositori-golang` → Settings → Deploy Keys → tambah `jenkins_gogs.pub`, read-only.
3. Tes: `ssh -i ~/.ssh/jenkins_gogs -o IdentitiesOnly=yes -p 52222 -T git@10.254.254.254`.
4. Di Jenkins, Credentials → **SSH Username with private key**, ID `gogs-ssh`, username `git`, isi dengan private key `~/.ssh/jenkins_gogs`.
5. Percayai host key: Manage Jenkins → Security → Git Host Key Verification Configuration → "Accept first connected key".
6. Di job → Branch Sources → Git: `ssh://git@10.254.254.254:52222/tora/repositori-golang.git`, credentials `gogs-ssh`. Repo harus punya `Jenkinsfile` di root.

Webhook hanya memicu scan. Key SSH-lah yang membuat clone bisa jalan.

### Instal Jenkins dan prasyaratnya

Perintah dari history (paket Debian/Ubuntu):

```sh
java -version                         # Java harus sudah terpasang
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword    # password login pertama
```

Lalu `sudo systemctl status jenkins` harus menunjukkan service berjalan.

Untuk pipeline yang menjalankan Docker (di history ada `sudo usermod -aG docker jenkins`), user `jenkins` harus ada di group `docker`, dan Jenkins harus di-restart (`sudo systemctl restart jenkins`) supaya group baru berlaku. Group `docker` setara akses root di mesin itu. Di laptop dev tidak masalah, tapi pikir dua kali di server bersama.

Project tes: `tora/repositori-golang` dibuat dengan tool scaffold internal (`semia init -stub stubgo -org tora -dir repositori-golang`), lalu di-push ke `ssh://git@10.254.254.254:52222/tora/repositori-golang.git`. `Jenkinsfile` ditambahkan di root repo. Commit yang mengubah `cmd/repositori_golang/main.go` dipakai untuk memicu build, dan tag `v1.0.0` juga di-push (kemungkinan untuk tes build tag, history tidak menyebutkan).

### nginx untuk `jenkins.test`

Dari history, `jenkins.test` ditambahkan seperti ini:

```sh
cd /etc/nginx/sites-available
sudo cp pikachu.test jenkins.test          # mulai dari file site yang sudah ada, lalu edit
sudo vim jenkins.test                      # server_name jenkins.test; proxy_pass http://127.0.0.1:8080;
sudo ln -s `pwd`/jenkins.test /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl reload nginx
sudo vim /etc/hosts                        # arahkan jenkins.test ke 127.0.0.1
```

Selalu jalankan `sudo nginx -t` sebelum reload. History juga menunjukkan `pikachu.test` dan `arma.test` di-`unlink` dari `sites-enabled`, artinya dua site itu nonaktif (file-nya tetap ada di `sites-available`). Site yang aktif sekarang `jenkins.test` dan `raichu.test`, cocok dengan output `nginx -T` tadi. Untuk mengaktifkan lagi: `sudo ln -s /etc/nginx/sites-available/NAMA /etc/nginx/sites-enabled/`.

### Post-build processing (`bfi-jenkins-postproc`)

History menunjukkan dua tahap:

1. Awalnya dibuat script manual `/usr/local/bin/jenkins-postbuild-failure` dan `/usr/local/bin/jenkins-postproc` (dengan `chmod +x`).
2. Lalu diganti dengan paket internal: clone `git@cx.bfi.io:sysop/jenkins-postproc.git`, build dengan `make deb`, lalu `sudo dpkg -i dist/bfi-jenkins-postproc_1.0.0_all.deb`. Script lama di `/usr/local/bin` dihapus.

Paket ini menyimpan konfigurasi di `/etc/bfi-jenkins-postproc/`:

- `postproc/` dan `postbuild-failure/`, masing-masing berisi satu script per repo dengan nama `<org>--<repo>`, contoh `tora--repositori-golang`.
- Setiap script harus executable (`sudo chmod +x`), dan ini sempat harus diperbaiki di history.

History tidak menunjukkan isi script-nya, jadi isinya tidak dijelaskan di sini. Commit "Testing salah" dan "Test postproc" kemungkinan dipakai untuk mengecek jalur gagal dan post-processing.

## Kesalahan dan Pelajaran

- `git clone` dengan URL placeholder gagal, tapi baris berikutnya tetap jalan, sehingga commit "test" berisi `bad.go` nyasar ke repo `bfi-ghooks`. Hapus dengan `git log -1` lalu `git reset --hard HEAD~1`.
- Perintah `sed`, `grep`, `cat` untuk membaca source harus dijalankan di folder `bfi-ghooks`, bukan di repo tes.
- `docker exec` berjalan sebagai root secara default, jadi pakai `-u git` untuk apa pun yang menulis file Gogs.
- Port: Gogs web 3000, SSH 52222, Jenkins 8080, hoverlint 19435.
- Opsi `docker exec` ditulis sebelum nama container: `docker exec -it rxgogs /bin/bash`. Di history sempat `docker exec rxgogs -it bin/bash` yang gagal (urutan salah, dan `bin/bash` butuh `/` di depan).

## Yang Masih Harus Dikerjakan

- Cari tahu kenapa bahasa selain Go menjawab `null`, lalu tes ulang Python, shell, JS, PHP, dan C.
- Pastikan webhook di port 8080 menjawab `200` dan build jalan setelah push yang bersih. History (beberapa commit tes "webhook", lalu tes post-build gagal) menunjukkan build kemungkinan sudah jalan, tapi respons delivery-nya belum terlihat. Ganti token lama.
- Selesaikan setup deploy key SSH dan host key di Jenkins.
- Siapkan cara darurat untuk melewati hook, karena hook berlaku untuk semua orang termasuk admin (misalnya mematikan satu bahasa di satu repo dengan `bghookctl set`).
- Putuskan pakai `jenkins.test` atau alamat IP di URL webhook, dan konsisten.
