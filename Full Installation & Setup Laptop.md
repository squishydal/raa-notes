## Command berguna

List semua file di sebuah direktori (termasuk hidden, urut waktu):

```bash
ls -alt
```

## Package yang harus diinstall (laptop/server)

- tree
- git
- screen
- tig
- rsync
- curl
- netcat-openbsd
- vim-nox
- tmux
- upx-ucl (dipakai `make dist` untuk compress binary)
- bzip2 (dibutuhkan installer dotfiles-mini)

```bash
sudo apt install -y tree git screen tig rsync curl netcat-openbsd \
    vim-nox tmux upx-ucl bzip2
```

## Installing dotfiles

```bash
curl -fsSL https://bfi.io/dl/dotfiles-mini/install.sh | bash
```

Dotfiles yang terpasang: `tmux.conf`, `bashrc`, `.vimrc`.

> Error `bzip2` not found → `sudo apt install bzip2`, lalu jalankan ulang.

## Installing Docker

Ikuti docs di website Docker untuk instalasinya.

Tambahkan `$USER` ke group docker supaya tidak perlu `sudo` setiap menjalankan command docker:

```bash
sudo usermod -aG docker $USER
```

> TODO: muncul warning saat login, credentials tersimpan tanpa enkripsi di `/home/raa/.docker/config.json`.
> Configure credential helper supaya warning hilang: https://docs.docker.com/go/credential-store/

## Installing Go

1. Download dari website resmi (`go.dev/dl`).
2. Untar:
   ```bash
   tar -zxf go1.27.3.linux-amd64.tar.gz    # hasilnya folder `go`
   ```
3. Pindahkan ke `/opt` dan namai sesuai versinya:
   ```bash
   sudo mv go /opt/go-1.27.3
   ```
4. Symlink `go` di folder `/opt` yang sama. `-sfn` supaya aman dipakai ulang saat upgrade versi:
   ```bash
   cd /opt && sudo ln -sfn go-1.27 go
   ```
5. Buat folder `_gopath` di Devel:
   ```bash
   mkdir -p ~/Devel/_gopath
   ```
6. Configure di `~/.bashrc`:
   ```bash
   export GOROOT=/opt/go
   export GOPATH=$HOME/Devel/_gopath
   export PATH=$PATH:/opt/go/bin
   ```
7. Cek: `go version`.

> Intinya kita override default setting. `GOROOT` dicari di `/opt/go` (symlink tadi). `GOPATH` bikin dependency yang di-download terinstall di lokasi yang kita tentukan. `PATH` bikin shell menemukan binary `go` di `/opt/go/bin`.

## Semia

Project initializer in-house, punya linter + code convention sendiri.

> **CX**: semua rule berlaku, project harus di-initialize pakai Semia.
> **RX**: tanpa rule, WILDWEST.

1. Clone repository:
   ```bash
   git clone git@cx.bfi.io:workflow/semia.git
   ```
2. Baca `README.md`, banyak hint tentang cara kerja program.
3. Build:
   ```bash
   make
   ```
4. Bisa dipanggil manual (`./semia -flag`). Best practice: pindahkan `semia` ke `$HOME/bin` dan tambahkan ke PATH di `.bashrc` / `.zshrc`:
   ```bash
   export PATH=$PATH:$HOME/bin
   ```
   Lalu test dengan menjalankan `semia`.

Reference: [[03-NOV-2026#Semia Introduced]]

### Aturan `bfi-revive` (hook saat `git push` ke CX)

Dari error push di [[04-NOV-2026]]:

- Cyclomatic complexity per fungsi maksimal **10**.
- Panjang baris maksimal **72 karakter**, tab dihitung **4**.
- Kode harus tetap kompatibel dengan Go versi minimal di `go.mod` (stub: 1.13).
- Hook berhenti di file pertama yang gagal, jadi file lain bisa punya masalah yang belum kelihatan. Jalankan `make lint` sebelum push.
- Install linter dari `go.bfi.io/sysop/golang-codeconv` (lihat komentar di target `lint` pada Makefile).

## Scratch: contoh snippet

Config `.yaml` program profer:

```yaml
secretKey: <some_secret>
useraccount: <some_user>
```

```js
console.log("hello world")
```

```python
def helloWorld():
	return "Hello World!"
```
