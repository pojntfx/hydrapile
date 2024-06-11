# Termux Wayland-based GStreamer and `ffmpeg` Experiments (Fedora)

SSH:

```shell
# In Termux
mkdir -p ~/.ssh
curl 'https://github.com/pojntfx.keys' > ~/.ssh/authorized_keys
sshd
```

Hardware acceleration with `virglrenderer-android` (and optionally `virglrenderer-mesa-zink` for hardware-accelerated decoding):

```shell
ssh -p 8022 u0_a395@192.168.1.73

pkg install virglrenderer-android
virgl_test_server_android &

pkg install mesa-zink virglrenderer-mesa-zink vulkan-loader-android

virgl_test_server --use-egl-surfaceless --use-gles &
```

```shell
dnf install -y https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

dnf install -y python3-dbus python3-gobject gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugins-bad-free gstreamer1-plugins-ugly-free gstreamer1-libav labwc xorg-x11-server-Xwayland dbus-daemon xdg-desktop-portal-wlr python3-pydbus

export XDG_RUNTIME_DIR="/tmp" WLR_BACKENDS=headless XDG_CURRENT_DESKTOP=sway XDG_CONFIG_HOME="$HOME/.config" WLR_LIBINPUT_NO_DEVICES=1 WLR_RENDERER=gles2 # This doesn't work with WLR_RENDERER=pixman because of https://github.com/emersion/xdg-desktop-portal-wlr/issues/289 and we can't use `llvmpipe` etc. because of https://gitlab.freedesktop.org/wlroots/wlroots/-/issues/2871

mkdir -p "${XDG_CONFIG_HOME}/xdg-desktop-portal-wlr"

echo '[screencast]
chooser_type=none' > "${XDG_CONFIG_HOME}/xdg-desktop-portal-wlr/config"

echo '#!/bin/bash

pipewire &
sleep 1

pipewire-pulse &
sleep 1

wireplumber &
sleep 1

/usr/libexec/xdg-desktop-portal-wlr --replace --loglevel DEBUG &
sleep 1

/usr/libexec/xdg-desktop-portal --replace --verbose &
sleep 1

./server.py # From https://gist.github.com/pojntfx/c3bbbf6dbf3759e75607c5e085ab8740
' > ./startup.sh
chmod +x ./startup.sh

dbus-run-session labwc -S ./startup.sh
```

```shell
gst-launch-1.0 tcpclientsrc host=localhost port=5000 ! multipartdemux ! jpegdec ! videoconvert ! autovideosink
```
