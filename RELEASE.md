# Release checklist

## 1. Version

Bump both numbers in `app/build.gradle.kts` before every upload:

```kotlin
versionCode = 2          // must increase for each Play upload
versionName = "1.1.0"
```

Debug builds install as `com.aurora.player.debug`, so a debug build can sit next to a store
build without wiping its library.

## 2. Create an upload keystore (once)

```bash
keytool -genkeypair -v \
  -keystore aurora-release.jks \
  -alias aurora \
  -keyalg RSA -keysize 4096 -validity 10000
```

Keep `aurora-release.jks` **outside** the repo (or anywhere, since `*.jks` is gitignored) and
back it up — losing it means you can never update the app on Play with the same listing.

## 3. Wire the keystore

Create `keystore.properties` in the project root (gitignored):

```properties
storeFile=../aurora-release.jks
storePassword=…
keyAlias=aurora
keyPassword=…
```

`app/build.gradle.kts` reads it defensively: if the file is missing, the release build is
simply unsigned instead of failing, which keeps CI artifact builds and fresh clones working.

## 4. Build

```bash
./gradlew assembleDebug          # APK for device testing
./gradlew assembleRelease        # signed APK (if keystore.properties exists)
./gradlew bundleRelease          # AAB for Play Console
```

Outputs land in `app/build/outputs/`.

Before shipping, verify the release build (R8 enabled) on a real device:

- import a music folder via SAF and confirm a full rescan
- play a FLAC with a `.lrc` sidecar and one with embedded lyrics
- open Audio Studio, move all 10 bands, toggle surround, change room mode
- set a 1-minute sleep timer with "finish current track"
- add the widget, then force-stop the app and confirm the widget still renders
- test Bluetooth connect/disconnect auto-play and pause
- rotate the device on Now Playing (no state loss)

R8 notes: jaudiotagger reflection and the widget provider are already covered in
`app/proguard-rules.pro`. If R8 strips something at runtime, the crash stack will name the
class — add a targeted `-keep` rather than disabling minification.

## 5. Store listing assets

Screenshots must be captured from a real device or emulator at the sizes Play requires
(phone 1080×1920 minimum, 16:9 or 9:16). Good set for this app:

1. Library — Songs tab with a populated library
2. Now Playing — mesh background, artwork, codec badge
3. Now Playing — synced lyrics with the active line glowing
4. Audio Studio — EQ curve plus band sliders
5. Folders tab — hierarchy browser
6. Settings — theme engine

Take them with `adb exec-out screencap -p > shot.png`, and hide personal file names by using a
test library of free-licensed audio.

Feature graphic: 1024×500 PNG. App icon: 512×512 PNG exported from the adaptive icon.

## 6. Play Console declarations

- **Data safety**: no data collected or shared, no network permission requested at all.
- **Permissions**: `READ_MEDIA_AUDIO` (audio files) and SAF-imported folders; media playback
  foreground service; notifications for transport controls. No `MANAGE_EXTERNAL_STORAGE` — say
  so explicitly in the permission declaration, it is a common review question.
- **Ads**: none. **In-app purchases**: none.
- **Content rating**: everyone; the app plays user-supplied audio.
- **Privacy policy**: still required by Play even with zero collection — a short page stating
  "Aurora reads only the music folders you select, stores metadata on-device, and transmits
  nothing" is enough.

## 7. Tag and publish the repo

```bash
git init
git add .
git commit -m "Aurora 1.0.0"
git remote add origin git@github.com:<you>/aurora-player.git
git tag -a v1.0.0 -m "Aurora 1.0.0"
git push -u origin main --tags
```

Never commit `keystore.properties`, `*.jks`, `local.properties` or build output — all four are
already in `.gitignore`.
