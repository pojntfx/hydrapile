# Greenfield & `proot`

## Building `proot`

```shell
git clone git@github.com:termux/proot.git
git apply proot/diff.patch # From this repo, not `termux/proot.git`
cd proot/src
make -j$(nproc)
sudo install ./proot /usr/local/bin
```

## Building Greenfield

```shell
cd greenfield
docker build --platform linux/arm64 -t pojntfx/greenfield . # --platform linux/amd64 to build for x86_64
```

## Starting Greenfield

### Using Docker

```shell
# Without hardware acceleration
docker run -it -p 8080:8080 -p 8081:8081 pojntfx/greenfield

# With hardware acceleration
docker run -it --privileged -v /dev/dri:/dev/dri -p 8080:8080 -p 8081:8081 pojntfx/greenfield
```

### Using `proot`

```shell
docker export "$(docker create pojntfx/greenfield)" --output="/tmp/rootfs.tar" # This can take a while

rm -rf ~/.proot-container && mkdir -p ~/.proot-container
proot --link2symlink tar -xf /tmp/rootfs.tar -C ~/.proot-container # This can take a while; we need to execute this in `proot` or the hard links fail during extraction

echo -e 'nameserver 8.8.8.8\nnameserver 8.8.4.4' > ~/.proot-container/etc/resolv.conf
echo -e '127.0.0.1	localhost\n::1     localhost ip6-localhost ip6-loopback\nff02::1 ip6-allnodes\nff02::2 ip6-allrouters' > ~/.proot-container/etc/hosts

rm -rf ~/.proot-container/proc && mkdir -p ~/.proot-container/proc

unset LD_PRELOAD
proot --link2symlink --kill-on-exit --kernel-release=6.8.10 -b /dev -b /proc -b /sys -b /proc/self/fd:/dev/fd -b /proc/self/fd/0:/dev/stdin -b /proc/self/fd/1:/dev/stdout -b /proc/self/fd/2:/dev/stderr -b ~/.proot-container/proc/.stat:/proc/stat -b ~/.proot-container/proc/.version:/proc/version -b ~/.proot-container/proc/.loadavg:/proc/loadavg -b ~/.proot-container/proc/.vmstat:/proc/vmstat -b ~/.proot-container/proc/.uptime:/proc/uptime -r ~/.proot-container -0 -w /root -b ~/.proot-container/root:/dev/shm /bin/su -l user
```

```shell
# In proot:
export XDG_RUNTIME_DIR="/home/user/.xdg-runtime-dir"
export XAUTHORITY="/tmp/.X11-unix/Xauthority"

mkdir -p /tmp/.X11-unix && touch "$XAUTHORITY" && xauth add :1 . "$(xxd -l 16 -p /dev/urandom)" && supervisord -c /etc/supervisor/supervisord.conf

# On Termux we get:
# Can't open device path: /dev/dri/renderD128: Permission denied
# Can't initialize EGL, wl_dmabuf and wl_drm disabled.
# Which seems to be why it won't render.
```

## Accessing the Applications

Open [http://localhost:8080/](http://localhost:8080/) and enter:

```plaintext
# Console
rem://localhost:8081/kgx
# GKT4 Demo
rem://localhost:8081/gtk4-demo
# GKT4 Widget Factory
rem://localhost:8081/gtk4-widget-factory
# libadwaita Demo
rem://localhost:8081/adwaita-1-demo
# glxgears
rem://localhost:8081/glxgears
# weston-terminal
rem://localhost:8081/weston-terminal
```
