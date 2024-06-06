# Termux X11-based GStreamer and `ffmpeg` Experiments

Setting up SSH:

```shell
# In Termux
mkdir -p ~/.ssh
curl 'https://github.com/pojntfx.keys' > ~/.ssh/authorized_keys
sshd
```

Setting up GL/hardware acceleration:

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

pkg install virglrenderer-android
virgl_test_server_android &

proot-distro install debian
proot-distro login debian --shared-tmp
```

Setting up headless X11:

```shell
apt install -y xvfb tmux
xvfb-run --server-args="-screen 0 360x722x24+32" tmux
# Or:
# Xvfb :1 -screen 0 360x722x24+32 &
# tmux
```

Starting a hardware-accelerated X11 application:

```shell
apt install -y mesa-utils
export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0
glxgears -fullscreen
```

Encoding in guest:

```shell
# In new tmux pane
apt install -y gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-x gstreamer1.0-libav

export XDG_RUNTIME_DIR=/tmp

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
```

Decoding in guest:

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

export DISPLAY=:0
termux-x11 :0 &

proot-distro login debian --shared-tmp

export GALLIUM_DRIVER=llvmpipe MESA_GL_VERSION_OVERRIDE=4.0 DISPLAY=:0 XDG_RUNTIME_DIR=/tmp # We can't use virpipe here or we get `*** stack smashing detected ***: terminated`

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
```

Encoding on host:

> This doesn't work yet; we can connect to the X server, but we get no output

> There is no `x264enc` in Termux's GStreamer on the host

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

export XAUTHORITY=/data/data/com.termux/files/usr/tmp/xvfb-run.LrURH3/Xauthority # Adjust this to the real path
export DISPLAY=:99
export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0 XDG_RUNTIME_DIR=/data/data/com.termux/files/usr/tmp

# VP8/TCP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! queue ! vp8enc deadline=1 ! webmmux ! queue ! tcpserversink host=0.0.0.0 port=5000
# MJPEG/UDP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! queue ! video/x-raw,format=I420 ! jpegenc ! rtpjpegpay ! queue ! udpsink host=0.0.0.0 port=5000
# MJPEG/TCP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! queue ! jpegenc ! multipartmux ! queue ! tcpserversink host=0.0.0.0 port=5000
```

Decoding on host:

> This doesn't work yet with GStreamer; use `ffplay` instead

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

export DISPLAY=:0
termux-x11 :0 &

export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0 XDG_RUNTIME_DIR=/data/data/com.termux/files/usr/tmp

# Use same commands from "Decoding in guest"
```
