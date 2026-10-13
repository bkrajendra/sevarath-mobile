# iOS Push Notifications + Google Sign-In Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. This plan mixes **Owner: Agent** tasks (code/Xcode-project edits, no real secrets touched) and **Owner: You** tasks (Apple Developer Portal / Firebase console — never delegate these). Work through tasks in order within each phase; the two phases (Google Sign-In, Push Notifications) are independent of each other and can be done in either order, but Push Notifications' own tasks are sequential (Task 4 depends on Task 3 existing, Task 6 depends on Task 4's portal change).

**Goal:** Make both iOS flavors receive push notifications while backgrounded/killed (with correct deep-linking) and let riders sign in with Google — closing the two gaps `docs/mobile-plan.md`'s Global Constraints explicitly deferred once iOS flavors existed.

**Architecture:** The backend needs **zero changes** — confirmed by reading `sevarath`'s `docs/open-items.md` #31/#34/#35/#36: Firebase Cloud Messaging bridges to APNs automatically server-side (`admin.messaging().send()` already reaches iOS devices once a device token is registered), and the full push pipeline (outbox → BullMQ → `PushNotificationConsumer`) already fans out correctly regardless of platform. This plan is entirely iOS-project and Flutter-client work: Xcode capabilities (Push Notifications, Background Modes), an APNs Authentication Key uploaded to Firebase, a `match` provisioning-profile regeneration (the exact risk flagged in `docs/mobile-plan.md`'s Review Focus — a capability added to an App ID after `match` already fetched a profile for it), one real Dart bug fix (`flutter_local_notifications`' iOS init crash, already observed and deferred during that plan's Task 5), and a per-flavor Google Sign-In URL scheme.

**Tech Stack:** Same as `docs/mobile-plan.md` — Flutter 3.47.x, Xcode 27, `xcodeproj` Ruby gem for project-file edits, Fastlane `match` (already bootstrapped, vault at `bkrajendra/mobile-signing-vault`), Firebase project `bksevarath`.

**Spec:** `sevarath`'s `docs/plan.md` §5 (Phase 10.2 "Firebase push: receive + deep-link (Android)", Phase 10.5 "Google Sign-In (rider app only)") describe the already-built Android implementation this plan brings to iOS feature parity with; `docs/open-items.md` rows #31/#34/#35/#36 confirm the backend side; `docs/architecture.md` §7.2/§9.4 give the system-level framing. None of these describe iOS specifically — this plan is the iOS-side argument, informed by reading all four.

## Global Constraints

- Bundle identifiers and Team ID are unchanged: `com.rajendrakhope.sevarath` / `.driver`, Team `3KA57X722P`.
- Google Sign-In stays **rider-only** — `FlavorConfig.isDriver`-gated in the UI already; this plan doesn't touch that gating, only makes the iOS side of the existing flow work.
- Driver accounts stay admin-provisioned, mobile+password only (`docs/plan.md` §5.6) — out of scope here, unaffected.
- No backend code changes — every file this plan touches lives in `sevarath-mobile` (`ios/`, `lib/`, `tools/`). If a task seems to need a backend change, stop and re-read `open-items.md` #34 first; it almost certainly doesn't.
- Real signing/API secrets (the APNs `.p8` key, Firebase service credentials) are never typed into or handled by an assistant session — every step touching one is **Owner: You** with the exact portal path.
- `aps-environment` in the committed entitlements files stays `development` — Xcode's own archive/export step remaps it to `production` automatically based on the provisioning profile's type (App Store profiles are production-only); hand-setting it to `production` in source would break Debug builds on a development-profile device.
- Every existing test must keep passing (`flutter test`, `flutter analyze`).

## Review Focus

- **`flutter_local_notifications.initialize()` without `DarwinInitializationSettings`** (`lib/features/notifications/push_notification_service.dart:53`) — this is the exact exception observed and explicitly deferred during `docs/mobile-plan.md`'s Task 5 real-device/simulator runs ("iOS settings must be set when targeting iOS platform"). Task 7 fixes it; Task 7's own verification step is where this gets confirmed gone, the same way it was confirmed present.
- **Double foreground-notification on iOS** — the existing `_showForegroundNotification` handler manually calls `_localNotifications.show(...)` on every `onMessage` event, which exists specifically because Android doesn't auto-display a system notification while foregrounded. iOS does, via `setForegroundNotificationPresentationOptions`. Calling both unconditionally would show two notifications for one push on iOS. Task 7 gates the manual path to Android only.
- **A capability added to the App ID after `match` already has a profile for it** — this is `docs/mobile-plan.md`'s own Review Focus line, now the actual scenario. Task 6 exists specifically to regenerate the profile; skipping it means the next `ci_release` archive fails on a capability mismatch, not silently.
- **Double permission prompt on iOS** — `flutter_local_notifications`' `DarwinInitializationSettings` can itself request alert/badge/sound permission, and `FirebaseMessaging.instance.requestPermission()` (already called later in the same `initialize()`) requests the same underlying `UNUserNotificationCenter` authorization. Task 7 sets `requestAlertPermission`/`requestBadgePermission`/`requestSoundPermission` to `false` on the Darwin settings so only one prompt appears.
- **Google Sign-In's OAuth redirect has nowhere to go without the URL scheme** — without `CFBundleURLTypes` registering the `REVERSED_CLIENT_ID`, the web-based sign-in flow has no way to hand control back to the app; it hangs or the OS shows "Safari cannot open the page" instead of a Flutter-level error, which is a confusing failure mode to debug blind. Task 1's own verification step catches this before it ships.

---

## Phase 1 — Google Sign-In iOS wiring

### Task 1: Register each flavor's `REVERSED_CLIENT_ID` as a URL scheme

**Owner: Agent**

**Files:**
- Modify: `ios/Runner.xcodeproj/project.pbxproj` (new `GOOGLE_REVERSED_CLIENT_ID` user-defined build setting, one value per flavor — same pattern as `APP_DISPLAY_NAME` from `docs/mobile-plan.md` Task 1)
- Modify: `ios/Runner/Info.plist`

**Interfaces:** Consumes the `REVERSED_CLIENT_ID` values already present in `ios/Runner/GoogleService-Info.plist` (rider) and `ios/Runner/GoogleService-Info-driver.plist` (driver) — both committed, non-secret (Firebase API keys are not security-sensitive). No code interface; this is pure Xcode/Info.plist config.

- [ ] **Step 1: Read the two REVERSED_CLIENT_ID values**

```bash
plutil -p ios/Runner/GoogleService-Info.plist | grep REVERSED_CLIENT_ID
plutil -p ios/Runner/GoogleService-Info-driver.plist | grep REVERSED_CLIENT_ID
```

Expected output (already confirmed this session, reproduced here so the values travel with the plan):
- rider: `com.googleusercontent.apps.1010698548403-adhek2kaiqetvhj5vmeelsnpsnmieqdf`
- driver: `com.googleusercontent.apps.1010698548403-gdnf4rg14h1djemqsch0n2mh496ivihb`

- [ ] **Step 2: Add the `GOOGLE_REVERSED_CLIENT_ID` build setting per flavor**

Using the `xcodeproj` Ruby gem (already a transitive CocoaPods dependency, confirmed working this session — see `docs/mobile-plan.md`'s Task 4 ledger for why this is preferred over hand-editing `project.pbxproj` text):

```ruby
require 'xcodeproj'

project = Xcodeproj::Project.open('ios/Runner.xcodeproj')
runner_target = project.targets.find { |t| t.name == 'Runner' }

values = {
  'Debug-user' => 'com.googleusercontent.apps.1010698548403-adhek2kaiqetvhj5vmeelsnpsnmieqdf',
  'Release-user' => 'com.googleusercontent.apps.1010698548403-adhek2kaiqetvhj5vmeelsnpsnmieqdf',
  'Profile-user' => 'com.googleusercontent.apps.1010698548403-adhek2kaiqetvhj5vmeelsnpsnmieqdf',
  'Debug-driver' => 'com.googleusercontent.apps.1010698548403-gdnf4rg14h1djemqsch0n2mh496ivihb',
  'Release-driver' => 'com.googleusercontent.apps.1010698548403-gdnf4rg14h1djemqsch0n2mh496ivihb',
  'Profile-driver' => 'com.googleusercontent.apps.1010698548403-gdnf4rg14h1djemqsch0n2mh496ivihb'
}

runner_target.build_configurations.each do |config|
  next unless values.key?(config.name)
  config.build_settings['GOOGLE_REVERSED_CLIENT_ID'] = values[config.name]
end

project.save
```

- [ ] **Step 3: Verify the setting landed on all 6 flavored configs**

```bash
grep -c "GOOGLE_REVERSED_CLIENT_ID" ios/Runner.xcodeproj/project.pbxproj
```

Expected: `6` (Debug/Release/Profile × user/driver — the un-flavored `Debug`/`Release`/`Profile` configs are skipped on purpose, same as `APP_DISPLAY_NAME`).

- [ ] **Step 4: Add the URL scheme to `Info.plist`**

Add this block to `ios/Runner/Info.plist`, as a sibling of the existing top-level keys (e.g. right after `CFBundleIdentifier`):

```xml
	<key>CFBundleURLTypes</key>
	<array>
		<dict>
			<key>CFBundleTypeRole</key>
			<string>Editor</string>
			<key>CFBundleURLSchemes</key>
			<array>
				<string>$(GOOGLE_REVERSED_CLIENT_ID)</string>
			</array>
		</dict>
	</array>
```

- [ ] **Step 5: Verify the substitution resolves per flavor**

```bash
flutter build ios --flavor user -t lib/main_user.dart --no-codesign
plutil -p build/ios/iphoneos/Runner.app/Info.plist | grep -A2 CFBundleURLSchemes
# expect: com.googleusercontent.apps.1010698548403-adhek2kaiqetvhj5vmeelsnpsnmieqdf

flutter build ios --flavor driver -t lib/main_driver.dart --no-codesign
plutil -p build/ios/iphoneos/Runner.app/Info.plist | grep -A2 CFBundleURLSchemes
# expect: com.googleusercontent.apps.1010698548403-gdnf4rg14h1djemqsch0n2mh496ivihb
```

- [ ] **Step 6: Commit**

```bash
git add ios/Runner.xcodeproj/project.pbxproj ios/Runner/Info.plist
git commit -m "Register each flavor's Google Sign-In URL scheme on iOS"
```

---

### Task 2: Verify the real Google Sign-In flow on iOS

**Owner: You** (needs a real Google account interacting with a real OAuth flow — not something to script)

- [ ] **Step 1:** Run the rider flavor on a simulator or your iPad:

```bash
flutter run --flavor user -t lib/main_user.dart --dart-define=API_BASE_URL=https://sevarath.rajendrakhope.com/api/v1 -d <device-id>
```

- [ ] **Step 2:** On the login screen, tap **Continue with Google**. Expected: a native Google sign-in sheet appears (not a Safari "cannot open page" error — that specific failure means Task 1's URL scheme didn't take effect, check Step 5's verification again).
- [ ] **Step 3:** Complete sign-in with a real Google account. Two outcomes are both correct, matching the design in `sevarath`'s `docs/plan.md` §5.5:
  - An account already linked by mobile/email logs straight in.
  - A brand-new Google identity is prompted for a mandatory mobile number (`CompleteProfileScreen`) before any account is created.
- [ ] **Step 4:** Confirm the **driver** flavor's login screen still has **no** "Continue with Google" button (unchanged, rider-only gating) — this isn't a new check, just confirming Task 1 didn't accidentally affect it.

---

## Phase 2 — Push Notifications

### Task 3: Add Push Notifications + Background Modes capabilities in Xcode

**Owner: Agent**

**Files:**
- Create: `ios/Runner/user.entitlements`
- Create: `ios/Runner/driver.entitlements`
- Modify: `ios/Runner.xcodeproj/project.pbxproj` (`CODE_SIGN_ENTITLEMENTS` per flavor config)
- Modify: `ios/Runner/Info.plist` (`UIBackgroundModes`)

**Interfaces:** Produces the two `.entitlements` files Task 6's `match` regeneration needs to match against (the App ID capability from Task 4, not the entitlements file itself, is what `match` actually checks — but Xcode's own build will fail to compile without `CODE_SIGN_ENTITLEMENTS` pointing at a file that declares `aps-environment` once the capability exists on the target).

- [ ] **Step 1: Create both entitlements files**

`ios/Runner/user.entitlements`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>aps-environment</key>
	<string>development</string>
</dict>
</plist>
```

`ios/Runner/driver.entitlements`: identical content (the entitlement itself doesn't encode the bundle ID — only which provisioning profile gets used to sign does that).

- [ ] **Step 2: Wire `CODE_SIGN_ENTITLEMENTS` and `UIBackgroundModes` via the `xcodeproj` gem**

```ruby
require 'xcodeproj'

project = Xcodeproj::Project.open('ios/Runner.xcodeproj')
runner_target = project.targets.find { |t| t.name == 'Runner' }

entitlements = {
  'Debug-user' => 'Runner/user.entitlements',
  'Release-user' => 'Runner/user.entitlements',
  'Profile-user' => 'Runner/user.entitlements',
  'Debug-driver' => 'Runner/driver.entitlements',
  'Release-driver' => 'Runner/driver.entitlements',
  'Profile-driver' => 'Runner/driver.entitlements'
}

runner_target.build_configurations.each do |config|
  next unless entitlements.key?(config.name)
  config.build_settings['CODE_SIGN_ENTITLEMENTS'] = entitlements[config.name]
end

project.save
```

- [ ] **Step 3: Add `UIBackgroundModes` to `Info.plist`**

```xml
	<key>UIBackgroundModes</key>
	<array>
		<string>remote-notification</string>
	</array>
```

- [ ] **Step 4: Verify both flavors still build** (will **not** yet deliver push — the App ID doesn't have the capability enabled until Task 4, so this only proves the project compiles with the new entitlements wiring)

```bash
flutter build ios --flavor user -t lib/main_user.dart --no-codesign
flutter build ios --flavor driver -t lib/main_driver.dart --no-codesign
```

Expected: both succeed. If either fails with a code-signing/entitlements error even with `--no-codesign`, check the entitlements file path in Step 2 is exactly `Runner/user.entitlements` (relative to the `ios/` directory, matching how `INFOPLIST_FILE` is already referenced in the same build settings).

- [ ] **Step 5: Commit**

```bash
git add ios/Runner/user.entitlements ios/Runner/driver.entitlements ios/Runner.xcodeproj/project.pbxproj ios/Runner/Info.plist
git commit -m "Add Push Notifications entitlements and Background Modes per iOS flavor"
```

---

### Task 4: Enable Push Notifications on both App IDs

**Owner: You** (developer.apple.com)

- [ ] **Step 1:** [developer.apple.com](https://developer.apple.com) → Certificates, IDs & Profiles → Identifiers → `com.rajendrakhope.sevarath` → check **Push Notifications** under Capabilities → Save.
- [ ] **Step 2:** Repeat for `com.rajendrakhope.sevarath.driver`.

---

### Task 5: Generate an APNs Authentication Key and upload it to Firebase

**Owner: You** — this is the one genuinely new Apple/Firebase artifact this plan needs; everything else reuses what already exists.

- [ ] **Step 1:** [developer.apple.com](https://developer.apple.com) → Certificates, IDs & Profiles → Keys → **+** → name it e.g. `sevarath-apns` → check **Apple Push Notifications service (APNs)** → Continue → Register → **Download** (one-time only, same rule as the App Store Connect API key — save it outside any git repo). Note the **Key ID** shown.
- [ ] **Step 2:** [Firebase console](https://console.firebase.google.com) → project `bksevarath` → ⚙️ Project Settings → **Cloud Messaging** tab → under **Apple app configuration**, for the rider app entry (`com.rajendrakhope.sevarath`) → **APNs Authentication Key** → **Upload** → select the `.p8` from Step 1, enter its Key ID and your Team ID (`3KA57X722P`).
- [ ] **Step 3:** Repeat Step 2 for the driver app entry (`com.rajendrakhope.sevarath.driver`) — the same key file/Key ID is reused (APNs keys are account-wide, not per-app), Firebase just needs it registered against each app entry separately.

This key is the **entire** Firebase/Apple-side requirement — per `sevarath`'s `open-items.md` #34, the NestJS backend calls a single `admin.messaging().send()` for both platforms with no APNs-specific code of its own; FCM relays to APNs using this key.

---

### Task 6: Regenerate `match`'s provisioning profiles

**Owner: You** — this is the scenario `docs/mobile-plan.md`'s Review Focus flagged in advance: the App ID now has a capability (Task 4) that the profile `match` fetched before that didn't.

- [ ] **Step 1:**

```bash
cd tools
export PATH="/Users/rajendrakhope/.gem/ruby/4.0.0/bin:/opt/homebrew/lib/ruby/gems/4.0.0/bin:$PATH"
export LANG=en_US.UTF-8
set -a; source .env.local; set +a
fastlane ios bootstrap
```

Expected: `match` detects the App Store profiles for both bundle IDs no longer match their App IDs' current capabilities and regenerates both, pushing the updated encrypted profiles to `mobile-signing-vault`. If it instead reports no changes needed, re-check Task 4 actually saved on both App IDs before re-running.

---

## Phase 3 — Fix the deferred Dart bug

### Task 7: Fix `flutter_local_notifications`' iOS initialization and avoid double foreground notifications

**Owner: Agent**

**Files:**
- Modify: `lib/features/notifications/push_notification_service.dart`

**Interfaces:** No change to `PushNotificationService`'s public surface (`initialize()`, `registerDeviceToken()`, `unregisterDeviceToken()` keep their existing signatures — `main_user.dart`/`main_driver.dart`/`auth_provider.dart` call these unchanged).

- [ ] **Step 1: Update the stale doc comment**

Current (`push_notification_service.dart:10-13`):
```dart
/// Android-only receive + deep-link half of Phase 7's push pipeline (Phase 10.2, docs/plan.md
/// §5) - the backend send side (including the driver ride-offer push, `RideDriverNotified` ->
/// `offeredDriver` in push-recipients.ts) was already fully built; nothing in `apps/mobile`
/// called any of it until now. iOS is out of scope - no iOS flavor split exists yet.
```

Replace with:
```dart
/// Receive + deep-link half of Phase 7's push pipeline (Phase 10.2, docs/plan.md §5), now
/// covering both Android and iOS (docs/ios-push-and-google-signin-plan.md) - the backend send
/// side (including the driver ride-offer push, `RideDriverNotified` -> `offeredDriver` in
/// push-recipients.ts) needs no platform-specific code of its own; FCM relays to APNs for iOS.
```

- [ ] **Step 2: Add the Darwin (iOS) initialization settings**

Current (`push_notification_service.dart:51-57`):
```dart
    await _localNotifications.initialize(
      settings: const InitializationSettings(android: AndroidInitializationSettings('@mipmap/ic_launcher')),
      onDidReceiveNotificationResponse: (response) {
        final rideId = response.payload;
        if (rideId != null) _navigate(eventType: null, rideId: rideId);
      },
    );
```

Replace with:
```dart
    await _localNotifications.initialize(
      settings: const InitializationSettings(
        android: AndroidInitializationSettings('@mipmap/ic_launcher'),
        // Permission is requested once, below, via FirebaseMessaging.instance.requestPermission()
        // - both plugins share the same underlying UNUserNotificationCenter authorization on
        // iOS, so requesting it here too would show a second, redundant system prompt.
        iOS: DarwinInitializationSettings(
          requestAlertPermission: false,
          requestBadgePermission: false,
          requestSoundPermission: false,
        ),
      ),
      onDidReceiveNotificationResponse: (response) {
        final rideId = response.payload;
        if (rideId != null) _navigate(eventType: null, rideId: rideId);
      },
    );
```

- [ ] **Step 3: Tell iOS to present foreground pushes natively, right after the existing permission request**

Current (`push_notification_service.dart:60-61`):
```dart
    // Also requests the Android 13+ POST_NOTIFICATIONS runtime permission.
    await FirebaseMessaging.instance.requestPermission();
```

Replace with:
```dart
    // Also requests the Android 13+ POST_NOTIFICATIONS runtime permission, and on iOS is what
    // actually triggers the system permission prompt (DarwinInitializationSettings above
    // deliberately doesn't, to avoid asking twice).
    await FirebaseMessaging.instance.requestPermission();

    // iOS-only: without this, a push received while the app is foregrounded is delivered
    // silently (no banner) - Android doesn't need the equivalent because _showForegroundNotification
    // below handles that case manually there instead.
    await FirebaseMessaging.instance.setForegroundNotificationPresentationOptions(
      alert: true,
      badge: true,
      sound: true,
    );
```

- [ ] **Step 4: Stop double-showing foreground notifications on iOS**

Current (`push_notification_service.dart:94-105`):
```dart
  void _showForegroundNotification(RemoteMessage message) {
    final notification = message.notification;
    if (notification == null) return;
    // firebase_messaging alone doesn't surface a system notification while the app is
    // foregrounded on Android - flutter_local_notifications fills that gap.
    _localNotifications.show(
      id: notification.hashCode,
      title: notification.title,
      body: notification.body,
      notificationDetails: const NotificationDetails(
        android: AndroidNotificationDetails(_rideUpdatesChannelId, _rideUpdatesChannelName, importance: Importance.high),
      ),
      payload: message.data['rideId'] as String?,
    );
  }
```

Replace with:
```dart
  void _showForegroundNotification(RemoteMessage message) {
    // iOS already shows this via setForegroundNotificationPresentationOptions (set in
    // initialize()) - calling _localNotifications.show() here too would duplicate it. Only
    // Android needs the manual path, which is why this whole method exists in the first place.
    if (defaultTargetPlatform != TargetPlatform.android) return;

    final notification = message.notification;
    if (notification == null) return;
    _localNotifications.show(
      id: notification.hashCode,
      title: notification.title,
      body: notification.body,
      notificationDetails: const NotificationDetails(
        android: AndroidNotificationDetails(_rideUpdatesChannelId, _rideUpdatesChannelName, importance: Importance.high),
      ),
      payload: message.data['rideId'] as String?,
    );
  }
```

Add the needed import at the top of the file (`defaultTargetPlatform` lives in Flutter's foundation library, already indirectly available via the `flutter/foundation.dart` that `debugPrint` comes from — check the existing `import 'package:flutter/foundation.dart';` line is already there before adding a duplicate).

- [ ] **Step 5: Verify — simulator first (both flavors, confirms the crash is gone)**

```bash
flutter run --flavor user -t lib/main_user.dart --dart-define=API_BASE_URL=https://sevarath.rajendrakhope.com/api/v1 -d <simulator-id>
```

Expected: no `[ERROR:flutter/runtime/dart_vm_initializer.cc] Unhandled Exception: Invalid argument(s): iOS settings must be set when targeting iOS platform` in the console — this exact line is what `docs/mobile-plan.md`'s Task 5 observed and deferred; its absence here is the fix confirmed. Repeat for the driver flavor.

- [ ] **Step 6: `flutter analyze` and `flutter test`**

```bash
flutter analyze
flutter test
```

Expected: no new issues, 25/25 tests passing (unchanged — this task touches no tested logic, only platform-glue wiring).

- [ ] **Step 7: Commit**

```bash
git add lib/features/notifications/push_notification_service.dart
git commit -m "Fix iOS crash in flutter_local_notifications init, avoid double foreground pushes"
```

---

### Task 8: Verify a real push reaches a real iOS device

**Owner: You** — the one check nothing in this plan can substitute for. Per `sevarath`'s `open-items.md` #36, **no push notification has ever been verified against a real device from the backend side either** — this is genuinely the first real-device confirmation for the whole pipeline, not just the iOS half.

- [ ] **Step 1:** Run the rider flavor on your real iPad (not the simulator — APNs doesn't deliver to simulators prior to iOS 16's limited simulator-push support, and this plan isn't relying on that), log in, confirm the push-permission system prompt appears once (not twice — confirms Task 7 Step 2/3's double-prompt fix).
- [ ] **Step 2:** Background the app (don't force-quit yet). From the [Firebase console](https://console.firebase.google.com) → project `bksevarath` → Engage → Messaging → **New campaign** → Notifications → send a test message targeting this device's FCM token (Settings → "Send test message" lets you paste a token directly — get it by temporarily adding a `debugPrint('FCM token: $token')` in `registerDeviceToken()`, or read it from Xcode's device console). Confirm a system notification banner appears.
- [ ] **Step 3:** Tap the notification. Confirm the app opens to Home (the generic rider-facing path, since a Firebase-console test message won't carry the `eventType`/`rideId` data payload a real ride event would).
- [ ] **Step 4:** The more meaningful test: trigger a **real** ride-lifecycle push by actually booking a ride end-to-end (rider requests, a driver on the driver-flavor app accepts) and confirming the rider receives the "driver assigned" push while backgrounded, and the driver receives the ride-offer push (`RideDriverNotified`) while backgrounded too — tap it and confirm it deep-links to `IncomingOfferScreen`, not just Home.
- [ ] **Step 5:** Foreground test — with the app open, trigger another real ride event and confirm exactly **one** notification banner appears (not two — Task 7 Step 4's fix).
