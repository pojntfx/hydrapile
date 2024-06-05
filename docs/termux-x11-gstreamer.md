# Termux X11-based GStreamer Experiments

Setting up SSH:

```shell
# In Termux
mkdir -p ~/.ssh
curl 'https://github.com/pojntfx.keys' > ~/.ssh/authorized_keys
sshd
```

Setting up GL:

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
glxgears
```

H264 encoding in guest:

```shell
# In new tmux pane
apt install -y gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-x gstreamer1.0-libav

# UDP and RTP
gst-launch-1.0 ximagesrc use-damage=0 ! videorate ! video/x-raw,framerate=60/1 ! clockoverlay shaded-background=true font-desc="Sans 38" ! videoconvert ! queue ! x264enc tune=zerolatency speed-preset=superfast ! rtph264pay config-interval=1 pt=96 ! queue ! udpsink host=0.0.0.0 port=5000
# TCP
gst-launch-1.0 ximagesrc use-damage=0 ! video/x-raw,framerate=60/1 ! videoconvert ! clockoverlay shaded-background=true font-desc="Sans 38" ! queue ! x264enc speed-preset=superfast tune=zerolatency ! video/x-h264,stream-format=byte-stream ! queue ! tcpserversink host=0.0.0.0 port=5000
```

H264 decoding in guest (we can't do H264 encoding on the host because the `x264enc` element isn't available):

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

export DISPLAY=:0
termux-x11 :0 &

proot-distro login debian --shared-tmp

export GALLIUM_DRIVER=llvmpipe MESA_GL_VERSION_OVERRIDE=4.0 DISPLAY=:0 XDG_RUNTIME_DIR=/tmp # We can't use virpipe here or we get `*** stack smashing detected ***: terminated`
# UDP and RTP
gst-launch-1.0 udpsrc port=5000 ! application/x-rtp, payload=96 ! rtph264depay ! h264parse ! avdec_h264 ! videoconvert ! autovideosink sync=false
# TCP
gst-launch-1.0 tcpclientsrc host=localhost port=5000 ! h264parse ! avdec_h264 ! videoconvert ! autovideosink
```

H264 decoding on host (leads to artifacting):

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

pkg install -y gst-libav gst-plugins-bad gst-plugins-base gst-plugins-gl-headers gst-plugins-good gst-plugins-ugly gstreamer

export DISPLAY=:0
termux-x11 :0 &

export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0 XDG_RUNTIME_DIR=/data/data/com.termux/files/usr/tmp
# UDP and RTP
gst-launch-1.0 udpsrc port=5000 ! application/x-rtp, payload=96 ! rtph264depay ! h264parse ! avdec_h264 ! videoconvert ! autovideosink sync=false
# TCP
gst-launch-1.0 tcpclientsrc host=localhost port=5000 ! h264parse ! avdec_h264 ! videoconvert ! autovideosink
```

VP9 encoding on host (very slow):

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

export XAUTHORITY=/data/data/com.termux/files/usr/tmp/xvfb-run.LrURH3/Xauthority # Adjust this to the real path

export DISPLAY=:99
export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0 XDG_RUNTIME_DIR=/data/data/com.termux/files/usr/tmp
gst-launch-1.0 ximagesrc use-damage=0 ! videorate ! video/x-raw,framerate=60/1 ! videoconvert ! queue ! vp9enc ! rtpvp9pay pt=96 ! queue ! udpsink host=0.0.0.0 port=5000
```

VP9 decoding on host (very slow):

```shell
ssh -p 8022 u0_a395@fels-google-pixel-6-pro.koi-monitor.ts.net

export DISPLAY=:0
termux-x11 :0 &

export GALLIUM_DRIVER=virpipe MESA_GL_VERSION_OVERRIDE=4.0 XDG_RUNTIME_DIR=/tmp
gst-launch-1.0 udpsrc port=5000 ! application/x-rtp, payload=96, encoding-name=VP9, clock-rate=90000 ! rtpvp9depay ! vp9dec ! videoconvert ! autovideosink sync=false
```
