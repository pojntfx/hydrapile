# Android Experiments

```shell
pkg install virglrenderer-android
virgl_test_server_android &
# Or:
pkg install tur-repo
pkg update -y && pkg upgrade -y
pkg install mesa-zink virglrenderer-mesa-zink vulkan-loader-android
MESA_LOADER_DRIVER_OVERRIDE=zink GALLIUM_DRIVER=zink ZINK_DESCRIPTORS=lazy virgl_test_server --use-egl-surfaceless &
# Or:
MESA_NO_ERROR=1 MESA_GL_VERSION_OVERRIDE=4.3COMPAT MESA_GLES_VERSION_OVERRIDE=3.2 GALLIUM_DRIVER=zink ZINK_DESCRIPTORS=lazy virgl_test_server --use-egl-surfaceless --use-gles &

export DISPLAY=:0
termux-x11 :0 &

proot-distro login debian --shared-tmp
export DISPLAY=:0

export GALLIUM_DRIVER=llvmpipe MESA_GL_VERSION_OVERRIDE=4.0
# Or:
export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0
# Or:
export GALLIUM_DRIVER=zink MESA_GL_VERSION_OVERRIDE=4.0

# Or:
curl -Lo /tmp/mesa-vulkan-kgsl_23.3.0-devel-20230905_arm64.deb 'https://drive.usercontent.google.com/download?id=1f4pLvjDFcBPhViXGIFoRE3Xc8HWoiqG-&export=download&authuser=0&confirm=t&uuid=c2b2b7eb-d605-4cc4-9b31-f31e8e707b84&at=APZUnTWPLawRsSalANrXZGy6qUZP%3A1716936199249'
dpkg -i /tmp/mesa-vulkan-kgsl_23.3.0-devel-20230905_arm64.deb
export MESA_LOADER_DRIVER_OVERRIDE=zink TU_DEBUG=noconform

export GDK_SCALE=3
gtk4-demo

export GDK_SCALE=3
adwaita-1-demo

glxgears

echo -e '[autolaunch]\npath=/usr/bin/adwaita-1-demo' >/etc/xdg/weston/weston.ini
export XDG_RUNTIME_DIR=/tmp
weston --width 1440 --height 2891
```
