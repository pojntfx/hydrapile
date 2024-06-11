# Termux Wayland-based `wayvnc` Experiments (Fedora)

```shell
pkg install virglrenderer-android
virgl_test_server_android &

pkg install mesa-zink virglrenderer-mesa-zink vulkan-loader-android

virgl_test_server --use-egl-surfaceless --use-gles &
```

```shell
proot-distro install fedora
proot-distro login fedora --shared-tmp

dnf install -y cage wayvnc gtk4-devel-tools

export XDG_RUNTIME_DIR="/tmp" WLR_BACKENDS=headless XDG_CURRENT_DESKTOP=sway WLR_LIBINPUT_NO_DEVICES=1 WLR_RENDERER=pixman GDK_BACKEND=wayland

export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0 # For both `virglrenderer-android` and `virglrenderer-mesa-zink`; using just `virglrenderer-mesa-zink` breaks for `glxgears` (other non-GL applications are accelerated well though!)

echo '#!/bin/bash

wayvnc -f 60 0.0.0.0 5000 & # Also supports WebSockets with `-w`
gtk4-demo
' > ./startup.sh
chmod +x ./startup.sh

dbus-run-session cage -- ./startup.sh
```

```shell
vncviewer 127.0.0.1:5000
```
