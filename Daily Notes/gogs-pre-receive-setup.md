# Gogs pre-receive lint hook, SSH and Jenkins: setup recap

Session date: 2026-11-17.

Goal: reject any push to any repository on Gogs when the pushed code fails the lint rules from `bfi-ghooks`, then connect Gogs to Jenkins so clean pushes trigger builds.

## 1. Architecture

```
laptop ──ssh :52222──▶ rxgogs (Gogs, Docker)
                          │  per-repo custom_hooks/pre-receive
                          │  └─ /usr/local/bin/bghook pre-receive
                          │        ├─ checks committer email and file size locally
                          │        └─ POST file content ──▶ hoverlint:19435 (linters)
                          ▼
                 push accepted / rejected
                          │ (accepted pushes only)
                          └─ webhook ──▶ Jenkins (laptop, :8080)
```

| Component | Role |
|---|---|
| `rxgogs` (`dreg.bfi.io/rxgogs:latest`) | Gogs. Runs as user `git`, `HOME=/data/git`. The image already contains `bghook`, `bghookctl`, `git` and `curl`. |
| `rxpg` (`postgres:12-alpine`) | Gogs database. Not changed. |
| `hoverlint` (`dreg.bfi.io/hoverlint:latest`) | Remote lint server (`/usr/bin/bghookcon`). Has all linters, so Gogs needs none. |
| `rxgogsnet` | External Docker network shared by the three containers. |
| Jenkins | systemd service on the laptop, `--httpPort=8080`, listens on all interfaces. nginx also serves it as `jenkins.test`. |

Addresses: `10.254.254.254` is an address on the laptop. Docker publishes Gogs on it: web `:3000`, SSH `:52222`. hoverlint is published on `:19435`.

## 2. hoverlint service

```yaml
services:
    hoverlint:
        image: dreg.bfi.io/hoverlint:latest
        container_name: hoverlint
        ports:
            - "${HOST}:${PORT}:19435"
        restart: unless-stopped
        env_file:
            - ./.env
        environment:
            - UID=1000
            - GHOOKS_LOGFILE=/hostdata/bghook.log
        volumes:
            - ./hostdata:/hostdata
        logging:
            options:
                max-file: "5"
                max-size: "10m"
        networks:
            - rxgogsnet

networks:
    rxgogsnet:
        external: true
```

Start with `docker compose up -d`. Notes:

- `./hostdata` must be writable by UID 1000.
- If only Gogs talks to hoverlint, the `ports:` section can be removed.
- `GHOOKS_LOGLEVEL=debug` is set inside the image, but `debug` is not a valid level: the log fills with `invalid log level, fall back to ERROR`, so no lint details are logged. See section 8.

Network checks:

```sh
docker network inspect rxgogsnet --format '{{range .Containers}}{{.Name}} {{end}}'
# expected: hoverlint rxpg rxgogs

docker exec rxgogs sh -c 'which git curl; curl -sv --max-time 5 http://hoverlint:19435/ -o /dev/null'
# HTTP 405 with "Allow: OPTIONS, POST" is the expected answer
```

If `rxgogs` is not on the network, run `docker network connect rxgogsnet rxgogs`, and add the network to the Gogs compose file so a recreate does not lose it.

Direct hoverlint test from the Gogs container (Go):

```sh
docker exec -u git rxgogs sh -c '
printf "package main\nfunc  main( ){x:=1}\n" > /tmp/bad.go
curl -s --max-time 10 -X POST http://hoverlint:19435/ \
  --data-urlencode "fname=bad.go" \
  --data-urlencode "content@/tmp/bad.go" \
  --data-urlencode "rcdata={\"MaxSize\":2097152}"'
```

The answer is JSON with `bfi-revive ... ok` and `gofmt ... failed` plus a diff. No credentials are needed.

## 3. Gogs-side configuration

All commands run as the `git` user (`docker exec -u git rxgogs ...`), so files are not owned by root.

### 3.0 Where things live on the host

From the shell history: the Gogs compose project is in `~/Devel/001-bficx/sysop/rxgogs/compose/gogs`, and its `gogsdata/` folder holds the container's `/data` (the folders `git/`, `gogs/` and `ssh/` match the `/data` layout). So these files can be edited from the host:

- `gogsdata/gogs/conf/app.ini` (make a backup first: `cp app.ini{,.bak}`)
- `gogsdata/git/.gitconfig`
- `gogsdata/git/git-templates/custom_hooks/pre-receive`

After changing `app.ini`, the stack was restarted from the compose folder with `docker compose down` and then `docker compose up`. Port and `extra_hosts` changes need this recreate, as noted in section 4.

### 3.1 Lint config: `/data/git/.config/bfi-ghooks/lint.json`

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

`bghook` POSTs a form (`fname`, `content`, `rcdata`) to the URL of the matching language, with a 4-second timeout. If a language has no URL, it tries local linters, which the Gogs container does not have.

Warning: `bghookctl reset-lint -help` does **not** print help. It resets `lint.json` to local defaults.

### 3.2 Hook template for new repositories

- `/data/git/.gitconfig` has `init.templatedir = /data/git/git-templates`.
- `/data/git/git-templates/custom_hooks/pre-receive` is executable and contains:

```bash
#!/bin/bash
/usr/local/bin/bghook pre-receive
```

Gogs copies it into each new repository. Repositories that existed earlier need `bghookctl rewrite-all` (check `-help` first). At setup time there were none.

### 3.3 Default rules (`repo.json`)

- Committer email must end in `bandungfe.net`, `bfinews.com` or `bfi.io`.
- Maximum file size 2 MB (configurable 1 to 8 MB).
- All language linters enabled (`DisablePython` and the others are `false`).
- Per-repo settings: `bghookctl set` / `get` / `reset`. `reset-all` copies the global config to all repos.

## 4. SSH access

Server side:

- Gogs uses the container's OpenSSH (`START_SSH_SERVER = false`, `SSH_LISTEN_PORT = 22`).
- Compose port mapping: `"10.254.254.254:52222:22"` and `"10.254.254.254:3000:3000"`.
- `app.ini` values (checked): `DOMAIN = 10.254.254.254`, `EXTERNAL_URL = http://10.254.254.254:3000/`, `SSH_PORT = 52222`. `SSH_PORT` only controls the port shown in clone URLs, so it must equal the published port.
- After changing the compose file, run `docker compose up -d`. `docker restart` does not apply port changes. Files under `/data` survive a recreate.
- Check with `docker port rxgogs`.

Laptop side:

- Add `~/.ssh/id_ed25519.pub` in Gogs (Settings → SSH Keys). Never copy the private key.
- `~/.ssh/config`:

```
Host gogs
    HostName 10.254.254.254
    Port 52222
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

- Tests: `nc -zv 10.254.254.254 52222`, then `ssh -T gogs`. The message "authenticated, but Gogs does not provide shell access" means success.
- If git still asks for a username and password, the remote is HTTP. Run `git remote set-url origin gogs:USER/REPO.git`.
- `~/.ssh/conf.d/bfi-systems.conf` was edited many times between pushes, and a `chmod 600` on it came right before the pushes started working. OpenSSH refuses config files with loose permissions, so keep them at `600`. This is my reading of the history, because the error message itself is not in it.

## 5. Test results (Go)

| Case | Result |
|---|---|
| Push `README.md` (HTTP) | Accepted. Email and size checks ran, so the hook runs. |
| Push badly formatted `bad.go` | **Rejected**: `gofmt ... failed` with a diff. |
| hoverlint stopped, push a code file | **Rejected**: `REMOTE LINTER: server down`. The hook fails closed. |
| `gofmt -w`, `git commit --amend`, push | **Accepted**: `bfi-revive ... ok`, `gofmt ... ok`. |
| Second bad file `bad2.go` | **Rejected**: `gofmt ... failed`. `bfi-revive` passed. |

Notes:

- `Gogs: Internal error` in the output is how Gogs shows a hook rejection. It is expected.
- The hook demands a squash or soft-reset of the failing commit. Fix the file, then `git commit --amend --no-edit` (or `git reset --soft HEAD~1`), then push.
- An unused variable is a compiler error, not a lint rule, so `bfi-revive` and `gofmt` do not catch it.
- When hoverlint is down, no code push is accepted. Start it again and push again, the commit does not need changing. A response slower than 4 seconds causes the same rejection.

## 6. Open issue: only Go is linted

A Python push (`tora/test-hooks-py`, file `bad.py`) was **accepted** with no linter lines at all, only `check committer email` and `check size`. Direct calls to hoverlint for `bad.py`, `bad.sh`, `bad.js`, `bad.php` and `bad.c` all answered `null`. Only Go returns results.

What was checked:

- `DisablePython` is `false` in the repo's `repo.json`.
- hoverlint's own `/hostdata/.config/bfi-ghooks/lint.json` is complete: every `cmds` entry has a path, and all `urls` are `null`, which is correct for a server that lints locally.
- All linters exist in the hoverlint image. Run by hand inside it, `bfi-flake8`, `bfi-pycodestyle3` and `shellcheck` report real problems (exit code 1).
- The installed `repo.json` has `DisableTS`, which is not in the local `bfi-ghooks` checkout. The image was built from different source than the checkout.

Not yet found: why the server returns `null` for non-Go files. Next steps:

1. Read the server code in `~/Devel/001-bficx/workflow/bfi-ghooks/cmd/bgrun/connector.go`, and `lint_python.go`, in that folder (not in the test repos). Look at how it picks a language from `fname`, and what it returns for a language it does not handle.
2. Run `git log -3` and `git status` there, to confirm the checkout matches the code that built the image.
3. Set a valid `GHOOKS_LOGLEVEL` in the hoverlint compose file (`trace` is my guess, because the code has `Trace` calls), run `docker compose up -d`, repeat the curl test, and read `hostdata/bghook.log`.
4. If the image was built from different source, rebuild it from the current source.

Until this is solved, only Go is protected. Other languages pass unchecked.

## 7. Jenkins integration

### 7.1 Network facts

- Jenkins runs on the laptop (`--httpPort=8080`, all interfaces). nginx listens on `0.0.0.0:80`: `jenkins.test` proxies to `http://127.0.0.1:8080`, and `raichu.test` is another site on the same port.
- Inside the `rxgogs` container, `127.0.0.1` is the container itself, and `jenkins.test` does not resolve (the laptop's `/etc/hosts` is not used). The container can reach Jenkins at `10.254.254.254`:

```sh
docker exec rxgogs curl -sI --max-time 5 http://10.254.254.254:8080/login | head -1
# HTTP/1.1 200 OK
```

So the webhook uses the direct address. The nginx way also works, but it needs `extra_hosts: ["jenkins.test:10.254.254.254"]` in the `rxgogs` compose service, and a recreate with `docker compose up -d`. Do not call nginx by the IP address, because nginx picks the site by the `Host` header.

- The URLs in the webhook payload (`html_url`, `clone_url`, `compare_url`) come from `EXTERNAL_URL`, and `ssh_url` from `SSH_DOMAIN` and `SSH_PORT`. They show `10.254.254.254`, and nothing needs changing: the Multibranch Scan Webhook Trigger only uses the token to start a scan, and the scan clones from the job's own Branch Source URL.

### 7.2 Webhook

- Plugin: **Multibranch Scan Webhook Trigger**. In the multibranch job, enable Scan Multibranch Pipeline Triggers → **Scan by webhook** and set a token (for example `openssl rand -hex 16`).
- Gogs: Repository → Settings → Webhooks, type Gogs, content type `application/json`, event push.
- Payload URL (Jenkins is on **8080**):

```
http://10.254.254.254:8080/multibranch-webhook-trigger/invoke?token=NEWTOKEN
```

- Mistake made: the URL was first set to port **3000**, which is Gogs itself. Gogs has no such path, so it showed its 404 page and failed to draw it (`template: base/footer ... PageStartTime: invalid value`). That error is a result, not the real problem. Fix: use port 8080.
- If delivery says an address is not allowed, add `LOCAL_NETWORK_ALLOWLIST = 10.254.254.254` under `[security]` in `app.ini` and restart `rxgogs` (not confirmed for this Gogs version).
- Check with Test Delivery, then Recent Deliveries → Response. `200` means success, and the job's "Scan Multibranch Pipeline Log" shows the scan.
- The webhook token was posted in chat, so generate a new one and update the URL in Gogs.

### 7.3 Jenkins clones over SSH

Jenkins runs as the `jenkins` user, so it needs its own key:

1. `ssh-keygen -t ed25519 -C "jenkins-gogs" -f ~/.ssh/jenkins_gogs -N ""`
2. Gogs: repo `tora/repositori-golang` → Settings → Deploy Keys → add `jenkins_gogs.pub`, read-only.
3. Test: `ssh -i ~/.ssh/jenkins_gogs -o IdentitiesOnly=yes -p 52222 -T git@10.254.254.254`.
4. Jenkins: Credentials → add **SSH Username with private key**, ID `gogs-ssh`, username `git`, key from `~/.ssh/jenkins_gogs`.
5. Trust the host key: Manage Jenkins → Security → Git Host Key Verification Configuration → "Accept first connected key", or run `ssh-keyscan -p 52222 10.254.254.254` as the `jenkins` user into its `known_hosts`.
6. Job → Branch Sources → Git: `ssh://git@10.254.254.254:52222/tora/repositori-golang.git`, credentials `gogs-ssh`. A `Jenkinsfile` must exist in the repo root.

The webhook only triggers the scan. The SSH key is what allows the clone.

### 7.4 Jenkins installation and prerequisites

Commands from the history (Debian/Ubuntu package):

```sh
java -version                         # Java must already be installed
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword    # password for the first login
```

`sudo systemctl status jenkins` should then show the service running.

For pipelines that run Docker (the history shows `sudo usermod -aG docker jenkins`), the `jenkins` user must be in the `docker` group, and Jenkins must be restarted (`sudo systemctl restart jenkins`) before the new group applies. The `docker` group is effectively root access on that machine. That is fine on a dev laptop, but worth a second thought on a shared server.

Test project: `tora/repositori-golang` was created with the internal scaffold tool (`semia init -stub stubgo -org tora -dir repositori-golang`) and pushed to `ssh://git@10.254.254.254:52222/tora/repositori-golang.git`. A `Jenkinsfile` was added in the repo root. Commits that edit `cmd/repositori_golang/main.go` were used to trigger builds, and a `v1.0.0` tag was pushed (probably to test tag builds, the history does not say).

### 7.5 nginx for `jenkins.test`

From the history, `jenkins.test` was added like this:

```sh
cd /etc/nginx/sites-available
sudo cp pikachu.test jenkins.test          # start from an existing site file, then edit it
sudo vim jenkins.test                      # server_name jenkins.test; proxy_pass http://127.0.0.1:8080;
sudo ln -s `pwd`/jenkins.test /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl reload nginx
sudo vim /etc/hosts                        # map jenkins.test to 127.0.0.1
```

Always run `sudo nginx -t` before reloading. The history also shows `pikachu.test` and `arma.test` being unlinked from `sites-enabled`, which disables those two sites (their files stay in `sites-available`). The enabled sites are now `jenkins.test` and `raichu.test`, which matches the earlier `nginx -T` output. To enable a site again: `sudo ln -s /etc/nginx/sites-available/NAME /etc/nginx/sites-enabled/`.

### 7.6 Post-build processing (`bfi-jenkins-postproc`)

The history shows two stages:

1. Hand-made scripts `/usr/local/bin/jenkins-postbuild-failure` and `/usr/local/bin/jenkins-postproc` were created first (with `chmod +x`).
2. They were replaced by the internal package: clone `git@cx.bfi.io:sysop/jenkins-postproc.git`, build with `make deb`, then `sudo dpkg -i dist/bfi-jenkins-postproc_1.0.0_all.deb`. The old scripts in `/usr/local/bin` were removed.

The package keeps its configuration in `/etc/bfi-jenkins-postproc/`:

- `postproc/` and `postbuild-failure/`, each with one script per repository, named `<org>--<repo>`, for example `tora--repositori-golang`.
- Each script must be executable (`sudo chmod +x`), which had to be fixed in the history.

The history does not show what the scripts do, so this recap does not describe their content. The commits "Testing salah" and "Test postproc" were most likely used to check the failure path and the post-processing.

## 8. Mistakes and lessons

- A `git clone` with a placeholder URL failed, but the next lines in `&&` and `cd` did not run as expected, so a "test" commit with `bad.go` landed in the `bfi-ghooks` repo. Remove it with `git log -1` and `git reset --hard HEAD~1`.
- Commands such as `sed`, `grep` and `cat` that read source code must run in the `bfi-ghooks` folder, not in the test repos.
- `docker exec` runs as root by default, so use `-u git` for anything that writes Gogs files.
- Port numbers: Gogs web is 3000, SSH is 52222, Jenkins is 8080, hoverlint is 19435.
- `docker exec` options go before the container name: `docker exec -it rxgogs /bin/bash`. The history shows `docker exec rxgogs -it bin/bash` failing first (wrong order, and `bin/bash` also needs the leading slash).

## 9. To-do

- Find out why non-Go languages return `null` (section 6) and retest Python, shell, JS, PHP and C.
- Confirm the webhook returns `200` on port 8080 and that builds start after a clean push. The history (several "webhook" test commits, then post-build failure tests) suggests builds already run, but the delivery response has not been seen. Rotate the old token.
- Finish the Jenkins SSH deploy key and host key setup.
- Plan an emergency bypass: the hook blocks everyone, including admins (for example, disable one language in one repo with `bghookctl set`).
- Decide whether `jenkins.test` or the IP address is used in the webhook URL, and keep it consistent.
