# Bluewater Machine App — APK Releases

This repository hosts the signed Android builds and metadata that the **Bluewater Machine** tablet app uses for over-the-air (OTA) self-updates and for first-time Device Owner provisioning. It contains **no source code** — only published release artifacts.

> **This repository is maintained automatically.** Every release here is published by the Bluewater Machine App **source repo's tooling** — you never need to build, upload, edit, or commit anything in this repo by hand. There is no manual work to do here for an APK release.

## What's in a release

Every GitHub Release carries two assets:

- **`app.apk`** — the signed Android build.
- **`metadata.json`** — describes that build:

```json
{
  "versionCode": 76,
  "versionName": "6.1.2",
  "apkUrl": "https://github.com/Youmoni/apkreleases/releases/download/v76/app.apk",
  "sha256": "<sha-256 of app.apk>"
}
```

## Release types

| Type | Tag | GitHub flag | Who sees it |
|------|-----|-------------|-------------|
| **Production** | `v<versionCode>` (e.g. `v76`) | **Latest** | All tablets in the field |
| **Staging / pre-release** | `v<versionCode>-staging` | **Pre-release** | Only admins running the in-app pre-release check — never production |

Because staging builds are marked *pre-release*, GitHub excludes them from **Latest**, so production tablets never pick them up.

## How a tablet uses this repo

1. **Checks metadata:**
   - Production devices read the Latest pointer: `releases/latest/download/metadata.json`.
   - Staging builds are found via the GitHub Releases API (newest pre-release), behind an admin-only check.
2. **Compares `versionCode`** — if the release is higher than the installed version, it downloads `app.apk` from `apkUrl`.
3. **Verifies + installs** — checks the download against `sha256`, validates package name + versionCode, then installs via Android `PackageInstaller`. On the kiosk tablets (Device Owner) the install is silent.

Downloads are anonymous — the public repo means tablets need no credentials.

## First-time provisioning (the base build)

New tablets are set up from a **Device Owner provisioning QR** that points at a fixed base APK:

```
https://github.com/Youmoni/apkreleases/releases/download/v74/app.apk
```

- The QR also embeds the **signing-certificate checksum**, so the base APK must always be signed with the same keystore.
- **The printed QR must not change.** To refresh the base, overwrite the `app.apk` asset *in place* (same tag, same signing key) — the QR stays valid:

```bash
gh release upload v74 /path/to/app.apk --repo Youmoni/apkreleases --clobber
```

After provisioning, the tablet OTA-updates to the current Latest.

## Publishing (done from the app source repo)

Releases are produced by the app repo's tooling — **not by hand in this repo**:

```bash
npm run publish-update-github -- /path/to/app.apk
# prompt: prod  or  staging
```

It creates/updates the GitHub Release, uploads `app.apk` + `metadata.json`, marks production releases as `--latest`, and verifies the published metadata is reachable and matches. Nothing in this repo needs to be touched manually.

## The "Latest" pointer — important

Tags like `v76` are **not valid semver**, so GitHub resolves "Latest" by **creation date**, not version number. If a lower version is published more recently it can wrongly become Latest. The publisher forces `--latest` on production releases to avoid this. To repair manually:

```bash
gh release edit v<versionCode> --repo Youmoni/apkreleases --latest
```

## Rules of thumb

- **Same keystore, always.** Base, staging, and production APKs must share the signing key — otherwise OTA installs fail (signature mismatch) and provisioning rejects the download.
- **`versionCode` must increase** for an update to be offered; devices ignore an equal/lower code. Re-cutting the same version replaces the release in place but won't re-update tablets already on it.
- **Never delete the Latest release** without immediately repointing Latest, or tablets will fetch an older build.
