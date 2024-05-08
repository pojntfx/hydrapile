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
