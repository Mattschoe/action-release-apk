# action-release-apk

A composite GitHub Action that builds a Kotlin/Gradle release artifact and attaches it (plus
optional documentation) to the GitHub Release that triggered it. Reference it from any of your
Android or Compose Multiplatform projects instead of copy-pasting build/sign/upload YAML.

Two targets are supported:

| `target` | Builds | Attaches |
|---|---|---|
| `android` (default) | `assembleRelease` | the signed (or unsigned) `.apk` |
| `desktop` | `packageReleaseDistributionForCurrentOS` | the native installer for the runner's OS (`.dmg`/`.pkg`, `.msi`/`.exe`, `.deb`/`.rpm`) |

> **iOS is not supported.** A `.ipa` attached to a GitHub Release isn't installable by whoever
> downloads it — iOS distribution goes through TestFlight, which doesn't involve Releases at all.
> See [#2](https://github.com/Mattschoe/action-release-apk/issues/2) for the discussion.

## Usage

### Android

```yaml
name: Build and Attach Release Artifacts
on:
  release:
    types: [published]
jobs:
  android:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: Mattschoe/action-release-apk@v2
        with:
          android-keystore-base64:   ${{ secrets.KEYSTORE_BASE64 }}
          android-keystore-password: ${{ secrets.KEYSTORE_PASSWORD }}
          android-key-alias:         ${{ secrets.KEY_ALIAS }}
          android-key-password:      ${{ secrets.KEY_PASSWORD }}
```

Cut a GitHub Release (e.g. via release-please or manually) and the APK is attached as an asset,
ready to download and install on a phone. Omit the signing inputs to build an unsigned APK (no
secrets required).

### Desktop

`packageReleaseDistributionForCurrentOS` can only produce installers for the OS it runs on —
jpackage cannot cross-build — so run one job per OS:

```yaml
  desktop:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    permissions:
      contents: write
    steps:
      - uses: Mattschoe/action-release-apk@v2
        with:
          target: desktop
```

Each leg attaches its own installer, so one release ends up with a `.deb`, a `.dmg` and a `.msi`.

### Both, plus Dokka docs

```yaml
jobs:
  android:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: Mattschoe/action-release-apk@v2
        with:
          gradle-task:               ':androidApp:assembleRelease'
          artifact-path:             'androidApp/build/outputs/apk/release/androidApp-release.apk'
          android-keystore-path:     'androidApp/keystore.jks'
          android-keystore-base64:   ${{ secrets.KEYSTORE_BASE64 }}
          android-keystore-password: ${{ secrets.KEYSTORE_PASSWORD }}
          android-key-alias:         ${{ secrets.KEY_ALIAS }}
          android-key-password:      ${{ secrets.KEY_PASSWORD }}
          docs-task: dokkaGenerate
          docs-dir:  androidApp/build/dokka/html

  desktop:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    permissions:
      contents: write
    steps:
      - uses: Mattschoe/action-release-apk@v2
        with:
          target: desktop
```

## Inputs

### Shared

| Input | Default | Description |
|-------|---------|-------------|
| `target` | `android` | `android` or `desktop`. |
| `java-version` | `21` | JDK version to build with. |
| `gradle-task` | _per target_ | Gradle task that produces the artifact. Several space-separated tasks are allowed. |
| `artifact-path` | _per target_ | Path/glob of the artifact(s) to attach, **one per line**. |
| `docs-task` | _(none)_ | Optional extra Gradle task to generate docs, e.g. `dokkaGenerate`. |
| `docs-dir` | _(none)_ | Optional directory zipped into `documentation.zip` and attached. |

### Android

| Input | Default | Description |
|-------|---------|-------------|
| `android-keystore-path` | `app/keystore.jks` | Where the decoded keystore is written (must match `build.gradle.kts`). |
| `android-keystore-base64` | _(none)_ | Base64-encoded release keystore (`base64 -i keystore.jks`). Omit to build unsigned. |
| `android-keystore-password` | _(none)_ | Keystore password. |
| `android-key-alias` | _(none)_ | Key alias. |
| `android-key-password` | _(none)_ | Key password. |

### Desktop

| Input | Default | Description |
|-------|---------|-------------|
| `desktop-module` | `composeApp` | Module holding the Compose Desktop application. Only used to build the default `artifact-path`. |

### Defaults resolved per target

`gradle-task` and `artifact-path` are empty by default and filled in based on `target`:

| `target` | `gradle-task` | `artifact-path` |
|---|---|---|
| `android` | `assembleRelease` | `app/build/outputs/apk/release/app-release.apk` |
| `desktop` | `packageReleaseDistributionForCurrentOS` | every `<desktop-module>/build/compose/binaries/main-release/<fmt>/*.<fmt>` for `dmg`, `pkg`, `msi`, `exe`, `deb`, `rpm` |

The desktop default lists all six jpackage formats so one value works on every runner; the formats
that don't apply to the current OS simply match nothing.

The action uses the workflow's built-in `GITHUB_TOKEN` to upload, so grant the calling job
`permissions: contents: write`.

## What the action decides, and what your Gradle config decides

The action is not opinionated about what gets built. It runs `./gradlew <gradle-task>` and uploads
whatever `<artifact-path>` matches — your `build.gradle.kts` is the source of truth. For desktop
that means `nativeDistributions` decides which installers exist at all:

```kotlin
compose.desktop {
    application {
        nativeDistributions {
            targetFormats(TargetFormat.Dmg, TargetFormat.Msi, TargetFormat.Deb)
            packageName = "MyApp"
            packageVersion = System.getenv("VERSION_NAME") ?: "1.0.0"
        }
    }
}
```

If nothing matches `artifact-path`, the job **fails** rather than publishing a release with no
assets attached.

Two things to watch on desktop:

- `packageVersion` must be a strict `x.y.z` and is validated per format (`.msi` rejects a `0`
  major, `.dmg` has its own rules), so a `v1.0.0-rc1` tag will fail packaging. Either don't attach
  desktop builds to pre-releases, or sanitise the version in Gradle.
- If you switch to an uber-JAR instead of installers, every matrix leg produces the *same*
  filename and they'll clobber each other on the release. Add an OS suffix if you do.

## Signing convention (Android)

Signing happens during the Gradle build. Your `app/build.gradle.kts` should read the keystore the
action decodes to `app/keystore.jks` and the password env vars:

```kotlin
android {
    defaultConfig {
        // Use the CI-provided version (from the release tag) when present.
        versionName = System.getenv("VERSION_NAME") ?: "1.0.0"
    }
    signingConfigs {
        create("release") {
            val keystoreFile = file("keystore.jks")
            if (keystoreFile.exists()) {
                storeFile = keystoreFile
                storePassword = System.getenv("KEYSTORE_PASSWORD")
                keyAlias = System.getenv("KEY_ALIAS")
                keyPassword = System.getenv("KEY_PASSWORD")
            }
        }
    }
    buildTypes {
        getByName("release") {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

`VERSION_NAME` is exported from the release tag (with any leading `v` stripped) for both targets.

## Migrating from v1

v2 renames the Android-specific inputs so the action can carry more than one platform. Behaviour
for an Android build is otherwise unchanged.

| v1 | v2 |
|----|----|
| `apk-path` | `artifact-path` |
| `keystore-path` | `android-keystore-path` |
| `keystore-base64` | `android-keystore-base64` |
| `keystore-password` | `android-keystore-password` |
| `key-alias` | `android-key-alias` |
| `key-password` | `android-key-password` |
| `java-version`, `gradle-task`, `docs-task`, `docs-dir` | unchanged |

Also new in v2: a build that matches no artifacts now fails instead of publishing an empty release.
