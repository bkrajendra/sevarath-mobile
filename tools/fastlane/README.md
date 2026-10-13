fastlane documentation
----

# Installation

Make sure you have the latest version of the Xcode command line tools installed:

```sh
xcode-select --install
```

For _fastlane_ installation instructions, see [Installing _fastlane_](https://docs.fastlane.tools/#installing-fastlane)

# Available Actions

## iOS

### ios bootstrap

```sh
[bundle exec] fastlane ios bootstrap
```

Local, read-write: create/fetch the App Store distribution cert + profiles for both bundle IDs

### ios ci_release

```sh
[bundle exec] fastlane ios ci_release
```

CI, read-only: fetch existing signing material, build both flavors, upload both to TestFlight

### ios create_app_records

```sh
[bundle exec] fastlane ios create_app_records
```

Create the two App Store Connect app records (idempotent - safe to re-run)

### ios upload_metadata

```sh
[bundle exec] fastlane ios upload_metadata
```

Local/manual: push App Store listing text for both apps from tools/fastlane/metadata/ios - no binary, no submission

----


## Android

### android build_apk

```sh
[bundle exec] fastlane android build_apk
```

CI: build both flavors' signed release APKs into dist/, named for a GitHub Release

### android release_playstore

```sh
[bundle exec] fastlane android release_playstore
```

CI, upload: build both flavors' signed AABs and push to Google Play (binary + version bump only - no metadata/images, see upload_play_metadata)

### android upload_play_metadata

```sh
[bundle exec] fastlane android upload_play_metadata
```

Local/manual: push Android store-listing text for both apps from tools/fastlane/metadata/android - no binary, no version bump

----

This README.md is auto-generated and will be re-generated every time [_fastlane_](https://fastlane.tools) is run.

More information about _fastlane_ can be found on [fastlane.tools](https://fastlane.tools).

The documentation of _fastlane_ can be found on [docs.fastlane.tools](https://docs.fastlane.tools).
