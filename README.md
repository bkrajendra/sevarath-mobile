# SevaRath Mobile

The SevaRath rider+driver Flutter app. Split out of [`bkrajendra/sevarath`](https://github.com/bkrajendra/sevarath) on 2026-10-12 - see that repo's [`docs/repo-split-plan.md`](https://github.com/bkrajendra/sevarath/blob/main/docs/repo-split-plan.md) for why and how.

System-level docs (architecture, API contract, product plan) stay in the main repo: [`sevarath/docs`](https://github.com/bkrajendra/sevarath/tree/main/docs).

## Releases

Push a commit to `main` that bumps `version:` in `pubspec.yaml` - the `Mobile Release` workflow creates the matching `vX.Y.Z` tag automatically and builds+publishes signed rider+driver APKs to that tag's GitHub Release. A push that doesn't change the version is a no-op. The same workflow can also be run manually (`workflow_dispatch`, with an existing `tag_name`) to re-release a tag, e.g. after fixing secrets or retrying a failed build.

## Getting Started

```bash
flutter pub get
flutter analyze
flutter test
```

A few resources if this is your first Flutter project:

- [Learn Flutter](https://docs.flutter.dev/get-started/learn-flutter)
- [Write your first Flutter app](https://docs.flutter.dev/get-started/codelab)
