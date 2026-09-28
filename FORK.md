# Patched build (this fork)

This fork is upstream [Immortal](https://github.com/starbrightlab/immortal) plus three screensaver
fixes. They're on the `smb-fixes` branch.

- **Whole SMB share:** the screensaver used to stop after 1,000 photos. It now reads the whole
  share.
- **EXIF rotation for SMB photos:** portrait shots from a NAS no longer show sideways.
- **Shuffle without repeats:** with shuffle on, every photo is shown once, then the list is
  reshuffled for the next round. The same photo never shows twice in a row.

## Getting the APK

Every push runs the **build-apk** workflow, which runs the unit tests and builds a debug APK. Open
[Actions → build-apk](../../actions/workflows/build-apk.yml), click the latest green run, and
download **immortal-debug-apk** from the Artifacts section. It's a zip containing
`app-debug.apk`. You have to be signed in to GitHub, and artifacts are deleted after 90 days.

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
adb install app-debug.apk

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

## Updating the patched build

Every CI build signs the APK with a new debug key, so a newer build won't install over an older
one. To update, uninstall first, which wipes its settings:

```sh
adb uninstall com.immortal.launcher.debug
adb install app-debug.apk
```

Then repeat steps 2 to 4.
