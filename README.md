# Unified Release Registry & Distribution Standard

Central release distribution registry, updater endpoint, and artifact host for all **@mrun1corn** projects.

---

## 1. Architecture & Purpose

This repository serves two primary roles:
1. **Direct Binary / Artifact Hosting**: Holds GitHub Releases containing binaries, installers, APKs, and archives for all projects.
2. **Central Update Manifest Registry**: Hosts version metadata under `manifests/<project>.json`. Client applications poll raw GitHub URLs without needing GitHub API tokens or running into rate limits.

```
Client App (e.g., u_app)
    │
    ▼ 1. Polls Update Manifest
https://raw.githubusercontent.com/mrun1corn/release/main/manifests/<project>.json
    │
    ▼ 2. Compares Version (SemVer)
Is latest > currentVersion?
    │
    ▼ 3. Downloads Binary Asset from GitHub Release
https://github.com/mrun1corn/release/releases/download/<tag>/<asset>
```

---

## 2. Strict Naming & Versioning Rules

All releases across all projects **MUST** adhere to these naming and versioning specifications.

### A. Project Identifiers (`project`)
- Project names must be lowercase alphanumeric with hyphens or underscores (e.g. `u_app`, `weather_cli`, `portfolio-desktop`).
- Must match the manifest filename: `manifests/<project>.json`.

### B. Tagging Specification
Every GitHub Release tag in this repository must follow this exact format:

$$\text{Tag Format} = \mathbf{<project>\text{-}v<MAJOR>.<MINOR>.<PATCH>[\text{-}<PRERELEASE>]}$$

**Examples:**
- `u_app-v0.1.0`
- `u_app-v1.0.0`
- `u_app-v1.2.3-beta.1`
- `desktop_client-v2.0.0-rc.2`

> **Note**: Tags without the `<project>-v` prefix will cause collisions in this multi-project repository and are rejected by automated validation.

### C. Release Title Specification
The GitHub Release title must follow:
```
[<project>] v<MAJOR>.<MINOR>.<PATCH> - <Release Title>
```
**Example:** `[u_app] v0.1.0 - Initial Portal Client Release`

### D. Asset Naming Specification
Attached binaries and artifacts must include the project identifier, version, platform, and optional architecture:

$$\mathbf{<project>\text{-}v<version>\text{-}<platform>[\text{-}<arch>].<ext>}$$

| Platform | Format Example | Notes |
| :--- | :--- | :--- |
| **Android APK** | `u_app-v0.1.0-android-universal.apk` | Universal APK |
| **Android ABI Split** | `u_app-v0.1.0-android-arm64-v8a.apk` | Target architecture APK |
| **Windows Installer** | `u_app-v0.1.0-windows-x64-setup.exe` | NSIS / InnoSetup Installer |
| **Windows Portable** | `u_app-v0.1.0-windows-x64.zip` | Standalone portable zip |
| **macOS DMG** | `u_app-v0.1.0-macos-universal.dmg` | Signed DMG |
| **Linux AppImage** | `u_app-v0.1.0-linux-x86_64.AppImage` | AppImage bundle |
| **Linux Deb** | `u_app-v0.1.0-linux-amd64.deb` | Debian package |

---

## 3. Update Manifest Specification (`manifests/<project>.json`)

Each project maintains a manifest file in `manifests/<project>.json`. Client apps query this file directly.

### Schema Definition
```json
{
  "$schema": "../schemas/release-manifest.schema.json",
  "project": "u_app",
  "name": "University Portal App",
  "latest_version": "0.1.0",
  "latest_version_code": 1,
  "release_tag": "u_app-v0.1.0",
  "release_date": "2026-09-21T00:00:00Z",
  "mandatory": false,
  "min_supported_version": "0.0.1",
  "title": "v0.1.0 Initial Beta",
  "changelog": [
    "Brand new UI overhaul with UniRide design system",
    "Background data synchronization and visual indicators",
    "Real-time class routine and payments tracker",
    "Automated in-app updater integration"
  ],
  "release_notes_url": "https://github.com/mrun1corn/release/releases/tag/u_app-v0.1.0",
  "assets": [
    {
      "platform": "android",
      "arch": "universal",
      "filename": "u_app-v0.1.0-android-universal.apk",
      "download_url": "https://github.com/mrun1corn/release/releases/download/u_app-v0.1.0/u_app-v0.1.0-android-universal.apk",
      "size_bytes": 28450123,
      "sha256": ""
    }
  ]
}
```

### Key Field Reference
- **`latest_version`**: Standard SemVer string (`MAJOR.MINOR.PATCH`).
- **`latest_version_code`**: Monotonically increasing integer (matching Android `versionCode` or build number).
- **`mandatory`**: If `true`, client apps must block usage until updated (breaking protocol/API changes).
- **`min_supported_version`**: Older versions below this will be forced to update even if `mandatory` is false.
- **`changelog`**: Array of human-readable bullet points displayed in the update dialog.
- **`assets`**: List of direct downloadable binaries keyed by platform and architecture.

---

## 4. Release Checklist & Publishing Workflow

When releasing a new version for any project:

### Step 1: Update the Project Manifest
Edit or create `manifests/<project>.json` in the `main` branch:
1. Increment `latest_version` and `latest_version_code`.
2. Update `release_tag` (e.g. `<project>-v<version>`).
3. Add changelog bullet points.
4. Add asset entries with accurate filenames and download URLs.
5. Commit and push to `main`.

### Step 2: Create GitHub Release & Upload Assets
Using GitHub CLI (`gh`):

```bash
# Example for u_app version 0.1.0
gh release create u_app-v0.1.0 \
  --repo mrun1corn/release \
  --title "[u_app] v0.1.0 - Initial Release" \
  --notes-file CHANGELOG.md \
  path/to/u_app-v0.1.0-android-universal.apk
```

---

## 5. Client Integration Guide (In-App Updater)

Client apps should implement update checks using this algorithm:

1. **Fetch Manifest**:
   ```http
   GET https://raw.githubusercontent.com/mrun1corn/release/main/manifests/<project>.json
   ```
2. **Compare Version**:
   Parse `latest_version` against local version (e.g., `0.1.0` vs `0.2.0`).
3. **Check Urgency**:
   If `current_version < min_supported_version` or `mandatory == true`, display a non-dismissible modal. Otherwise, display an optional update prompt.
4. **Download / Install**:
   Direct user to `download_url` or trigger in-app download and installation.

---

## 6. Project Registry

| Project ID | Name | Platforms | Latest Version | Manifest |
| :--- | :--- | :--- | :--- | :--- |
| `u_app` | University Portal App | Android | `0.1.0` | [`manifests/u_app.json`](manifests/u_app.json) |

---

## 7. Automated CI & Validation

This repository runs automated GitHub Actions (`.github/workflows/validate-manifests.yml`) on every push and pull request to verify:
- All manifest files conform to `schemas/release-manifest.schema.json`.
- Tag names follow `<project>-v<semver>`.
- Download URLs match the specified release tags.
