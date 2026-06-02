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
| `java-version` | `21` | JDK version to build with. |
| `gradle-task` | `assembleRelease` | Gradle task that produces the release APK. |
| `apk-path` | `app/build/outputs/apk/release/app-release.apk` | APK to attach. |
| `keystore-path` | `app/keystore.jks` | Where the decoded keystore is written (must match `build.gradle.kts`). |
| `docs-task` | _(none)_ | Optional extra Gradle task to generate docs, e.g. `dokkaGenerate`. |
| `docs-dir` | _(none)_ | Optional directory zipped into `documentation.zip` and attached. |
| `keystore-base64` | _(none)_ | Base64-encoded release keystore (`base64 -i keystore.jks`). Omit to build unsigned. |
| `keystore-password` | _(none)_ | Keystore password. |
| `key-alias` | _(none)_ | Key alias. |
| `key-password` | _(none)_ | Key password. |

The action uses the workflow's built-in `GITHUB_TOKEN` to upload, so grant the calling job
`permissions: contents: write`.

## Signing convention

Signing happens during the Gradle build. Your `app/build.gradle.kts` should read the
keystore the action decodes to `app/keystore.jks` and the password env vars:

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
