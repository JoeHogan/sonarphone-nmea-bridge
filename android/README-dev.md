# SP200A Bridge developer runbook

The GitHub-flavor debug APK can be built and driven entirely over ADB. Its
package is `ca.dynamicsolutions.sonarbridge`. Run commands from the repository
root unless a section says otherwise.

## Build

The long-lived `sonarbridge-builder` container is arm64-native and keeps the
Gradle cache warm. Start it if it already exists but is stopped:

```sh
docker start sonarbridge-builder
docker exec sonarbridge-builder ./gradlew assembleGithubDebug
stat android/app/build/outputs/apk/github/debug/app-github-debug.apk
```

The Gradle command must finish successfully, and the APK modification time must
advance. Do not rely on the last output line alone because a failed build can
still end with a configuration-cache message.

If the builder does not exist, create it once:

```sh
docker build -t sonarbridge-build:arm64 \
  -f android/docker/Dockerfile.arm64 android/docker
mkdir -p android/.gradle
printf 'android.aapt2FromMavenOverride=/opt/android-tools/aapt2\n' \
  > android/.gradle/gradle.properties
docker run -d --name sonarbridge-builder --restart unless-stopped \
  -u "$(id -u):$(id -g)" \
  -e GRADLE_USER_HOME=/project/.gradle \
  -v "$PWD/android:/project" -w /project \
  sonarbridge-build:arm64 sleep infinity
```

The AAPT2 override is container-local and gitignored. Do not add it to the
committed `android/gradle.properties`, where it would break non-arm64 builds.
The pinned debug keystore at `android/keystore/debug.keystore` keeps
`adb install -r` compatible across builds.

## Connect the phone

The Pixel 9 Pro already trusts the ADB key in the host's Android configuration.
Wireless debugging must still be enabled for its current network.

```sh
adb devices
python3 scripts/find-phone.py <phone-ip>
adb devices
```

The script remembers successful endpoints in `~/.android/adb-endpoint` and
rediscovers a rotated wireless-debugging port. If it has no usable saved
endpoint, get `<phone-ip>` from **Settings > Developer options > Wireless
debugging > IP address & Port**. Use the IP only.

Do not guess the phone from randomized MAC addresses or scan the whole subnet.
The Pixel does not reliably answer ICMP, other devices use randomized MACs,
and ADB mDNS does not cross WSL2 NAT.

## Install

```sh
adb install -r android/app/build/outputs/apk/github/debug/app-github-debug.apk
adb shell pm grant ca.dynamicsolutions.sonarbridge android.permission.POST_NOTIFICATIONS
adb shell dumpsys package ca.dynamicsolutions.sonarbridge | grep lastUpdateTime
```

Confirm that `lastUpdateTime` changed. A debug build and a release build use
different signing keys, so switching between them can require one uninstall
and will clear settings.

## Drive and observe

Start the service with the default `SonarPhone_` prefix and password:

```sh
adb shell am start-foreground-service \
  -n ca.dynamicsolutions.sonarbridge/.BridgeService \
  -a ca.dynamicsolutions.sonarbridge.START
```

Optional extras are `-e ssid X` for an exact SSID, `-e pattern PREFIX`,
`-e pass Y`, `-e lograw true`, `-e udp 2000`, and `-e demo true`.

Stop the service:

```sh
adb shell am start-foreground-service \
  -n ca.dynamicsolutions.sonarbridge/.BridgeService \
  -a ca.dynamicsolutions.sonarbridge.STOP
```

Observe logs and the NMEA stream:

```sh
adb logcat -s SonarBridge
adb forward tcp:10110 tcp:10110
nc 127.0.0.1 10110
```

Pull raw frames after starting with `-e lograw true`:

```sh
adb pull /sdcard/Android/data/ca.dynamicsolutions.sonarbridge/files/ ./frames/
```

The state machine is `WIFI_WAIT → DISCOVER (FX @1 Hz) → RUN (FC @10 s) →
DISCOVER` after 15 seconds of silence. `NEED_MASTER` means the T-Box is
factory-fresh; run the official SonarPhone app once. The first connection can
show Android's Wi-Fi approval dialog, which the user must accept on the phone.
If the T-Box is off, stop the service after testing so the dialog does not
reappear repeatedly.

## Watch it work

```sh
adb logcat -s SonarBridge          # STATE/REDYFX/FRAME/NMEA lines, ~1 Hz

# tap the NMEA stream from the dev box (same stream Navionics sees):
adb forward tcp:10110 tcp:10110
nc 127.0.0.1 10110                 # expect $SDDPT/$SDDBT/$YXMTW every ~1 s

# pull raw frame logs (when started with -e lograw true):
adb shell ls /sdcard/Android/data/ca.dynamicsolutions.sonarbridge/files/
adb pull /sdcard/Android/data/ca.dynamicsolutions.sonarbridge/files/ ./frames/
```

Raw log record format: `u64le wall-clock ms, u16le frame length, frame bytes`.

## Navionics pairing

Menu → Paired devices → **+** → Host `127.0.0.1`, Port `10110`, Protocol TCP.
Loopback TCP is verified with Navionics on Android. UDP on port 2000 remains a
legacy fallback.

## Validation checklist

1. `STATE DISCOVER` → `REDYFX serial=… masterMac=…` decodes sanely.
2. `STATE RUN`, `FRAME` lines: note packet size, depth/temp vs known water.
3. No-bottom-lock behavior: watch depth field with transducer out of water.
4. Stream survives on 10 s FC cadence; watchdog recovers after AP power-cycle.
5. Phone keeps internet while attached to T-Box AP (browse in another app).
6. Navionics pairs to 127.0.0.1:10110 and shows depth.
7. Screen off 10+ min: stream continues (check FRAME counter in logcat).
8. Enable Menu > SonarChart Live in Navionics while moving: our NMEA depth +
   phone GPS should draw live personal bathymetry contours (the bridge acts
   as a free Digital Yacht Sonar Server). Raw-sonar split view is NOT
   possible because Garmin removed third-party sonar rendering after v19.

## Releases & updates

- Cut a release: `git tag v0.2.0 && git push origin v0.2.0`. GitHub Actions
  builds a signed APK and publishes a Release with a filtered changelog.
- The app checks the latest release on open/resume (3 h throttle) and offers
  the APK with the release notes; "Later" mutes that version.
- Release signing: keystore lives only in GitHub secrets + gitignored
  `android/keystore/release.keystore` (creds in `release.env` beside it).
- NOTE: release APKs are signed with the release key. A phone running a
  debug build must uninstall once before its first release install
  (signature mismatch; settings are lost that one time).
