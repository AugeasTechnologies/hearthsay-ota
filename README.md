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
    "sizeBytes": 82345678,
    "mandatory": false,
    "notes": "What changed in this build."
  }
  ```

  `sizeBytes` is the exact byte length of the APK. `sha256` is published so a
  person can verify a download by hand (`sha256sum`); the app does not compute
  it, because hashing an ~80 MB file in JavaScript would take tens of seconds on
  the phone of the person waiting for it.

- The in-app updater (`apps/mobile/lib/appUpdater.ts`) fetches the raw URL, compares
  `versionCode` to the build's baked-in `APP_VERSION_CODE`, downloads the APK with
  `expo-file-system`, and launches the Android package installer via a
  `FileProvider` content URI (`expo-intent-launcher`).

## What is actually verified

Worth being exact, because the two are often confused:

- **Android is the security boundary.** It refuses to install an update whose
  package is signed by a different certificate than the installed app, so a
  substituted APK cannot replace Hearthsay on anyone's phone. This is why every
  published build must be signed with the real Hearthsay release certificate —
  CI does this from repository secrets and fails the build if a release-signed
  APK did not come out (`.github/workflows/mobile.yml`).
- **The app's own check is an integrity check**, not a second security boundary.
  Before handing the file to the package installer it checks the HTTP status and
  compares the downloaded byte length to `sizeBytes`
  (`packages/shared/src/updates/manifest.ts`). That catches the realistic
  failures: an interrupted transfer, and a GitHub error page served in place of
  a missing release asset — which would otherwise be written into a `.apk` file
  and produce a meaningless "package appears to be corrupt" error on a screen
  the family opened because the app asked them to.

A manifest published without `sizeBytes` still installs; releases predate the
field, and stranding them would strand exactly the builds with no other way
forward. The app reports those as unverified rather than as verified.
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
