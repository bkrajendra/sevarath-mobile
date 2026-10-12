# iOS Build Flavors + Signing/TestFlight Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. This plan mixes **Owner: Agent** tasks (safe to delegate — code/config edits, no real secrets touched) and **Owner: You** tasks (Apple ID / App Store Connect / Firebase console / physical device — never delegate these, and never paste real secret values into any chat tool call). Work through tasks in order — later tasks depend on artifacts (bundle IDs, Team ID, API key, Firebase config) produced by earlier ones.

**Goal:** Stand up two installable iOS apps (rider `SevaRath`, driver `SevaRath Driver`) from this one Flutter codebase — mirroring the Android `user`/`driver` flavors that already ship via `android/app/build.gradle.kts` — get a real signed build running on a device/simulator, and wire CI to build + upload both to TestFlight on every push to `main`, the same way `.github/workflows/release.yml` already does for Android.

**Architecture:** iOS flavors via duplicated Xcode build configurations + two new schemes (`user`, `driver`) matching Flutter's `--flavor` convention, each with its own `PRODUCT_BUNDLE_IDENTIFIER` (`com.rajendrakhope.sevarath` / `com.rajendrakhope.sevarath.driver` — identical to the Android `applicationId`s already in use) and display name. Distribution is **TestFlight** (not Ad Hoc) — no per-device UDID registration, scales to real drivers/riders the way GitHub-Release APKs already do for Android. Signing uses **Fastlane `match`**, backed by the private `bkrajendra/mobile-signing-vault` repo that already exists (created while planning `factoryvu-flutter`'s own iOS signing, never bootstrapped — this plan bootstraps it for real, for both apps). Android's existing keystore-based signing (`android/key.properties` + GitHub Secrets) is untouched — only iOS gets Fastlane.

**Tech Stack:** Flutter 3.44.8 (already installed on this Mac, matches `flutter --version`), Xcode 27.0 (installed), Fastlane (Ruby/Bundler), GitHub Actions `macos-15` runner, App Store Connect API key, `mobile-signing-vault` (existing private repo).

**Spec:** none written separately — scope and the two binding decisions below were confirmed directly with the user in this session (see chat: TestFlight over Ad Hoc; Fastlane+vault over plain scripts), informed by reading `factoryvu-flutter`'s `docs/superpowers/{specs,plans}/2026-08-05-ios-build-verification-*.md` and `2026-08-06-mobile-signing-toolkit-*.md`.

## Global Constraints

- Bundle identifiers stay exactly `com.rajendrakhope.sevarath` (rider) and `com.rajendrakhope.sevarath.driver` (driver) — identical to the Android `applicationId`/`applicationIdSuffix` already shipping, so this is additive, not a rename.
- iOS deployment target stays `15.0` (already set in `project.pbxproj`) — every dependency (`maplibre_gl`, `firebase_core` 4.x, `google_sign_in` 7.x) supports it.
- Android's signing pipeline (`android/key.properties`, the 5 existing `ANDROID_*`/`GOOGLE_SERVICES_JSON` secrets, `release.yml`) is **not modified** by this plan — it already works.
- No real signing secrets (`.p8` key content, `match` passphrase, API key IDs) are ever typed into or generated-and-stored by an assistant session. Every step touching real secret material is marked **Owner: You** with the exact command to run.
- **Out of scope for this plan** (deferred until sevarath's own docs — `plan.md`/`open-items.md`/`architecture.md` — are brought into this repo, per the user's own stated sequencing): iOS Push Notifications capability/entitlements, Google Sign-In's iOS URL scheme (`GoogleService-Info.plist`'s `REVERSED_CLIENT_ID`), and any App Store (non-TestFlight) production listing. Firebase is touched only to the minimum needed so `Firebase.initializeApp()` doesn't crash on launch (see Task 4) — not to make push or Google Sign-In actually work on iOS yet.
- Every existing test must keep passing unmodified (`flutter test`, `flutter analyze`) — this plan touches native iOS config and CI YAML, not Dart application code.

## Review Focus

- **`Firebase.initializeApp()` with no `GoogleService-Info.plist`** (`lib/main_user.dart:16`, `lib/main_driver.dart:14`) — Firebase's iOS SDK throws synchronously if it can't find the plist in the app bundle, which happens *before* `runApp()`. This is a guaranteed crash-on-launch, not a maybe — Task 4 exists specifically to prevent it, and Task 5's local run is where it gets confirmed fixed.
- **Scheme name vs `--flavor` value mismatch** — Flutter's `--flavor user` only works if a scheme literally named `user` (case-sensitive) exists in `ios/Runner.xcodeproj`. A typo here fails with a Flutter-level error, not an Xcode one — Task 1 Step 6 verifies both flavors build before moving on.
- **`match`'s App Store provisioning profile vs the App ID's registered capabilities** — if a capability (e.g. Push, deferred here) is ever added to the App ID in the portal but the profile isn't regenerated, `match` silently serves a stale profile and the archive fails on a capability mismatch. Not a risk yet (we add no capabilities in this plan), but Task 9's comment flags it for whoever adds push later.
- **Two App Store Connect app records sharing one Fastlane `Appfile`** — `match`'s `app_identifier` must be the exact array `["com.rajendrakhope.sevarath", "com.rajendrakhope.sevarath.driver"]`; passing a bare string here only provisions the first one, and the driver archive fails late (at `upload_to_testflight`, not at `match`) with a confusing "no profile found" error. Task 7 Step 2 and Task 9's lane both use the array form explicitly.
- **Internal TestFlight testers must be added per app record**, not once globally — a tester added to the rider app's internal group cannot install the driver app until added there too. Task 8 Step 4 calls this out so the user doesn't file a false "my test device can't see the build" bug later.

---

## Phase 1 — iOS flavors (no Apple account needed)

### Task 1: Duplicate Xcode build configurations + schemes for `user`/`driver`

**Owner: You** (Xcode GUI — no reliable CLI/scripted path for duplicating build configurations and schemes without risking silent `project.pbxproj` corruption; this mirrors how `factoryvu-flutter`'s plan also routed every Xcode-GUI step through **Owner: You**)

**Files modified (by Xcode, not by hand):**
- `ios/Runner.xcodeproj/project.pbxproj` (6 new `XCBuildConfiguration` entries)
- New: `ios/Runner.xcodeproj/xcshareddata/xcschemes/user.xcscheme`
- New: `ios/Runner.xcodeproj/xcshareddata/xcschemes/driver.xcscheme`

- [ ] **Step 1: Open the project**

```bash
open ios/Runner.xcodeproj
```

(No CocoaPods `Podfile` exists yet in this repo — Flutter 3.44.8 defaults to Swift Package Manager for plugin dependencies when no `Podfile` is present, so open the `.xcodeproj` directly; there's no `.xcworkspace` to prefer over it yet. If `pod install` output or a `Podfile` appears after Task 5's first build, switch to opening `Runner.xcworkspace` instead from then on.)

- [ ] **Step 2: Duplicate the three build configurations, twice each**

Project Navigator → select the top-level **Runner** project (blue icon) → **Info** tab → **Configurations**. For each of `Debug`, `Release`, `Profile`: click the **+** at the bottom → **Duplicate "Debug" Configuration** (etc.) → name the copy `Debug-user`. Repeat so you end up with all 9: `Debug`, `Debug-user`, `Debug-driver`, `Release`, `Release-user`, `Release-driver`, `Profile`, `Profile-user`, `Profile-driver`. (The original 3 stay — Flutter's default `flutter run`/`flutter test` tooling expects them to still exist; only the `-user`/`-driver` ones are new.)

- [ ] **Step 3: Set `PRODUCT_BUNDLE_IDENTIFIER` per new configuration**

Select the **Runner** target (not the project) → **Build Settings** tab → filter for `Product Bundle Identifier`. Set:
- `Debug-user` / `Release-user` / `Profile-user` → `com.rajendrakhope.sevarath`
- `Debug-driver` / `Release-driver` / `Profile-driver` → `com.rajendrakhope.sevarath.driver`
- Leave `Debug`/`Release`/`Profile` (the originals) at `com.rajendrakhope.sevarath` — unused once schemes below exist, but harmless to leave as-is.

- [ ] **Step 4: Add a per-flavor display name**

Still in **Build Settings** → **+** (top-left, "Add Build Setting") → **Add User-Defined Setting** → name it `APP_DISPLAY_NAME`. Set its value per configuration: `SevaRath` for every `Debug/Release/Profile` and `*-user` row, `SevaRath Driver` for every `*-driver` row.

Then open `ios/Runner/Info.plist` (Agent will do this edit — skip ahead to Task 2, Step 1, then come back here) so `CFBundleDisplayName`/`CFBundleName` read `$(APP_DISPLAY_NAME)` instead of the hardcoded `SevaRath` literal.

- [ ] **Step 5: Create the `user` and `driver` schemes**

**Product → Scheme → Manage Schemes…** → select the existing **Runner** scheme → gear icon → **Duplicate**. Rename the duplicate to exactly `user` (this exact, case-sensitive name is what `flutter build ipa --flavor user` matches against). Edit it (pencil icon) → for **Run**, **Test**, **Profile**, **Analyze**, and **Archive**, each has a **Build Configuration** dropdown — set every single one to the matching `*-user` configuration (`Debug-user` for Run/Test, `Release-user` for Archive, `Profile-user` for Profile). Check **Shared** (bottom-left of the scheme editor) so it's written to `xcshareddata/xcschemes/` and committed, not left in per-user `xcuserdata`.

Repeat identically for a `driver` scheme pointing at the `*-driver` configurations.

- [ ] **Step 6: Verify both flavors build** (back in a terminal — this is the actual proof the scheme/config wiring is correct)

```bash
flutter build ios --flavor user -t lib/main_user.dart --no-codesign
flutter build ios --flavor driver -t lib/main_driver.dart --no-codesign
```

Expected: both succeed (`Built build/ios/iphoneos/Runner.app`). If either fails with `Error: The Xcode project does not define custom schemes` or similar, the scheme name in Step 5 doesn't exactly match `user`/`driver` — re-check for a typo or stray capitalization. This may also be the point where Flutter decides it needs CocoaPods after all and creates `ios/Podfile` — if so, that's expected, just let it happen.

- [ ] **Step 7: Commit**

```bash
git add ios/Runner.xcodeproj
git commit -m "Add iOS user/driver flavor build configurations and schemes"
```

---

### Task 2: Wire `APP_DISPLAY_NAME` into `Info.plist`

**Owner: Agent**

**Files:**
- Modify: `ios/Runner/Info.plist`

- [ ] **Step 1: Replace the hardcoded display name**

Current (`Info.plist:9-10,17-18`):
```xml
	<key>CFBundleDisplayName</key>
	<string>SevaRath</string>
```
```xml
	<key>CFBundleName</key>
	<string>SevaRath</string>
```

Replace both `<string>SevaRath</string>` values with `<string>$(APP_DISPLAY_NAME)</string>` (the user-defined build setting from Task 1 Step 4 — Xcode substitutes this at build time per-configuration, same mechanism already driving `$(FLUTTER_BUILD_NAME)`/`$(PRODUCT_BUNDLE_IDENTIFIER)` two lines above/below).

- [ ] **Step 2: Verify the substitution resolves** (requires Task 1 done first)

```bash
flutter build ios --flavor driver -t lib/main_driver.dart --no-codesign
plutil -p build/ios/iphoneos/Runner.app/Info.plist | grep CFBundleDisplayName
```

Expected: `"CFBundleDisplayName" => "SevaRath Driver"` — confirms the driver build picked up its own name, not the rider's.

- [ ] **Step 3: Commit**

```bash
git add ios/Runner/Info.plist
git commit -m "Drive iOS display name from the per-flavor APP_DISPLAY_NAME build setting"
```

---

## Phase 2 — Unblock a non-crashing launch (Firebase minimum)

### Task 3: Register two iOS apps in the existing Firebase project

**Owner: You** (Firebase console — reuses the project already live for Android, `bksevarath`; no new project)

- [ ] **Step 1:** [Firebase console](https://console.firebase.google.com) → project `bksevarath` → **Add app** → iOS → bundle ID `com.rajendrakhope.sevarath`, nickname "SevaRath iOS (rider)". Skip the "Download config file" step for a second — do both registrations first.
- [ ] **Step 2:** Repeat → bundle ID `com.rajendrakhope.sevarath.driver`, nickname "SevaRath iOS (driver)".
- [ ] **Step 3:** Download each app's `GoogleService-Info.plist` (Project Settings → General → scroll to "Your apps" → the iOS app → download icon). You'll get two files with the same filename — rename them locally before the next task, e.g. `GoogleService-Info-user.plist` and `GoogleService-Info-driver.plist`, so they don't overwrite each other.

---

### Task 4: Add both `GoogleService-Info.plist` files to the Xcode project, one per flavor

**Owner: You** (Xcode's "Add Files" dialog is the reliable way to get a resource correctly added to the target's Copy Bundle Resources build phase — a manual `project.pbxproj` edit here risks the same corruption concern as Task 1)

- [ ] **Step 1:** Place both downloaded files somewhere outside the repo temporarily (e.g. `~/Downloads`) — they contain Firebase API keys, which are not secret in the traditional sense (Firebase's own docs confirm these are safe to ship client-side and are not security-sensitive the way a private key is), but for flavor-correctness they still need per-target handling, not a blanket copy.
- [ ] **Step 2:** In Xcode's Project Navigator, right-click the `Runner` folder → **Add Files to "Runner"…** → select `GoogleService-Info-user.plist` → rename it to `GoogleService-Info.plist` during/after add (Xcode's rename, or `mv` in Finder first) → in the file inspector's **Target Membership**, this file must be bundled **only for the `user` configurations**. Xcode's GUI ties Target Membership to targets, not configurations directly — the practical way to get per-flavor resource selection is a **Build Phase script** rather than Target Membership (see Step 4 below); for now, just add the rider one at `ios/Runner/GoogleService-Info.plist` (checked in, since no project secrets live in it) and target membership = Runner.
- [ ] **Step 3:** Add the driver one at a **different path**, e.g. `ios/Runner/GoogleService-Info-driver.plist`, target membership = Runner too (so both ship in the compiled app bundle, harmless — only one is *read* at runtime per Step 4).
- [ ] **Step 4: Add a Run Script build phase that picks the right file per flavor**

Target `Runner` → **Build Phases** → **+** → **New Run Script Phase** → drag it to run **before** "Copy Bundle Resources" → name it "Select flavor GoogleService-Info.plist" → paste:

```bash
if [ "${CONFIGURATION}" == *"-driver" ]; then
  cp "${SRCROOT}/Runner/GoogleService-Info-driver.plist" "${BUILT_PRODUCTS_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/GoogleService-Info.plist"
  echo "Using driver GoogleService-Info.plist"
else
  echo "Using default (rider) GoogleService-Info.plist"
fi
```

(The `*-user` configs already ship the correct rider plist from Step 2's normal Copy Bundle Resources step — this script only needs to override it for `*-driver` builds, copying over whatever Copy Bundle Resources already placed.)

- [ ] **Step 5: Verify per-flavor config selection**

```bash
flutter build ios --flavor user -t lib/main_user.dart --no-codesign
plutil -p build/ios/iphoneos/Runner.app/GoogleService-Info.plist | grep BUNDLE_ID
# expect: com.rajendrakhope.sevarath

flutter build ios --flavor driver -t lib/main_driver.dart --no-codesign
plutil -p build/ios/iphoneos/Runner.app/GoogleService-Info.plist | grep BUNDLE_ID
# expect: com.rajendrakhope.sevarath.driver
```

- [ ] **Step 6: Commit**

```bash
git add ios/Runner/GoogleService-Info.plist ios/Runner/GoogleService-Info-driver.plist ios/Runner.xcodeproj
git commit -m "Add per-flavor Firebase iOS config, selected at build time by configuration"
```

---

## Phase 3 — Local verification (still no Apple Developer Program needed yet)

### Task 5: Run both flavors on a simulator

**Owner: Agent**, with **You** watching the simulator for anything a log can't tell you (visual glitches, a stuck splash, etc.)

- [ ] **Step 1: Boot a simulator and run the rider flavor**

```bash
xcrun simctl boot "iPhone 17 Pro"
flutter run --flavor user -t lib/main_user.dart --dart-define=API_BASE_URL=https://sevarath.rajendrakhope.com/api/v1 -d "iPhone 17 Pro"
```

Expected: app launches past the splash screen to the login screen with no red error screen and no crash — confirms Task 4 actually fixed the `Firebase.initializeApp()` crash risk flagged in this plan's Review Focus. Watch the Flutter console for anything thrown by `pushNotificationService.initialize()` (it's `unawaited`, so it won't crash the app, but a stack trace here is worth noting as a known gap for the deferred push-notifications-on-iOS work, not something to fix in this plan).

- [ ] **Step 2: Repeat for the driver flavor**

```bash
flutter run --flavor driver -t lib/main_driver.dart --dart-define=API_BASE_URL=https://sevarath.rajendrakhope.com/api/v1 -d "iPhone 17 Pro"
```

Expected: same — launches to the driver login flow, home-screen app name reads "SevaRath Driver" (confirms Task 2's display name wiring).

- [ ] **Step 3: Smoke-test login + map rendering**

Log in as a real rider/driver account against the production API (the same backend Android already talks to). Confirm: login succeeds, the campus map (`maplibre_gl`) actually renders tiles (this is the first time this plugin has run on iOS in this codebase — Android's working status doesn't guarantee iOS parity), and location permission prompts appear when expected (pickup resolution for rider, go-online for driver).

No code changes expected from this task unless something breaks — if it does, that's new information for a follow-up task, not something to pre-solve here.

---

### Task 6: Run on a real device (free Apple ID signing)

**Owner: You** (physical device, Apple ID sign-in in Xcode)

- [ ] **Step 1:** Connect your iPhone. In Xcode (`ios/Runner.xcodeproj`), select the `user` scheme → Runner target → Signing & Capabilities → check "Automatically manage signing" → sign in with your Apple ID (Xcode → Settings → Accounts, if not already) → select your personal team (a free Apple ID works for on-device Debug runs, no paid Developer Program needed yet).
- [ ] **Step 2:** Select your iPhone as the run destination → ▶ Run. Confirm the same checklist as Task 5 Step 3, but on real hardware — GPS behaves differently than the simulator's simulated location, and this is the first real signal on whether `geolocator`'s `NSLocationWhenInUseUsageDescription` prompt (already in `Info.plist`) actually fires correctly.
- [ ] **Step 3:** Repeat for the `driver` scheme.

If this works, Phase 1-3 are done: both flavors build, launch, and run real app logic on-device. Everything from here on is about getting a *signed, distributable* build out via CI — not about the app working at all.

---

## Phase 4 — TestFlight distribution setup

### Task 7: Register both App IDs and note the Team ID

**Owner: You** (developer.apple.com — requires the paid Apple Developer Program; if not already enrolled, enrollment itself — $99/year — is a prerequisite this plan can't do for you)

- [ ] **Step 1:** [developer.apple.com](https://developer.apple.com) → **Certificates, IDs & Profiles** → **Identifiers** → **+** → **App IDs** → **App**. Register `com.rajendrakhope.sevarath` (description "SevaRath"). Leave all capabilities unchecked (Push Notifications is deliberately deferred per this plan's Global Constraints).
- [ ] **Step 2:** Repeat for `com.rajendrakhope.sevarath.driver` (description "SevaRath Driver").
- [ ] **Step 3:** Note your **Team ID** (Membership tab, or Xcode's Signing & Capabilities pane) — a 10-character alphanumeric string. You'll need it in Task 9.

(Side benefit: this is the exact same Team ID `factoryvu-flutter`'s own `tools/fastlane/Appfile` has sitting as a `YOUR_TEAM_ID` placeholder, if that's the same Apple Developer account — filling it in there too is optional and outside this plan's scope, but there's no new lookup needed if you want to.)

---

### Task 8: Create both App Store Connect app records + enable internal TestFlight

**Owner: You** (App Store Connect)

- [ ] **Step 1:** [App Store Connect](https://appstoreconnect.apple.com) → **My Apps** → **+** → **New App**. Platform iOS, name "SevaRath", primary language, bundle ID = the `com.rajendrakhope.sevarath` App ID from Task 7, SKU (anything unique, e.g. `sevarath-rider-ios`).
- [ ] **Step 2:** Repeat for "SevaRath Driver" / `com.rajendrakhope.sevarath.driver` / SKU `sevarath-driver-ios`.
- [ ] **Step 3:** For each app → **TestFlight** tab → **Internal Testing** → create a group (e.g. "Internal") → add yourself (and any other internal testers) as a tester. Internal testing needs **no App Review** — a build becomes installable as soon as it finishes processing after upload.
- [ ] **Step 4:** Remember testers are per-app-record — add the same person to *both* apps' internal groups if they need to test both the rider and driver builds (see this plan's Review Focus).

---

### Task 9: Generate one App Store Connect API key (covers both apps, and `factoryvu-flutter` too)

**Owner: You** (App Store Connect — your Apple ID login, not delegable)

- [ ] **Step 1:** App Store Connect → **Users and Access** → **Integrations** → **App Store Connect API** → **Team Keys** → **+**. Name it `mobile-signing-toolkit` (matching the name already planned for `factoryvu-flutter`'s identical key — if one by this name already exists from that other project, reuse it instead of creating a second one; a Team Key authorizes against every app under the team, not just one). Access level **App Manager**.
- [ ] **Step 2:** Download the `.p8` file **immediately** — Apple only allows one download, ever. Save it outside any git repo (password manager attachment).
- [ ] **Step 3:** Note the **Key ID** and **Issuer ID** shown on the same page.

---

## Phase 5 — Fastlane + the existing signing vault

### Task 10: Point the existing `mobile-signing-vault` at sevarath-mobile too

**Owner: Agent**

**Files:**
- Create: `tools/Gemfile`
- Create: `tools/fastlane/Appfile`
- Create: `tools/fastlane/Matchfile`
- Modify: `.gitignore`

**Interfaces:** `Appfile` defines `app_identifier` (an **array** of both bundle IDs — see Review Focus) and `team_id`, consumed by Task 11's `match` call. `Matchfile` points at the same `mobile-signing-vault` repo `factoryvu-flutter` already created — `match` stores material under a path keyed by bundle ID, so one vault safely holds both apps' profiles without collision.

- [ ] **Step 1: Create `tools/Gemfile`**

```ruby
source "https://rubygems.org"

gem "fastlane"
```

- [ ] **Step 2: Create `tools/fastlane/Appfile`**, substituting the real Team ID from Task 7 Step 3:

```ruby
# The file to edit if either bundle ID or the team ever changes.

app_identifier(["com.rajendrakhope.sevarath", "com.rajendrakhope.sevarath.driver"])
team_id("YOUR_TEAM_ID") # from Task 7 Step 3
```

- [ ] **Step 3: Create `tools/fastlane/Matchfile`**

```ruby
git_url("https://github.com/bkrajendra/mobile-signing-vault.git")
storage_mode("git")
type("appstore")
```

(`type("appstore")` — not `"adhoc"` — per this plan's TestFlight decision. App Store-type certs/profiles aren't tied to specific device UDIDs, so there's no `devices.txt` needed here, unlike `factoryvu-flutter`'s Ad Hoc-based plan.)

- [ ] **Step 4: Add secret-material patterns to `.gitignore`**

Add to the end of `.gitignore`:

```
# Fastlane/match signing toolkit — never commit real secret material
tools/.env.local
tools/*.p8
tools/vendor/
tools/.bundle/
```

- [ ] **Step 5: Verify Bundler resolves the Gemfile**

```bash
cd tools && bundle install --path vendor/bundle && cd ..
```

Expected: exits 0, installs `fastlane` into `tools/vendor/bundle` (gitignored).

- [ ] **Step 6: Commit**

```bash
git add tools/Gemfile tools/Gemfile.lock tools/fastlane/Appfile tools/fastlane/Matchfile .gitignore
git commit -m "Point the shared mobile-signing-vault at sevarath-mobile's two iOS bundle IDs"
```

---

### Task 11: Write the Fastfile and two ExportOptions plists

**Owner: Agent**

**Files:**
- Create: `tools/fastlane/Fastfile`
- Create: `ios/ExportOptions-user.plist`
- Create: `ios/ExportOptions-driver.plist`

**Interfaces:**
- Consumes: `Appfile`'s `app_identifier`/`team_id` (Task 10).
- Produces: lanes `ios bootstrap` (local, read-write) and `ios ci_release` (CI, read-only) — consumed by Task 12's local bootstrap run and Task 13's CI workflow.
- Reads env vars `ASC_KEY_ID`, `ASC_ISSUER_ID`, `ASC_KEY_CONTENT_BASE64`, `MATCH_PASSWORD`, `MATCH_GIT_BASIC_AUTHORIZATION` — set locally via `tools/.env.local` (Task 12) or GitHub Actions secrets (Task 13).

- [ ] **Step 1: Create `tools/fastlane/Fastfile`**

```ruby
default_platform(:ios)

platform :ios do
  before_all do
    app_store_connect_api_key(
      key_id: ENV.fetch("ASC_KEY_ID"),
      issuer_id: ENV.fetch("ASC_ISSUER_ID"),
      key_content: ENV.fetch("ASC_KEY_CONTENT_BASE64"),
      is_key_content_base64: true
    )
  end

  desc "Local, read-write: create/fetch the App Store distribution cert + profiles for both bundle IDs"
  lane :bootstrap do
    match(type: "appstore", readonly: false)
  end

  desc "CI, read-only: fetch existing signing material, build both flavors, upload both to TestFlight"
  lane :ci_release do
    match(type: "appstore", readonly: true)

    {
      "user" => { bundle_id: "com.rajendrakhope.sevarath", target: "lib/main_user.dart", export_options: "ExportOptions-user.plist" },
      "driver" => { bundle_id: "com.rajendrakhope.sevarath.driver", target: "lib/main_driver.dart", export_options: "ExportOptions-driver.plist" }
    }.each do |flavor, cfg|
      sh(
        "flutter", "build", "ipa",
        "--release",
        "--flavor", flavor,
        "-t", cfg[:target],
        "--dart-define=API_BASE_URL=https://sevarath.rajendrakhope.com/api/v1",
        "--export-options-plist=../ios/#{cfg[:export_options]}"
      )

      upload_to_testflight(
        api_key: Spaceship::ConnectAPI.token,
        app_identifier: cfg[:bundle_id],
        ipa: Dir.glob("../build/ios/ipa/*.ipa").first,
        skip_waiting_for_build_processing: true
      )

      # flutter build ipa always writes to the same build/ios/ipa/ directory, so
      # it must be cleared between flavors or the second glob above would still
      # match the first flavor's (already-uploaded) .ipa, not the new one.
      FileUtils.rm_rf("../build/ios/ipa")
    end
  end
end
```

- [ ] **Step 2: Create `ios/ExportOptions-user.plist`**, substituting the real Team ID:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>method</key>
	<string>app-store-connect</string>
	<key>teamID</key>
	<string>YOUR_TEAM_ID</string>
	<key>signingStyle</key>
	<string>manual</string>
	<key>signingCertificate</key>
	<string>Apple Distribution</string>
	<key>provisioningProfiles</key>
	<dict>
		<key>com.rajendrakhope.sevarath</key>
		<string>match AppStore com.rajendrakhope.sevarath</string>
	</dict>
</dict>
</plist>
```

(`match AppStore <bundle-id>` is the exact, fixed naming convention `match` uses for App Store-type profiles — not a name you choose, unlike `factoryvu-flutter`'s Ad Hoc plan where `FactoryVU AdHoc` was a human-picked name.)

- [ ] **Step 3: Create `ios/ExportOptions-driver.plist`**, identical but for the driver bundle ID:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>method</key>
	<string>app-store-connect</string>
	<key>teamID</key>
	<string>YOUR_TEAM_ID</string>
	<key>signingStyle</key>
	<string>manual</string>
	<key>signingCertificate</key>
	<string>Apple Distribution</string>
	<key>provisioningProfiles</key>
	<dict>
		<key>com.rajendrakhope.sevarath.driver</key>
		<string>match AppStore com.rajendrakhope.sevarath.driver</string>
	</dict>
</dict>
</plist>
```

- [ ] **Step 4: Verify the Fastfile parses**

```bash
cd tools && bundle exec fastlane lanes && cd ..
```

Expected: exits 0, lists `ios bootstrap` and `ios ci_release` with no Ruby syntax errors (it won't run successfully yet — no API key/`.env.local` exists until Task 12).

- [ ] **Step 5: Commit**

```bash
git add tools/fastlane/Fastfile ios/ExportOptions-user.plist ios/ExportOptions-driver.plist
git commit -m "Add Fastlane ci_release lane building both flavors and uploading to TestFlight"
```

---

### Task 12: Run local bootstrap for real

**Owner: You** (creates real, first-time signing material)

- [ ] **Step 1:** Create `tools/.env.local` (gitignored):

```bash
export ASC_KEY_ID="..."              # from Task 9 Step 3
export ASC_ISSUER_ID="..."           # from Task 9 Step 3
export ASC_KEY_CONTENT_BASE64="$(base64 -i /path/to/AuthKey_XXXXXXXXXX.p8)"   # from Task 9 Step 2
export MATCH_PASSWORD="..."          # pick one yourself, store it in your password manager
```

- [ ] **Step 2:** Run bootstrap:

```bash
cd tools
set -a; source .env.local; set +a
bundle exec fastlane ios bootstrap
cd ..
```

Expected: `match` creates an Apple Distribution certificate (if one doesn't already exist for this team — it may already, since `factoryvu-flutter` is on the same team) and two App Store provisioning profiles (one per bundle ID), pushes them encrypted to `mobile-signing-vault`, and Xcode picks them up automatically next time you open the project.

- [ ] **Step 3:** If prompted for a `MATCH_GIT_BASIC_AUTHORIZATION`-equivalent (a GitHub login for pushing to the vault repo) and you're not already authenticated via SSH/`gh`, follow the prompt — this is local, one-time, and doesn't need to be a long-lived secret (that's Task 13's concern, for CI specifically).

---

### Task 13: Store GitHub Actions secrets and add the CI workflow

**Owner: You** for Step 1 (real secret values); **Owner: Agent** for Step 2 (the workflow file itself contains no secrets)

- [ ] **Step 1:** Create a fine-grained GitHub PAT with **read-only** access to just `mobile-signing-vault`, then:

```bash
gh secret set MATCH_PASSWORD --repo bkrajendra/sevarath-mobile
printf '%s' "$(echo -n 'YOUR_GITHUB_USERNAME:YOUR_PAT' | base64)" | gh secret set MATCH_GIT_BASIC_AUTHORIZATION --repo bkrajendra/sevarath-mobile
gh secret set ASC_KEY_ID --repo bkrajendra/sevarath-mobile
gh secret set ASC_ISSUER_ID --repo bkrajendra/sevarath-mobile
base64 -i /path/to/AuthKey_XXXXXXXXXX.p8 | gh secret set ASC_KEY_CONTENT_BASE64 --repo bkrajendra/sevarath-mobile
gh secret list --repo bkrajendra/sevarath-mobile
```

Expected final command lists all 4 new secrets alongside the 5 existing Android ones (9 total).

- [ ] **Step 2 (Owner: Agent): Create `.github/workflows/ios-testflight.yml`**

```yaml
name: iOS TestFlight

on:
  push:
    branches: [main]
  workflow_dispatch: {}

permissions:
  contents: read

concurrency:
  group: ios-testflight
  cancel-in-progress: true

jobs:
  build-and-upload:
    runs-on: macos-15
    env:
      MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
      MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
      ASC_KEY_ID: ${{ secrets.ASC_KEY_ID }}
      ASC_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
      ASC_KEY_CONTENT_BASE64: ${{ secrets.ASC_KEY_CONTENT_BASE64 }}
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.44.8'
          channel: stable

      - run: flutter pub get

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true
          working-directory: tools

      - name: Build both flavors and upload to TestFlight
        working-directory: tools
        run: bundle exec fastlane ios ci_release
```

(No "combine tag+release" complexity here, unlike Android's `release.yml` — TestFlight's own build-number uniqueness check is the natural gate; every push just ships a new TestFlight build, matching the recommended CI cadence for internal testing.)

- [ ] **Step 3: Commit, push, and watch it run**

```bash
git add .github/workflows/ios-testflight.yml
git commit -m "Add iOS TestFlight CI workflow using Fastlane match"
git push origin main
gh run watch --repo bkrajendra/sevarath-mobile
```

- [ ] **Step 4: Confirm both builds landed**

App Store Connect → each app's **TestFlight** tab → a new build should appear within a few minutes of the workflow finishing (TestFlight processing is async, separate from the upload step succeeding).

---

### Task 14: Install and verify on a real device via TestFlight

**Owner: You**

- [ ] **Step 1:** Install the **TestFlight** app from the App Store on your iPhone (one-time, if not already installed).
- [ ] **Step 2:** Accept the internal-tester invite for both apps (App Store Connect sends an email, or the TestFlight app shows them directly once you're added as an internal tester per Task 8 Step 3).
- [ ] **Step 3:** Install and run both builds, re-confirming Task 6's on-device checklist (login, map rendering, location prompts) — this time against the real CI-built, `match`-signed binary rather than a local Xcode Debug run, closing the loop on "CI produces a working build," not just "my Mac does."

---

Once Task 14 passes, iOS has reached parity with Android's existing CI: push to `main`, both signed flavors ship automatically — to TestFlight instead of a GitHub Release, per this plan's distribution decision. From here, the next planned step (explicitly out of scope above) is bringing `sevarath`'s own `docs/plan.md`/`open-items.md`/`architecture.md` into this repo and tackling iOS push notifications + Google Sign-In for real.
