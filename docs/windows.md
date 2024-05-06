# Windows Experiments

```shell
docker run --privileged --ulimit nofile=32768:32768 -e MSYSTEM=MINGW64 -v ${PWD}:/root/.wine/drive_c/msys64/home/root/delfin:z -it ghcr.io/msys2/msys2-docker-experimental

pacman -Sy --noconfirm

pacman -S --noconfirm git
rm -rf ~/delfin
git clone https://codeberg.org/avery42/delfin.git ~/delfin
cd ~/delfin

pacman -S --noconfirm mingw-w64-x86_64-meson mingw-w64-x86_64-gcc mingw-w64-x86_64-gtk4 mingw-w64-x86_64-libadwaita mingw-w64-x86_64-mpv mingw-w64-x86_64-libepoxy mingw-w64-x86_64-rust mingw-w64-x86_64-desktop-file-utils

cmd.exe # We need to use `cmd.exe`, otherwise Cargo uses a mix of `/` and `\` in the code generation steps
meson setup build --wipe
cd build
meson compile
# Exit cmd.exe here

pacman -R --noconfirm git mingw-w64-x86_64-meson mingw-w64-x86_64-gcc mingw-w64-x86_64-rust mingw-w64-x86_64-desktop-file-utils

rm -rf delfin/build/dist
cp -r /mingw64 delfin/build/dist/
cp delfin/delfin.exe delfin/build/dist/bin/delfin.exe

# On host
sudo chown -R ${USER} build/delfin/build
cd build/data && glib-compile-resources cafe.avery.Delfin.Devel.gresource.xml --sourcedir . --sourcedir ../../data --internal --generate --target ../../build/delfin/build/dist/bin/resources.gresource --dependency-file resources.gresource.d; cd ../..
zip -r build/delfin/build/delfin.zip build/delfin/build/dist/

ww send build/delfin/build/delfin.zip # To distribute the complete app
ww send build/delfin/build/dist/bin/delfin.exe # To update only the `.exe`

# In WINE on other host where the ZIP file has been shared to
cmd.exe
cd 'build\delfin\build\dist'
delfin.exe
```

```diff
diff --git a/data/meson.build b/data/meson.build
index 8dc5743..faed403 100644
--- a/data/meson.build
+++ b/data/meson.build
@@ -48,12 +48,13 @@ resources_file = configure_file(
   },
 )

-# Compile and install resources
-gnome.compile_resources(
-  'resources',
-  resources_file,
-  gresource_bundle: true,
-  source_dir: meson.current_build_dir(),
-  install: true,
-  install_dir: pkgdatadir,
-)
+# We need to disable this since "glib-compile-resources.exe" returns "WSADuplicateSocket() failed: ." (on MSYS64, CLANG64 and UCRT)
+## Compile and install resources
+#gnome.compile_resources(
+#  'resources',
+#  resources_file,
+#  gresource_bundle: true,
+#  source_dir: meson.current_build_dir(),
+#  install: true,
+#  install_dir: pkgdatadir,
+#)
diff --git a/delfin/meson.build b/delfin/meson.build
index a23e4b1..77ea568 100644
--- a/delfin/meson.build
+++ b/delfin/meson.build
@@ -26,20 +26,26 @@ if get_option('profile') == 'release'
   rust_target = 'release'
 endif

-custom_target(
+rust_build = custom_target(
   'cargo-build',
   build_by_default: true,
   build_always_stale: true,
-  output: meson.project_name(),
+  output: meson.project_name() + '-raw',
   console: true,
-  command: [
-    cargo, 'build', cargo_options,
-    '&&', 'cp', 'delfin' / rust_target / meson.project_name(), '@OUTPUT@',
-  ],
-  install: true,
-  install_dir: bindir,
+  command: [cargo, 'build', cargo_options],
   env: {
     'MESON_BUILD_ROOT': meson.project_build_root(),
     'FLATPAK': get_option('flatpak') ? 'true' : 'false',
   },
 )
+
+custom_target(
+  'copy-build',
+  build_by_default: true,
+  depends: rust_build,
+  command: ['cp', 'delfin' / rust_target / meson.project_name(), '@OUTPUT@'],
+  output: 'some_output_name',  # Specify your output name here
+  output: meson.project_name(),
+  install: true,
+  install_dir: bindir,
+)
diff --git a/delfin/src/borgar/about.rs b/delfin/src/borgar/about.rs
index 38aec71..7f9cbc8 100644
--- a/delfin/src/borgar/about.rs
+++ b/delfin/src/borgar/about.rs
@@ -1,7 +1,7 @@
 use gtk::prelude::*;
 use relm4::{prelude::*, ComponentParts, SimpleComponent};

-use crate::meson_config::{APP_ID, VERSION};
+use crate::meson_config::VERSION;

 pub(crate) struct About;

@@ -12,10 +12,7 @@ impl SimpleComponent for About {
     type Output = ();

     view! {
-        adw::AboutWindow::from_appdata(
-            &format!("/cafe/avery/Delfin/{}.metainfo.xml", APP_ID),
-            Some(VERSION),
-        ) {
+        adw::AboutWindow::new() {
             set_modal: true,
             set_visible: true,
             set_version: VERSION,
diff --git a/delfin/src/main.rs b/delfin/src/main.rs
index 26cad10..2bb7656 100644
--- a/delfin/src/main.rs
+++ b/delfin/src/main.rs
@@ -1,9 +1,9 @@
-use std::{fmt::Debug, path::PathBuf};
+use std::{fmt::Debug};

 use anyhow::{bail, Context, Result};
 use delfin::{
     app::{App, APP_BROKER},
-    meson_config::{APP_ID, BUILDDIR, RESOURCES_FILE},
+    meson_config::{APP_ID, RESOURCES_FILE},
 };
 use gtk::gio;
 use relm4::{gtk, RelmApp};
@@ -37,7 +37,7 @@ fn load_resources() -> Result<()> {
     let res = match gio::Resource::load(RESOURCES_FILE) {
         Ok(res) => res,
         Err(_) if cfg!(debug_assertions) => {
-            gio::Resource::load(PathBuf::from(BUILDDIR).join("data/resources.gresource"))
+            gio::Resource::load("resources.gresource")
                 .context("Could not load fallback gresource file")?
         }
         Err(_) => bail!("Could not load gresource file"),
diff --git a/delfin/src/video_player/mod.rs b/delfin/src/video_player/mod.rs
index 6946d69..b5dd159 100644
--- a/delfin/src/video_player/mod.rs
+++ b/delfin/src/video_player/mod.rs
@@ -1,7 +1,6 @@
 pub mod backends;
 mod controls;
 mod keybindings;
-mod mpris;
 mod next_up;
 mod session;
 mod skip_intro;
@@ -45,7 +44,6 @@ use self::controls::play_pause::{PlayPauseInput, PLAY_PAUSE_BROKER};
 use self::controls::scrubber::{ScrubberInput, SCRUBBER_BROKER};
 use self::controls::volume::{VolumeInput, VOLUME_BROKER};
 use self::controls::{VideoPlayerControls, VideoPlayerControlsInput};
-use self::mpris::MprisPlaybackReporter;
 use self::next_up::NextUpInput;
 use self::session::SessionPlaybackReporter;
 use self::skip_intro::{SkipIntro, SkipIntroInput};
@@ -63,7 +61,6 @@ pub struct VideoPlayer {
     show_controls_locked: bool,
     revealer_reveal_child: bool,
     session_playback_reporter: SessionPlaybackReporter,
-    mpris_playback_reporter: Option<MprisPlaybackReporter>,
     inhibit_cookie: Option<InhibitCookie>,
     player_state: PlayerState,
     next: Option<BaseItemDto>,
@@ -238,7 +235,6 @@ impl Component for VideoPlayer {
             show_controls_locked: false,
             revealer_reveal_child: show_controls,
             session_playback_reporter: SessionPlaybackReporter::default(),
-            mpris_playback_reporter: None,
             inhibit_cookie: None,
             player_state: PlayerState::Loading,
             next: None,
@@ -399,11 +395,6 @@ impl Component for VideoPlayer {
                 self.session_playback_reporter
                     .start(&api_client, &item.id.unwrap(), &self.backend);

-                self.mpris_playback_reporter = Some(MprisPlaybackReporter::new(
-                    api_client.clone(),
-                    *item.clone(),
-                    self.backend.clone(),
-                ));

                 self.api_client = Some(api_client);

@@ -477,8 +468,6 @@ impl Component for VideoPlayer {
                 // Stop background playback progress reporter
                 self.session_playback_reporter.stop(&self.backend);

-                self.mpris_playback_reporter = None;
-
                 FULLSCREEN_BROKER.send(FullscreenInput::ExitFullscreen);
             }
             VideoPlayerInput::PlayerStateChanged(play_state) => {
diff --git a/delfin/src/video_player/mpris.rs b/delfin/src/video_player/mpris.rs
deleted file mode 100644
index 51f6650..0000000
--- a/delfin/src/video_player/mpris.rs
+++ /dev/null
@@ -1,273 +0,0 @@
-use std::{
-    cell::RefCell,
-    sync::{Arc, Mutex},
-    time::Duration,
-};
-
-use jellyfin_api::types::BaseItemDto;
-use souvlaki::{
-    MediaControlEvent, MediaControls, MediaMetadata, MediaPlayback, MediaPosition, PlatformConfig,
-    SeekDirection,
-};
-use tokio::sync::mpsc::{self, UnboundedSender};
-use uuid::Uuid;
-
-use crate::{
-    app::{AppInput, APP_BROKER},
-    jellyfin_api::api_client::ApiClient,
-    meson_config::APP_ID,
-    utils::item_name::ItemName,
-    video_player::controls::next_prev_episode::{
-        NextPrevEpisodeInput, NEXT_EPISODE_BROKER, PREV_EPISODE_BROKER,
-    },
-};
-
-use super::{
-    backends::{PlayerState, VideoPlayerBackend},
-    controls::{
-        play_pause::{PlayPauseInput, PLAY_PAUSE_BROKER},
-        skip_forwards_backwards::{
-            SkipForwardsBackwardsInput, SKIP_BACKWARDS_BROKER, SKIP_FORWARDS_BROKER,
-        },
-    },
-};
-
-#[derive(Debug)]
-enum MprisInput {
-    Play,
-    Pause,
-    Duration(usize),
-    Position(usize),
-    Close,
-}
-
-pub struct MprisPlaybackReporter {
-    video_player: Arc<RefCell<dyn VideoPlayerBackend>>,
-    signal_handler_ids: Vec<Uuid>,
-    tx: UnboundedSender<MprisInput>,
-}
-
-impl MprisPlaybackReporter {
-    pub fn new(
-        api_client: Arc<ApiClient>,
-        item: BaseItemDto,
-        video_player: Arc<RefCell<dyn VideoPlayerBackend>>,
-    ) -> Self {
-        let config = PlatformConfig {
-            dbus_name: APP_ID,
-            display_name: "Delfin",
-            // Will need to pass a window handle for Windows support
-            hwnd: None,
-        };
-
-        let mut controls = MediaControls::new(config).expect("Failed creating MediaControls");
-        attach_media_control_events(&mut controls);
-        let controls = Arc::new(Mutex::new(controls));
-
-        let (tx, mut rx) = mpsc::unbounded_channel::<MprisInput>();
-
-        let mut signal_handler_ids = Vec::new();
-
-        signal_handler_ids.push(
-            video_player
-                .borrow_mut()
-                .connect_player_state_changed(Box::new({
-                    let tx = tx.clone();
-                    move |player_state| {
-                        if tx.is_closed() {
-                            return;
-                        }
-                        if let PlayerState::Playing { paused } = player_state {
-                            tx.send(if paused {
-                                MprisInput::Pause
-                            } else {
-                                MprisInput::Play
-                            })
-                            .expect("Failed to update MPRIS state");
-                        }
-                    }
-                })),
-        );
-
-        signal_handler_ids.push(
-            video_player
-                .borrow_mut()
-                .connect_duration_updated(Box::new({
-                    let tx = tx.clone();
-                    move |duration| {
-                        if tx.is_closed() {
-                            return;
-                        }
-                        tx.send(MprisInput::Duration(duration))
-                            .expect("Failed to update MPRIS duration");
-                    }
-                })),
-        );
-
-        signal_handler_ids.push(
-            video_player
-                .borrow_mut()
-                .connect_position_updated(Box::new({
-                    let tx = tx.clone();
-                    move |position| {
-                        if tx.is_closed() {
-                            return;
-                        }
-                        tx.send(MprisInput::Position(position))
-                            .expect("Failed to update MPRIS position");
-                    }
-                })),
-        );
-
-        tokio::spawn({
-            let controls = controls.clone();
-            async move {
-                let title = item.episode_name_with_number();
-                let series_name = item.series_name.clone();
-                let cover_url = api_client.get_next_up_thumbnail_url(&item).ok();
-                let metadata = MediaMetadata {
-                    title: title.as_deref(),
-                    album: series_name.as_deref(),
-                    cover_url: cover_url.as_deref(),
-                    ..Default::default()
-                };
-
-                controls
-                    .lock()
-                    .unwrap()
-                    .set_metadata(metadata.clone())
-                    .expect("Failed to set MPRIS metadata");
-
-                let mut paused = false;
-
-                while let Some(msg) = rx.recv().await {
-                    let mut controls = controls
-                        .lock()
-                        .expect("Failed to acquire lock on MPRIS media controls.");
-
-                    match msg {
-                        MprisInput::Play => {
-                            controls
-                                .set_playback(MediaPlayback::Playing { progress: None })
-                                .expect("Error setting MPRIS playback");
-                            paused = false;
-                        }
-                        MprisInput::Pause => {
-                            controls
-                                .set_playback(MediaPlayback::Paused { progress: None })
-                                .expect("Error setting MPRIS playback");
-                            paused = true;
-                        }
-                        MprisInput::Duration(duration) => {
-                            controls
-                                .set_metadata(MediaMetadata {
-                                    duration: Some(Duration::from_secs(duration as u64)),
-                                    ..metadata.clone()
-                                })
-                                .expect("Error setting MPRIS metadata");
-                        }
-                        MprisInput::Position(position) => {
-                            controls
-                                .set_playback(if paused {
-                                    MediaPlayback::Paused {
-                                        progress: Some(MediaPosition(Duration::from_secs(
-                                            position as u64,
-                                        ))),
-                                    }
-                                } else {
-                                    MediaPlayback::Playing {
-                                        progress: Some(MediaPosition(Duration::from_secs(
-                                            position as u64,
-                                        ))),
-                                    }
-                                })
-                                .expect("Error setting MPRIS playback");
-                        }
-                        MprisInput::Close => {
-                            rx.close();
-                            return;
-                        }
-                    }
-                }
-            }
-        });
-
-        Self {
-            video_player,
-            signal_handler_ids: Vec::default(),
-            tx,
-        }
-    }
-}
-
-impl Drop for MprisPlaybackReporter {
-    fn drop(&mut self) {
-        for id in self.signal_handler_ids.drain(0..) {
-            self.video_player
-                .borrow_mut()
-                .disconnect_signal_handler(&id);
-        }
-
-        self.tx
-            .send(MprisInput::Close)
-            .expect("Failed to close MPRIS playback reporter channel");
-    }
-}
-
-fn attach_media_control_events(controls: &mut MediaControls) {
-    controls
-        .attach(|event: MediaControlEvent| {
-            match event {
-                MediaControlEvent::Play => {
-                    PLAY_PAUSE_BROKER.send(PlayPauseInput::SetPlaying(true));
-                }
-                MediaControlEvent::Pause => {
-                    PLAY_PAUSE_BROKER.send(PlayPauseInput::SetPlaying(false));
-                }
-                MediaControlEvent::Toggle => {
-                    PLAY_PAUSE_BROKER.send(PlayPauseInput::TogglePlaying);
-                }
-                MediaControlEvent::SeekBy(direction, amount) => match direction {
-                    SeekDirection::Forward => {
-                        SKIP_FORWARDS_BROKER.send(SkipForwardsBackwardsInput::SkipByAmount(amount));
-                    }
-                    SeekDirection::Backward => {
-                        SKIP_BACKWARDS_BROKER
-                            .send(SkipForwardsBackwardsInput::SkipByAmount(amount));
-                    }
-                },
-                MediaControlEvent::Seek(direction) => match direction {
-                    SeekDirection::Forward => {
-                        SKIP_FORWARDS_BROKER.send(SkipForwardsBackwardsInput::Skip);
-                    }
-                    SeekDirection::Backward => {
-                        SKIP_BACKWARDS_BROKER.send(SkipForwardsBackwardsInput::Skip);
-                    }
-                },
-                MediaControlEvent::SetPosition(MediaPosition(position)) => {
-                    SKIP_BACKWARDS_BROKER.send(SkipForwardsBackwardsInput::SkipTo(position));
-                }
-                MediaControlEvent::Previous => {
-                    PREV_EPISODE_BROKER.send(NextPrevEpisodeInput::Play);
-                }
-                MediaControlEvent::Next => {
-                    NEXT_EPISODE_BROKER.send(NextPrevEpisodeInput::Play);
-                }
-                MediaControlEvent::Stop | MediaControlEvent::Quit => {
-                    APP_BROKER.send(AppInput::NavigateBack);
-                }
-                MediaControlEvent::Raise => {
-                    APP_BROKER.send(AppInput::Present);
-                }
-                #[allow(clippy::match_same_arms)]
-                MediaControlEvent::SetVolume(_) => {
-                    // TODO
-                }
-                #[allow(clippy::match_same_arms)]
-                MediaControlEvent::OpenUri(_) => {
-                    // not supported
-                }
-            }
-        })
-        .unwrap();
-}
diff --git a/video_player_mpv/sys/video-player-mpv/track-list.h b/video_player_mpv/sys/video-player-mpv/track-list.h
index cb86ae1..84ca275 100644
--- a/video_player_mpv/sys/video-player-mpv/track-list.h
+++ b/video_player_mpv/sys/video-player-mpv/track-list.h
@@ -2,6 +2,8 @@

 #include <gtk/gtk.h>

+typedef unsigned int uint;
+
 #include "track.h"

 G_BEGIN_DECLS
diff --git a/video_player_mpv/sys/video-player-mpv/video-player-mpv.c b/video_player_mpv/sys/video-player-mpv/video-player-mpv.c
index 2c1c152..0492f2a 100644
--- a/video_player_mpv/sys/video-player-mpv/video-player-mpv.c
+++ b/video_player_mpv/sys/video-player-mpv/video-player-mpv.c
@@ -2,7 +2,7 @@
 #include "video-player-mpv/track.h"

 #include <epoxy/egl.h>
-#include <epoxy/glx.h>
+// #include <epoxy/glx.h>
 #include <gtk/gtk.h>
 #include <locale.h>
 #include <mpv/client.h>
@@ -103,7 +103,7 @@ static void vpm_video_player_mpv_class_init(VpmVideoPlayerMpvClass *klass) {
 }

 static void *get_proc_address(void *fn_ctx, const gchar *name) {
-  GdkDisplay *display = gdk_display_get_default();
+  // GdkDisplay *display = gdk_display_get_default();

 #ifdef GDK_WINDOWING_WAYLAND
   if (GDK_IS_WAYLAND_DISPLAY(display)) {
@@ -115,11 +115,11 @@ static void *get_proc_address(void *fn_ctx, const gchar *name) {
     return (void *)(intptr_t)glXGetProcAddressARB((const GLubyte *)name);
   }
 #endif
-#ifdef GDK_WINDOWING_WIN32
-  if (GDK_IS_WIN32_DISPLAY(display)) {
+// #ifdef GDK_WINDOWING_WIN32
+  // if (GDK_IS_WIN32_DISPLAY(display)) {
     return wglGetProcAddress(name);
-  }
-#endif
+  // }
+// #endif
   g_assert_not_reached();
   return NULL;
 }
diff --git a/video_player_mpv/sys/video-player-mpv/video-player-mpv.h b/video_player_mpv/sys/video-player-mpv/video-player-mpv.h
index a888a5c..e7f9c7e 100644
--- a/video_player_mpv/sys/video-player-mpv/video-player-mpv.h
+++ b/video_player_mpv/sys/video-player-mpv/video-player-mpv.h
@@ -5,6 +5,8 @@
 #include <stdbool.h>
 #include <sys/types.h>

+typedef unsigned int uint;
+
 G_BEGIN_DECLS

 G_DECLARE_FINAL_TYPE(VpmVideoPlayerMpv, vpm_video_player_mpv, VPM,
```
