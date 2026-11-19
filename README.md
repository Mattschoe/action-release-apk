# action-release-apk

A composite GitHub Action that builds an Android APK with Gradle and attaches it (plus
optional documentation) to the GitHub Release that triggered it. Reference it from any of
your Android projects instead of copy-pasting build/sign/upload YAML.

## Usage

Below is an example of a basic usage:

```yaml
name: Build and Attach Release Artifacts
on:
  release:
    types: [published]
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write          
    steps:
      - uses: Mattschoe/action-release-apk@v1
        with:
          keystore-base64:   ${{ secrets.KEYSTORE_BASE64 }}
          keystore-password: ${{ secrets.KEYSTORE_PASSWORD }}
          key-alias:         ${{ secrets.KEY_ALIAS }}
          key-password:      ${{ secrets.KEY_PASSWORD }}
```

Cut a GitHub Release (e.g. via release-please or manually) and the APK is attached as an
asset, ready to download and install on a phone. Omit the signing inputs (`keystore-*`and `key-*`) to build an unsigned APK (no secrets required).

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `working-directory` | _(auto-detect)_ | Folder containing the Gradle project (`gradlew`), relative to the repo root. See [Project location](#project-location). |
| `java-version` | `21` | JDK version to build with. |
| `gradle-task` | `assembleRelease` | Gradle task that produces the release APK, e.g. `:androidApp:assembleRelease`. |
| `apk-path` | _(auto-detect)_ | Path or glob of the APK(s) to attach. Empty finds the single APK in any module's `build/outputs/apk/release/`. |
| `keystore-path` | `app/keystore.jks` | Where the decoded keystore is written (must match `build.gradle.kts`). |
| `docs-task` | _(none)_ | Optional extra Gradle task to generate docs, e.g. `dokkaGenerate`. |
| `docs-dir` | _(none)_ | Optional directory zipped into `documentation.zip` and attached. |
| `keystore-base64` | _(none)_ | Base64-encoded release keystore (`base64 -i keystore.jks`). Omit to build unsigned. |
| `keystore-password` | _(none)_ | Keystore password. |
| `key-alias` | _(none)_ | Key alias. |
| `key-password` | _(none)_ | Key password. |

`apk-path`, `keystore-path` and `docs-dir` are relative to the Gradle project folder, not the
repo root. For a project at the repo root the two are the same.

## Project location

The Gradle project doesn't have to sit at the repo root. When `working-directory` is empty the
action uses the repo root if it has `gradlew`; otherwise it searches subfolders (up to 4 levels,
skipping `build/`, `.gradle/` and `node_modules/`) and uses the single folder it finds. If it
finds none or several, it fails and asks you to set `working-directory`. The chosen folder is
logged ("Using Gradle project at ./app") and exposed as the `project-dir` output.

Likewise, when `apk-path` is empty the action attaches the one APK under
`**/build/outputs/apk/release/`, whatever the module is called and whether or not it is signed
(`app-release.apk` or `app-release-unsigned.apk`). If several modules produce release APKs, set
`apk-path`; an explicit glob may match several files and all of them are attached.

The action uses the workflow's built-in `GITHUB_TOKEN` to upload, so grant the calling job
`permissions: contents: write`.

## Signing convention

Signing happens during the Gradle build. Your `app/build.gradle.kts` should read the
keystore the action decodes to `app/keystore.jks` (relative to the Gradle project folder; set
`keystore-path` if your app module has another name) and the password env vars:

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

## Example: project with Dokka docs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: Mattschoe/action-release-apk@v1
        with:
          keystore-base64:   ${{ secrets.KEYSTORE_BASE64 }}
          keystore-password: ${{ secrets.KEYSTORE_PASSWORD }}
          key-alias:         ${{ secrets.KEY_ALIAS }}
          key-password:      ${{ secrets.KEY_PASSWORD }}
          docs-task: dokkaGenerate
          docs-dir:  app/build/dokka/html
```

## Example: Gradle project in a subfolder

For a repo where the Gradle build lives in `app/` and the Android module is `androidApp`
(e.g. Kotlin Multiplatform), auto-detection finds both the project and the APK; only the
keystore location needs to follow the module name:

```yaml
      - uses: Mattschoe/action-release-apk@v1
        with:
          gradle-task:       ':androidApp:assembleRelease'
          keystore-path:     androidApp/keystore.jks
          keystore-base64:   ${{ secrets.KEYSTORE_BASE64 }}
          keystore-password: ${{ secrets.KEYSTORE_PASSWORD }}
          key-alias:         ${{ secrets.KEY_ALIAS }}
          key-password:      ${{ secrets.KEY_PASSWORD }}
```
