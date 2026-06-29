# hearthsay-ota

Over-the-air update channel for the **Hearthsay** Android app.

The app checks `latest.json` on launch (and from Settings → Check for updates). When
the published `versionCode` is newer than the installed build, it offers to download
the APK release asset and install it — no cable, no Play Store.

## How it works

- `latest.json` (this repo, `main` branch, served raw) is the manifest:

  ```json
  {
    "versionCode": 2,
    "versionName": "1.1.0",
    "apkUrl": "https://github.com/AugeasTechnologies/hearthsay-ota/releases/download/v1.1.0/Hearthsay-v1.1.0.apk",
    "sha256": "<hex>",
    "mandatory": false,
    "notes": "What changed in this build."
  }
  ```

- The in-app updater (`apps/mobile/lib/appUpdater.ts`) fetches the raw URL, compares
  `versionCode` to the build's baked-in `APP_VERSION_CODE`, downloads the APK with
  `expo-file-system`, verifies size, and launches the Android package installer via a
  `FileProvider` content URI (`expo-intent-launcher`).
- The user grants "install unknown apps" for Hearthsay once; thereafter updates are
  one tap.

## Publishing a release

From the monorepo:

```
apps/mobile/scripts/release-ota.sh <versionName> <versionCode> "<notes>"
```

It builds the release APK, creates the GitHub release + uploads the APK asset, and
updates `latest.json` on `main`. See the script for details.

## Note

The **first** OTA-capable build must be installed once over USB (it carries the
updater). Every build after that arrives over the air.
