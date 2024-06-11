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

> The pipeline never rolls unless there is any movement in weston with an XWayland client for some reason

```shell
dnf install -y https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

dnf install -y gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugins-bad-free gstreamer1-plugins-ugly-free gstreamer1-libav dbus-daemon weston pipewire pipewire-gstreamer pipewire-utils xorg-x11-server-Xwayland glx-utils

export PIPEWIRE_RUNTIME_DIR="/pipewire-0"
export PULSE_RUNTIME_PATH="/pulse"
export DISABLE_RTKIT=y

export XDG_RUNTIME_DIR="/tmp" XDG_CURRENT_DESKTOP=weston XDG_CONFIG_HOME="$HOME/.config" GDK_BACKEND=wayland

echo '[autolaunch]
path=/usr/bin/glxgears
watch=true
' > ./weston.ini

mkdir -p /run/dbus
dbus-daemon --system --fork

export DBUS_SESSION_BUS_ADDRESS="unix:path=/run/dbus/system_bus_socket"

mkdir -p /pipewire-0
pipewire &

mkdir -p /pulse
pipewire-pulse &

wireplumber &

mkdir -p /tmp/.X11-unix/
weston --xwayland --shell desktop --renderer=pixman --backend=pipewire --width 800 --height 480 -c ${PWD}/weston.ini &

pw-cli list-objects Node

gst-launch-1.0 pipewiresrc path=33 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! jpegenc ! multipartmux ! queue ! tcpserversink host=0.0.0.0 port=5000
```

```shell
gst-launch-1.0 tcpclientsrc host=localhost port=5000 ! multipartdemux ! jpegdec ! videoconvert ! autovideosink
```
