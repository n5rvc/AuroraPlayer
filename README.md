# Aurora — offline-first Android music player

A flagship-style local music player written in Kotlin: Jetpack Compose (Material 3), Media3
(ExoPlayer + `MediaLibraryService`), Room, DataStore, Coil 3, Palette and a hand-written DSP
chain on top of the ExoPlayer audio session.

No network, no accounts, no analytics. Everything is read from folders you grant via the
Storage Access Framework.

---

## Build

Requires **Android Studio Ladybug (2024.2) or newer** — Meerkat/Narwhal work too — with JDK 17.

1. `File → Open` and pick this folder.
2. Let Gradle sync (uses the bundled wrapper, Gradle 8.9 / AGP 8.7.3 / Kotlin 2.0.21).
3. Run the `app` configuration on a device or emulator with **API 26+** (10-band DSP needs API 28+).
4. On first launch grant *Music and audio* (and notifications, so playback controls appear).

Debug builds install as `com.aurora.player.debug`, so they can live next to a store build.
A prebuilt, installable debug APK ships next to this archive as `AuroraPlayer-debug.apk`
(`adb install AuroraPlayer-debug.apk`). Release signing and store prep: see `RELEASE.md`.

Compose Compiler is provided by the Kotlin 2.0 `org.jetbrains.kotlin.plugin.compose` plugin;
Room is the only annotation processor (KSP). Everything else uses a small manual DI container
(`di/AppContainer.kt`), so there is no Hilt/KSP setup to get wrong.

---

## Architecture

```
data/
  model/Models.kt            domain models, EQ band table, codec badge formatting
  db/                        Room entities, DAOs, database (tracks, playlists, rules, queue)
  settings/                  DataStore settings + AppSettings snapshot
  scanner/                   SAF + MediaStore scanner, metadata reader, artwork cache
  lyrics/                    LRC parser, ID3 USLT/SYLT reader, lyrics repository
  playlist/M3uSupport.kt     .m3u/.m3u8 parser, resolver and writer
  tag/TagEditor.kt           jaudiotagger tag + artwork writer (copy → edit → stream back)
  repository/                combined flows, albums/artists/genres/folder tree
audio/
  AudioEffectsManager.kt     10-band DynamicsProcessing (+5-band fallback), Virtualizer,
                             PresetReverb, BassBoost, LoudnessEnhancer, Spatializer
  ReplayGainReader.kt        scans REPLAYGAIN_TRACK_GAIN from ID3/Vorbis tags
playback/
  PlaybackController.kt      app-scoped ExoPlayer, queue, crossfade, sleep timer, DSP push
  PlaybackService.kt         MediaLibraryService session + browse tree + Bluetooth/shake
  ShakeDetector.kt           accelerometer skip gesture
ui/
  theme/                     theme engine, Album-art Palette extraction
  components/                fluid mesh background, mini player, track row, synced lyrics
  screens/                   Now Playing, Library, Audio Studio, Settings, dialogs
  viewmodel/                 Player / Library / AudioStudio / Settings + factory
widget/
  PlayerWidgetProvider.kt    home-screen widget (RemoteViews, no player instance held)
  WidgetBridge.kt            cached snapshot + MediaButton transport intents
```

Playback lives in an app-scoped `PlaybackController`; `PlaybackService` wraps the same
ExoPlayer in a `MediaSession`/`MediaLibrarySession` so the notification, Bluetooth controls,
Android Auto and the in-app UI never disagree.

---

## Feature notes

**Library** — SAF folder import with `takePersistableUriPermission` (no
`MANAGE_EXTERNAL_STORAGE`), MediaStore fallback, folder hierarchy browser built from scanned
paths, short-clip/size filters, folder blacklist, rescan with progress.

**Now Playing** — Palette-derived 4–5 colour mesh background (animated, one-time blur, no
per-frame bitmaps), spring-scaled artwork with deeper shadow while paused, drag-down to
collapse, codec badge (`FLAC • 24-bit / 96kHz`, `MP3 • 320kbps`), queue sheet, output
switcher.

**Lyrics** — sidecar `.lrc` first (sibling document probe), then embedded ID3 `SYLT`/`USLT`
or Vorbis `LYRICS`. Active line scales/glows, neighbours dim, tap to seek.

**DSP** — 10 true bands 31 Hz…16 kHz through `DynamicsProcessing` (pre-EQ + limiter, and a
post-EQ band used as a stereo balance trim); automatic 5-band `Equalizer` fallback on API
26/27. Surround = `Virtualizer` width + room `PresetReverb` + `Spatializer` when the device
advertises it. Sub-bass is shelved onto the three lowest bands. ReplayGain via
`LoudnessEnhancer`. Every control applies live and persists with a 250 ms debounce.

**Widget** — resizable home-screen card with artwork, title/artist and previous/play-pause/next.
Transport taps are `KEYCODE_MEDIA_*` events routed through Media3's `MediaButtonReceiver`, so
the widget drives the same `MediaSession` as the notification and Bluetooth. The layout is
rebuilt from a cache written by `PlaybackService`, so a launcher rebuild never needs a player.

---

## Honest limitations

- **Build status**: `assembleDebug` and `assembleRelease` both succeed here with Gradle 8.9,
  AGP 8.7.3, Kotlin 2.0.21 and SDK 35; the release variant passes R8 (2.8 MB, shrunk from a
  22 MB debug APK). The app was compiled, not played: no device or emulator is attached to the
  build environment, so widget rendering, SAF import and per-device DSP behaviour are
  code-reviewed but not exercised on hardware.
- **Gapless**: Media3 renders gapless natively for MP3/AAC/FLAC. Public APIs have no "disable
  gapless" switch, so turning the setting off inserts a 350 ms seam at each transition instead.
  Crossfade is a fade-out/fade-in on one player, not a true two-deck overlap.
- **Cast / AirPlay**: the Now Playing button lists physical outputs and relies on Media3's
  automatic Bluetooth/USB routing. Google Cast needs the Cast SDK and AirPlay needs a receiver;
  neither is bundled.
- **Tag editing** writes via jaudiotagger after copying to cache. Format coverage depends on
  jaudiotagger (MP3, FLAC, M4A, OGG, WAV typically fine). Back up irreplaceable files.
- **SAF scanning** walks `DocumentFile` trees recursively; first import of tens of thousands of
  files is slow (progress is shown). Rescans preserve play counts by URI.
- **Embedded SYLT** parsing covers the common UTF-8/UTF-16 layouts; unusual encodings fall back
  to unsynced text.
- **Album art** is extracted into the app cache; artwork is re-extracted when the cache is
  cleared (a rescan is not required).
- Room's `fallbackToDestructiveMigration()` means a future schema change wipes cached metadata
  (audio files themselves are never touched).
