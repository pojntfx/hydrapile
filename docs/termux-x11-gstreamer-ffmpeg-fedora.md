# Termux X11-based GStreamer and `ffmpeg` Experiments (Fedora)

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

Hardware-accelerated X11 application:

```shell
dnf install -y glx-utils mangohud

export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0 # For both `virglrenderer-android` and `virglrenderer-mesa-zink`; using just `virglrenderer-mesa-zink` breaks for `glxgears` (other non-GL applications are accelerated well though!)

mangohud glxgears -fullscreen &
```

Encoding in guest:

```shell
export XDG_RUNTIME_DIR="${TMPDIR}"

dnf install -y https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
dnf install -y https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

dnf install -y gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugins-bad-free gstreamer1-plugins-ugly-free gstreamer1-libav

# H264/UDP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! x264enc speed-preset=superfast tune=zerolatency ! video/x-h264,stream-format=byte-stream ! queue ! udpsink host=0.0.0.0 port=5000
# H264/TCP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! x264enc speed-preset=superfast tune=zerolatency ! video/x-h264,stream-format=byte-stream ! queue ! tcpserversink host=0.0.0.0 port=5000
# VP8/TCP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! vp8enc deadline=1 ! webmmux ! queue ! tcpserversink host=0.0.0.0 port=5000
# MJPEG/UDP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! video/x-raw,format=I420 ! jpegenc ! rtpjpegpay ! queue ! udpsink host=0.0.0.0 port=5000
# MJPEG/TCP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! jpegenc ! multipartmux ! queue ! tcpserversink host=0.0.0.0 port=5000
# Raw/TCP (very high-bandwidth)
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! tcpserversink host=0.0.0.0 port=5000
# Raw/FIFO (very high-bandwidth)
rm -f $TMPDIR/mydisplay.pipe && mkfifo $TMPDIR/mydisplay.pipe
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! filesink location=$TMPDIR/mydisplay.pipe
# Raw/Shared Memory (very high-bandwidth)
rm -f $TMPDIR/mydisplay.shm
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! shmsink socket-path=$TMPDIR/mydisplay.shm sync=false wait-for-connection=true
```

Decoding in guest:

```shell
ssh -p 8022 u0_a395@192.168.1.73

export DISPLAY=:0
termux-x11 :0 &

proot-distro login fedora --shared-tmp

export GALLIUM_DRIVER=llvmpipe MESA_GL_VERSION_OVERRIDE=4.0 DISPLAY=:0 XDG_RUNTIME_DIR="${TMPDIR}" # We can't use `virglrenderer` or `virglrenderer-mesa-zink` here or we get `*** stack smashing detected ***: terminated` or `glx: failed to create drisw screen` respectively

# H264/UDP
gst-launch-1.0 udpsrc port=5000 ! h264parse ! queue ! avdec_h264 ! videoconvert ! autovideosink
ffplay -i udp://localhost:5000 -codec:v h264 -flags low_delay -fflags nobuffer
# H264/TCP
gst-launch-1.0 tcpclientsrc host=localhost port=5000 ! h264parse ! queue ! avdec_h264 ! videoconvert ! autovideosink
ffplay -i tcp://localhost:5000 -codec:v h264 -flags low_delay -fflags nobuffer
# VP8/TCP
gst-launch-1.0 tcpclientsrc host=localhost port=5000 ! matroskademux ! queue ! vp8dec ! videoconvert ! autovideosink
ffplay -i tcp://localhost:5000 -codec:v vp8 -flags low_delay -fflags nobuffer -analyzeduration 0 -probesize 32
# MJPEG/UDP
gst-launch-1.0 udpsrc port=5000 caps="application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)JPEG, payload=(int)26" ! rtpjpegdepay ! queue ! jpegdec ! videoconvert ! autovideosink
# MJPEG/TCP
gst-launch-1.0 tcpclientsrc host=localhost port=5000 ! multipartdemux ! jpegdec ! videoconvert ! autovideosink
ffplay -f mjpeg -i tcp://localhost:5000 -flags low_delay -fflags nobuffer -analyzeduration 0 -probesize 32
# Raw/TCP (very high-bandwidth)
gst-launch-1.0 tcpclientsrc host=localhost port=5000 ! videoparse width=720 height=1440 framerate=60/1 format=bgra ! queue ! videoconvert ! autovideosink
ffplay -f rawvideo -pixel_format rgb32 -video_size 720x1440 -framerate 60 -i tcp://localhost:5000 -flags low_delay -fflags nobuffer -analyzeduration 0 -probesize 32
# Raw/FIFO (very high-bandwidth)
gst-launch-1.0 filesrc location=$TMPDIR/mydisplay.pipe ! videoparse width=720 height=1440 framerate=60/1 format=bgra ! queue ! videoconvert ! autovideosink
ffplay -f rawvideo -pixel_format rgb32 -video_size 720x1440 -framerate 60 -i $TMPDIR/mydisplay.pipe -flags low_delay -fflags nobuffer -analyzeduration 0 -probesize 32
# Raw/Shared Memory (very high-bandwidth) - we can't use this to communicate between proot and non-proot!
gst-launch-1.0 shmsrc socket-path=$TMPDIR/mydisplay.shm ! video/x-raw,framerate=60/1,width=720,height=1440,format=BGRA ! queue max-size-buffers=1 leaky=2 ! videoconvert ! autovideosink
```

Decoding on host:

> This doesn't work yet with GStreamer; use `ffplay` instead

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

export DISPLAY=:0
termux-x11 :0 &

export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0 # For `virglrenderer-android`
export MESA_NO_ERROR=1 MESA_GL_VERSION_OVERRIDE=4.3COMPAT MESA_GLES_VERSION_OVERRIDE=3.2 GALLIUM_DRIVER=zink ZINK_DESCRIPTORS=lazy XDG_RUNTIME_DIR="${TMPDIR}" # For `virglrenderer-mesa-zink`

# Use same commands from "Decoding in guest"
```
