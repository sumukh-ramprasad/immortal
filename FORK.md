# Patched build (this fork)

This fork is upstream [Immortal](https://github.com/starbrightlab/immortal) plus three screensaver
fixes. They're on the `smb-fixes` branch.

- **Whole SMB share:** the screensaver used to stop after 1,000 photos. It now reads the whole
  share.
- **EXIF rotation for SMB photos:** portrait shots from a NAS no longer show sideways.
- **Shuffle without repeats:** with shuffle on, every photo is shown once, then the list is
  reshuffled for the next round. The same photo never shows twice in a row.

## Getting the APK

Download `immortal-smb-fixes-debug.apk` from the
[latest release](../../releases/latest). No sign-in needed, and it doesn't expire.

Every push also runs the **build-apk** workflow, which runs the unit tests and builds a fresh
debug APK. You can get it from [Actions → build-apk](../../actions/workflows/build-apk.yml) under
Artifacts, but you have to be signed in and artifacts are deleted after 90 days. A CI build is
signed with a different key than the release APK (see Updating below).

## Why it installs as a separate app

Android identifies an app by its package name, and the debug build's package is
`com.immortal.launcher.debug`. The upstream app is `com.immortal.launcher`, so Android treats
them as two different apps, each with its own settings. The patched build can't replace upstream
directly because only the upstream developer has the key needed to sign updates to it.

The Portal has one slot for the home screen and one for the screensaver. Provisioning pointed both
at upstream. Using the patched build means pointing them at it instead.

## Install and switch over

Connect the Portal over ADB, then:

```sh
# 1. Install
adb install immortal-smb-fixes-debug.apk

# 2. Make it the home screen and the screensaver
adb shell cmd package set-home-activity com.immortal.launcher.debug/com.immortal.launcher.HomeActivity
adb shell settings put secure screensaver_components com.immortal.launcher.debug/com.immortal.launcher.PhotoDreamService

# 3. Let it turn the screen off (screensaver sleep needs this)
adb shell dpm set-active-admin com.immortal.launcher.debug/com.immortal.launcher.AdminReceiver
```

In each command, the part before the `/` is the new package name and the part after it is the
class name, which is still the original. That's why `.debug` and `com.immortal.launcher` appear
together.

4. Press Home on the Portal to open the patched Immortal. Its settings start empty, so enter the
   SMB share in the screensaver settings again and turn shuffle on.

## Switch back to upstream

```sh
adb shell cmd package set-home-activity com.immortal.launcher/.HomeActivity
adb shell settings put secure screensaver_components com.immortal.launcher/.PhotoDreamService
```

Upstream stays installed the whole time, so switching back is instant.

## Remove upstream and keep only the patched build

Provisioning gave upstream a set of permissions and made it the device admin. Move both to the
patched build before uninstalling upstream. Android won't uninstall an app while it's a device
admin.

**1. Give the patched build upstream's permissions**

```sh
P=com.immortal.launcher.debug
for perm in WRITE_SECURE_SETTINGS READ_EXTERNAL_STORAGE WRITE_EXTERNAL_STORAGE READ_LOGS CAMERA RECORD_AUDIO; do
  adb shell pm grant $P android.permission.$perm
done
adb shell appops set $P SYSTEM_ALERT_WINDOW allow
adb shell appops set $P REQUEST_INSTALL_PACKAGES allow
adb shell appops set $P GET_USAGE_STATS allow
adb shell cmd notification allow_listener $P/com.immortal.launcher.MediaNotificationListenerService
adb shell dpm set-active-admin $P/com.immortal.launcher.AdminReceiver
```

These are the grants `provision.sh` gives upstream (`grant_perms`). Without them the app store,
the Home Assistant camera, now-playing, and screen-off may not work.

**2. Remove upstream's device admin**

```sh
adb shell dpm remove-active-admin com.immortal.launcher/.AdminReceiver
```

Android usually blocks this from ADB. If it errors, do it on the Portal: open **Settings →
Security → Device admin apps** (the exact menu name can vary), open the original **Immortal**,
and tap **Deactivate**. Both apps are called Immortal in that list, so make sure you pick the
original.

**3. Uninstall upstream**

```sh
adb shell cmd notification disallow_listener com.immortal.launcher/com.immortal.launcher.MediaNotificationListenerService
adb uninstall com.immortal.launcher
```

**4. Check**

```sh
adb shell cmd package resolve-activity -a android.intent.action.MAIN -c android.intent.category.HOME | grep packageName
adb shell settings get secure screensaver_components
adb shell dumpsys device_policy | grep -i admin
```

All three should show `com.immortal.launcher.debug`. If the home screen check doesn't, run the
`set-home-activity` command from "Install and switch over" again.

**Things that change without upstream**

- **No automatic updates.** New upstream releases don't reach the patched build. To get them,
  merge `upstream/main` into this fork and rebuild.
- **The kit's restore script won't work as-is.** `provision.sh` restore targets
  `com.immortal.launcher`. To go back to Meta's stock launcher, first set these in
  `provisioning/config.env`. Use full class names: the short `/.Name` form would add `.debug` to
  the class too.

  ```sh
  PKG=com.immortal.launcher.debug
  HOME_ACTIVITY=com.immortal.launcher.debug/com.immortal.launcher.HomeActivity
  DREAM_SERVICE=com.immortal.launcher.debug/com.immortal.launcher.PhotoDreamService
  ```

## Updating the patched build

Every CI build signs the APK with a new debug key, so a newer build won't install over an older
one. To update, uninstall first, which wipes its settings:

```sh
adb uninstall com.immortal.launcher.debug
adb install immortal-smb-fixes-debug.apk
```

Then repeat steps 2 to 4 of "Install and switch over", and step 1 of "Remove upstream".
