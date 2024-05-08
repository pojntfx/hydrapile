# macOS Experiments

See: https://docs.darlinghq.org/contributing/updating-sources/index.html?highlight=updating%20sour#updating-version-number

```diff
diff --git a/src/frameworks/CoreServices/SystemVersion.plist b/src/frameworks/CoreServices/SystemVersion.plist
index ee78ea540..4b9334e5f 100644
--- a/src/frameworks/CoreServices/SystemVersion.plist
+++ b/src/frameworks/CoreServices/SystemVersion.plist
@@ -5,14 +5,14 @@
 	<key>ProductBuildVersion</key>
 	<string>Darling</string>
 	<key>ProductCopyright</key>
-	<string>2012-2023 Lubos Dolezel</string>
+	<string>2012-2024 Lubos Dolezel</string>
 	<key>ProductName</key>
 	<string>macOS</string>
 	<key>ProductUserVisibleVersion</key>
-	<string>11.7.4</string>
+	<string>14.4.1</string>
 	<key>ProductVersion</key>
-	<string>11.7.4</string>
+	<string>14.4.1</string>
 	<key>iOSSupportVersion</key>
-	<string>14.7</string>
+	<string>17.4</string>
 </dict>
 </plist>
diff --git a/src/frameworks/CoreServices/SystemVersionCompat.plist b/src/frameworks/CoreServices/SystemVersionCompat.plist
index 2ad16ea5b..6d12a4673 100644
--- a/src/frameworks/CoreServices/SystemVersionCompat.plist
+++ b/src/frameworks/CoreServices/SystemVersionCompat.plist
@@ -5,7 +5,7 @@
 	<key>ProductBuildVersion</key>
 	<string>Darling</string>
 	<key>ProductCopyright</key>
-	<string>2012-2023 Lubos Dolezel</string>
+	<string>2012-2024 Lubos Dolezel</string>
 	<key>ProductName</key>
 	<string>Mac OS X</string>
 	<key>ProductUserVisibleVersion</key>
@@ -13,6 +13,6 @@
 	<key>ProductVersion</key>
 	<string>10.16</string>
 	<key>iOSSupportVersion</key>
-	<string>14.7</string>
+	<string>17.4</string>
 </dict>
 </plist>
```

```plaintext
# Change the section in src/external/xnu/darling/src/libsystem_kernel/emulation/linux/CMakeLists.txt to:
add_definitions(-DBSDTHREAD_WRAP_LINUX_PTHREAD
	-DEMULATED_SYSNAME="Darwin"
	-DEMULATED_RELEASE="23.4.0"
	-DEMULATED_VERSION="Darwin Kernel Version 23.4.0"
	-DEMULATED_OSVERSION="23E224"
	-DEMULATED_OSPRODUCTVERSION="14.4.1"
)
```

```shell
# We need to use the `vfs` storage driver since we get `overlay: filesystem on /root/.darling not supported as upperdir`/`Cannot mount overlay: Invalid argument` otherwise
sudo mkdir -p /etc/docker/
echo '{
  "storage-driver": "vfs"
}' | sudo tee /etc/docker/daemon.json
systemctl restart docker

docker run -it --privileged -v ${PWD}/darling:/data:z debian:bookworm
cd /data
apt update
apt install -y sudo cmake clang bison flex xz-utils libfuse-dev libudev-dev pkg-config libc6-dev-i386 libcap2-bin git git-lfs libglu1-mesa-dev libcairo2-dev libgl1-mesa-dev libtiff5-dev libfreetype6-dev libxml2-dev libegl1-mesa-dev libfontconfig1-dev libbsd-dev libxrandr-dev libxcursor-dev libgif-dev libpulse-dev libavformat-dev libavcodec-dev libswresample-dev libdbus-1-dev libxkbfile-dev libssl-dev llvm-dev
mkdir -p build && cd build
cmake ..
make -j$(nproc)
make -j$(nproc) install
adduser --disabled-password --gecos "" lisa
usermod -aG sudo lisa
su - lisa
darling shell

# Try https://github.com/darlinghq/darling-docs/compare/40b57a6c95d6152d2bf057f02499fbeb1a7877ed...caba31680071d106b17daaf3d8ad89eebbe55b98

# Manually press "y" to accept the XCode EULA
NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

(echo; echo 'eval "$(/usr/local/bin/brew shellenv)"') >> /Users/lisa/.bash_profile
eval "$(/usr/local/bin/brew shellenv)"

brew install meson gcc mpv libepoxy rust desktop-file-utils

brew install gtk4 # Will fail to run because of missing `LSCopyApplicationURLsForURL` implementation, but is safe to ignore since we only need the headers
brew install libadwaita # Will fail to run because of missing `LSCopyApplicationURLsForURL` implementation, but is safe to ignore since we only need the headers
```

```c
// https://docs.gtk.org/gtk4/getting_started.html#hello-world-in-c
// example-1.c
#include <gtk/gtk.h>

static void
print_hello (GtkWidget *widget,
             gpointer   data)
{
  g_print ("Hello World\n");
}

static void
activate (GtkApplication *app,
          gpointer        user_data)
{
  GtkWidget *window;
  GtkWidget *button;
  GtkWidget *box;

  window = gtk_application_window_new (app);
  gtk_window_set_title (GTK_WINDOW (window), "Window");
  gtk_window_set_default_size (GTK_WINDOW (window), 200, 200);

  box = gtk_box_new (GTK_ORIENTATION_VERTICAL, 0);
  gtk_widget_set_halign (box, GTK_ALIGN_CENTER);
  gtk_widget_set_valign (box, GTK_ALIGN_CENTER);

  gtk_window_set_child (GTK_WINDOW (window), box);

  button = gtk_button_new_with_label ("Hello World");

  g_signal_connect (button, "clicked", G_CALLBACK (print_hello), NULL);
  g_signal_connect_swapped (button, "clicked", G_CALLBACK (gtk_window_destroy), window);

  gtk_box_append (GTK_BOX (box), button);

  gtk_window_present (GTK_WINDOW (window));
}

int
main (int    argc,
      char **argv)
{
  GtkApplication *app;
  int status;

  app = gtk_application_new ("org.gtk.example", G_APPLICATION_DEFAULT_FLAGS);
  g_signal_connect (app, "activate", G_CALLBACK (activate), NULL);
  status = g_application_run (G_APPLICATION (app), argc, argv);
  g_object_unref (app);

  return status;
}
```

```shell
gcc $( pkg-config --cflags gtk4 ) -o example-1 example-1.c $( pkg-config --libs gtk4 )

exit
```

```shell
cp ~/.darling/Users/lisa/test/example-1 /data/
# You can run `example-1` on a real Mac
```

```shell
rm -rf ~/delfin
git clone https://codeberg.org/avery42/delfin.git ~/delfin
cd ~/delfin

cargo build # You might have to re-run this a few times to to misreported `signal: 11, SIGSEGV: invalid memory reference` errors

# The steps below will fail due to `Symbol not found: (_mkfifoat)` issues of recent Python (see https://github.com/python/cpython/issues/100384) and various Rust build issues
meson setup build --wipe
cd build
meson compile
```

```shell
curl -Lo /tmp/go.pkg https://go.dev/dl/go1.21.10.darwin-amd64.pkg # We need to use v.1.21.10 - newer version fail with https://github.com/darlinghq/darling/issues/1178
sudo installer -pkg /tmp/go.pkg -target / # Will fail; feel free to ignore

git clone https://github.com/pojntfx/multiplex.git
cd multiplex

export GOPROXY=direct # Needed due to https://github.com/darlinghq/darling/issues/1454
go generate -x ./... # Might need to stop/start this a few times if it hangs

# Build still fails with:
# internal/goarch
# unexpected fault address 0x20801e0
# fatal error: fault
# [signal SIGSEGV: segmentation violation code=0x1 addr=0x20801e0 pc=0x12f0383]
```
