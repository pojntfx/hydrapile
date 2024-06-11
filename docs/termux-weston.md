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
proot-distro install fedora
proot-distro login fedora --shared-tmp

dnf install -y https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

dnf install -y gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugins-bad-free gstreamer1-plugins-ugly-free gstreamer1-libav dbus-daemon weston pipewire pipewire-gstreamer pipewire-utils xorg-x11-server-Xwayland glx-utils

export XDG_RUNTIME_DIR="/tmp" XDG_CURRENT_DESKTOP=weston XDG_CONFIG_HOME="$HOME/.config" GDK_BACKEND=wayland

echo '[autolaunch]
path=/usr/bin/glxgears
watch=true
' > ./weston.ini

echo '#!/bin/bash

pipewire &
sleep 1

pipewire-pulse &
sleep 1

wireplumber &
sleep 1

bash -l
' > ./startup.sh
chmod +x ./startup.sh

dbus-run-session ./startup.sh # Wait until we get the nested bash, then continue

mkdir -p /tmp/.X11-unix/
weston --xwayland --shell desktop --renderer=pixman --backend=pipewire --width 720 --height 1440 -c ${PWD}/weston.ini &

pw-cli list-objects Node # Pick the ID of the weston node, i.e. 42, and use it `path` below

gst-launch-1.0 pipewiresrc path=42 ! videoconvert ! video/x-raw,format=I420 ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! jpegenc ! multipartmux ! queue ! tcpserversink host=0.0.0.0 port=5000
# Or any other pipeline from termux-x11-gstreamer-ffmpeg-fedora.md, just be sure to use the `pipewiresrc` instead of `ximagesrc`
```

Now run any matching decoding pipeline from [./termux-x11-gstreamer-ffmpeg-fedora.md](termux-x11-gstreamer-ffmpeg-fedora.md)!

If you're using the raw format, use `I420/YUV420` like so: `ffplay -f rawvideo -pixel_format yuv420p -video_size 720x1440 -framerate 60 -i $TMPDIR/mydisplay.pipe -flags low_delay -fflags nobuffer -analyzeduration 0 -probesize 32`
