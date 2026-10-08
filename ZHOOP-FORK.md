# Fork maintenance — Zhoop on NOOP

This checkout is `hackyguru/zhoop` (a personal weight-loss fork of NOOP) replayed onto current
upstream `ryanbr/noop`. Upstream documentation under `docs/`, `AGENTS.md` and `CHANGELOG.md` is
NOOP's own and describes NOOP; this file covers only what is specific to carrying the fork.

## Branches and remotes

| Ref | What it is |
|---|---|
| `main` | `hackyguru/zhoop` as published: one commit on an upstream base from 2026-09-24. |
| `zhoop-on-upstream` | That commit replayed onto current `upstream/main`. Build from here. |
| `origin` | `https://github.com/hackyguru/zhoop.git` |
| `upstream` | `https://github.com/ryanbr/noop.git` |

The fork's entire contribution is one commit (`4eb04683`): a `NOOP`→`Zhoop` rebrand across 85 files,
plus `StrandiOS/Cut/` (`CutPlanStore`, `CutTodayView`, `CutSleepView`) and a two-tab `RootTabView`.
Everything else is upstream's.

## Carrying it forward onto a newer upstream

```bash
git fetch upstream
git checkout -b zhoop-on-upstream-$(date +%Y%m%d) upstream/main
git cherry-pick 4eb04683
```

Six files conflict, every time, because the rebrand touches strings upstream keeps editing. Resolve
as follows rather than by taking one side wholesale:

- **`Strand/BLE/LiveState.swift`, `StrandiOS/Widgets/LiveActivityController.swift`,
  `StrandiOS/Widgets/SyncLiveActivityController.swift`** — keep upstream's logic, re-apply the brand
  rename to the string literal only. The fork's side carries older code (e.g. `\(v)` where upstream
  now uses `Self.appIdentityLine`).
- **`Strand/Screens/SettingsView.swift`** — drop the fork's Live Activity toggle block. Upstream
  replaced it with the `liveNotificationSwitch` section; taking the fork's side duplicates both
  toggles.
- **`project.yml`** — keep upstream's `BGTaskSchedulerPermittedIdentifiers` additions, rename the
  `NS*UsageDescription` strings.
- **`README.md`** — take the fork's version whole. It is the fork's own document.

Then sweep upstream's new user-facing strings, which the fork has never seen:

```bash
grep -rn '"[^"]*NOOP[^"]*"' --include='*.swift' Strand StrandiOS StrandiOSShared StrandiOSWidgets
```

**Leave as `NOOP` on purpose:** the About screen's "Built on NOOP" credit and its accessibility
label, `ProjectInfo.swift`'s attribution row, code comments, the `NOOPIMU2` file magic in
`ImuSessionFileStore`, `NOOPWidget`'s `kind` string, every bundle identifier, and the `.noopbak`
backup format. Renaming any of those breaks compatibility or removes credit the licence requires.

## Known gaps inherited from the fork

- **286 renamed strings are absent from the String Catalogs.** The fork renamed Swift literals but
  left every `.xcstrings` untouched, so `Tools/i18n_audit.py --ci` fails on `main` and on
  `zhoop-on-upstream` alike. English is unaffected (SwiftUI falls back to the literal key); German,
  French, Spanish, Italian and Russian show English for those strings. Reseeding needs
  `Tools/seed-string-catalog.py`, which reads `.stringsdata` from an Xcode build and therefore needs
  macOS.
- **`UpdateChecker` points at a repository with no releases.** `Strand/System/UpdateChecker.swift`
  queries `api.github.com/repos/hackyguru/noop/releases/latest`. That slug redirects to
  `hackyguru/zhoop` correctly, but the fork publishes no releases, so the check always 404s. The
  Android half still points at `ryanbr/noop`.
- **`CutTodayView.seedPlanIfNeeded()` hardcodes the fork author's body stats** — 83.3 kg, 180 cm,
  male, age 24, goal 75 kg — and writes them into the profile on first launch without asking. Set
  your own under *Edit plan* on the Today screen, or change the defaults in
  `StrandiOS/Cut/CutTodayView.swift`.
- **`AppChangelog.swift`'s "Zhoop now lives at noop.fans" entry is upstream history**, caught by the
  blanket rename. Zhoop does not live there.
- **`CutTodayView` reloads every 60s and reads up to 120 days of HR samples per pass.** Local SQLite
  only, but it costs battery.

## Building for iPhone

`xcodegen generate`, then the `NOOPiOS` scheme — Xcode on macOS. Without a Mac, dispatch
`.github/workflows/fork-testing-build.yml` from the Actions tab of a repository you own: its `ios`
job builds Release on a `macos-26` runner with `CODE_SIGNING_ALLOWED=NO` and attaches
`NOOP-ios-unsigned-v<ver>.ipa` to a prerelease. Sign that on-device with AltStore or SideStore under
your own free Apple ID; re-signing is every 7 days and the sideloader automates it. Keep
`NOOPWidgets.appex` when the sideloader asks, or widgets and Live Activities stop rendering.

`Config/BundleIdSecrets.xcconfig` (gitignored) sets `BUNDLE_ID_PREFIX` so this build can sit beside
an upstream NOOP install instead of colliding with it. A CI build does not see that file and falls
back to `com.noopapp`; to use your own prefix in CI, write the file in the workflow before
`xcodegen generate`.

## Privacy switches worth setting

Upstream is offline by default, with these exceptions:

- **The release check runs daily and is on by default.** Settings → About turns it off. It sends
  nothing about you, but it does reveal your IP to GitHub once a day.
- **Leave the AI Coach unconfigured** unless you want it. It is off until you supply your own API
  key, and the biometric payload needs a second explicit consent. Note that coach replies render
  through MarkdownUI with no `imageProvider` override, so a reply containing `![](url)` fetches that
  URL; a one-line `.markdownImageProvider` override on `CoachView.swift`'s `Markdown(...)` closes it.
- **Android's self-hosted push is the only path that exports health data, GPS routes included.** It
  is default-off and needs both a URL and a bearer token. Irrelevant to an iPhone build.
