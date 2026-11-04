untuk menjalankan command me-list semua file di sebuah direktori, gunakan `ls -alt`. ls 

untuk konfigurasi .yaml program profer, bisa gunakan sample di bawah ini

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


## Installing dotfiles

` curl -fsSL https://bfi.io/dl/dotfiles-mini/install.sh | bash`

ada error `bzip2` not found. berarti, run `apt install bzip2`

The dotfile for `tmux.conf`, `bashrc`, `.vimrc`

## Installing Docker

Follow the docs on the docker website for installing docker 
Add $USER to docker group with 

`sudo usermod -aG docker $USER`

> so didn't need to use sudo everytime run a docker related command

> TODO 
 WARNING! Your credentials are stored unencrypted in '/home/raa/.docker/config.json'.
 Configure a credential helper to remove this warning. See
 https://docs.docker.com/go/credential-store/



## Installing Go

1. Install go dari official website nya (`go.dev/dl`)
2. Untar file nya lalu namai sesuai version nya 
`tar -zxf go1.27.3.linux-amd64.tar.gz`
3. mv go ke /opt/go-(version)
`mv go /opt/go-(version)`
4. Symlink go-(version) ke go di folder yang sama `/opt`
`ln -s /opt/go-(version) go`
5. Buat dir dengan nama "\_gopath" di Devel 
6. Configure golang at the bashrc
	1. Export GOROOT=/opt/go
	2. export GOPATH=$HOME/Devel/_gopath
	3. export PATH=$PATH:/opt/go/bin
	
> Jadi intinya kita nge Override default setting agar GOROOT Nyari nya di opt/go tempat yang udah di setting tadi, Sama juga dengan GOPATH biar si Dependancy yang nanti di download akan terinstall di lokasi yang sudah di setting tadi. Kalau PATH berarti nanti Computer/filesystem nyari nya di lokasi yang telah di declarasikan

## Yang harus di install Laptop/server 
- tree
- git
- screen
- tig
- rsync
- curl
- netcat-openbsd
- vim-nox
- tmux
- upx-ucl

# Semia
In house project initializer that have it's own linter + code convention 

> CX : all rule, Need semia to initialize project 
> RX : No Rule, WILDWEST

1. Git clone the repository using 
`git clone git@cx.bfi.io:workflow/semia.git`
2. Read the README.md there's a lot of hint about how the program works 
3. make using 
`make`
4. You can call it manually using `./semia -flag` or ideally and best practices you can move semia somewhere and add that file to the path, If place semia at `$HOME/bin` you can add this line at your bashrc or zsh `PATH=$PATH:$HOME/bin` and then test it by running semia 
reference([[03-NOV-2026#Semia Introduced]]) 
