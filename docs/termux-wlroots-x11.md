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

Headless X11:

```shell
proot-distro install fedora
proot-distro login fedora --shared-tmp

dnf install -y xorg-x11-server-Xvfb

export DISPLAY=":1" XAUTHORITY="${TMPDIR}/Xauthority"

rm -f "$XAUTHORITY" && touch "$XAUTHORITY" && xauth add "$DISPLAY" . $(mcookie)

Xvfb "$DISPLAY" -auth "$XAUTHORITY" -screen 0 720x1440x24+32 &
```

```shell
dnf install -y https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

dnf install -y gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugins-bad-free gstreamer1-plugins-ugly-free gstreamer1-libav cage xorg-x11-server-Xwayland weston

export XDG_RUNTIME_DIR="/tmp" WLR_BACKENDS=x11 XDG_CURRENT_DESKTOP=sway XDG_CONFIG_HOME="$HOME/.config" WLR_LIBINPUT_NO_DEVICES=1 WLR_RENDERER=pixman

cage -- weston-terminal &

# And run X11-based GStreamer server/client pipelines etc. like in termux-x11-gstreamer-ffmpeg-fedora.md
```
