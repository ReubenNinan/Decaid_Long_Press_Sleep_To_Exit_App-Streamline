# Proposal: Long-press Sleep to exit Decaid

A proposed change for **Decaid** and the **Streamline.js** skin that brings back
a de1app feature: hold **Sleep** to put the machine to sleep and leave the app,
so the tablet can be used normally while the machine stays asleep.

> **Status: merged into Decaid.** `window.decentApp.exitApp()` landed in
> [decentespresso/decaid#1134](https://github.com/decentespresso/decaid/pull/1134)
> (issue [#1035](https://github.com/decentespresso/decaid/issues/1035)). Once
> your Decaid release includes it, this skin's long press exits the app. On
> older releases, holding Sleep just puts the machine to sleep.

## The problem

In the de1app, long-pressing Sleep put the machine to sleep **and** closed the
app. Many owners use the tablet for other things between coffees and relied on
this.

In Decaid, Sleep puts the machine to sleep and shows the skin's screensaver.
To use the tablet, you have to leave Decaid by hand with the Home gesture. No
add-on can fix this today:

- **Skins** are web pages. Their only way out is `exitToDashboard()`, which
  stays inside Decaid. Android intent links, which a web page would need to
  reach the home screen, are blocked in the skin view.
- **Plugins** run in a sandbox, with no access to the app or the screen, and
  can't reach other apps on the tablet over HTTP.

## The proposed solution

Two small changes: one in Decaid, one in the skin.

### 1. Decaid: `window.decentApp.exitApp()`

A new function in the skin API, next to the existing `exitToDashboard()`. It
uses the same safety design:

- The skin navigates to `/__decent/exit-app?nonce=<nonce>`. Decaid cancels that
  navigation and acts on it only with the per-view nonce it injects into the
  skin's main frame, the same protection as `switchSkin()`. Embedded pages from
  other sites, subframes, other origins and modified URLs are rejected.
- **Android:** `moveTaskToBack(true)`, the same as pressing Home. Decaid keeps
  running in the background, so the machine connection and scheduled wake times
  keep working, and reopening Decaid returns straight to the skin. The screen
  wake lock and the skin's dimmed brightness apply only to Decaid's own window,
  so the tablet behaves normally once you leave.
- **macOS, Windows, Linux:** quit the app, through the existing exit path.
- **iOS:** does nothing, because apps aren't allowed to close themselves.
- **External browsers:** does nothing, the same as `exitToDashboard()`.

Skins that don't call it are unaffected.

**Patch:** [`patches/1-decaid-exitApp.patch`](patches/1-decaid-exitApp.patch),
about 230 lines including tests and docs, against
[decentespresso/decaid](https://github.com/decentespresso/decaid).

| File | Change |
|---|---|
| `lib/src/webui_support/webui_service.dart` | Adds `exitApp()` to the skin API script and the exit-app URL |
| `lib/src/skin_feature/skin_view.dart` | Routes the exit-app URL, checks its nonce, and backgrounds or quits by platform |
| `android/.../MainActivity.kt` | Adds `moveTaskToBack` to the existing `app/lifecycle` channel |
| `doc/Skins.md`, `doc/AI_BUILD_NOTES.md` | Documents `exitApp()` and the platform behaviour |
| `test/unit/skin_feature/skin_exit_app_navigation_test.dart` | Tests the URL rules, missing, wrong and stale nonces, and subframe-started navigations |

### 2. Streamline.js: long-press Sleep

- **Tap Sleep:** unchanged; sleeps or wakes the machine.
- **Hold Sleep for 0.8 s:** sleeps the machine, then calls `exitApp()` once
  Decaid accepts the sleep command.
- **Reopen Decaid after a long-press exit:** goes straight to the Streamline
  interface and wakes the machine, as the de1app did, with no extra tap on the
  screensaver. A busy machine is never interrupted, and leaving the app any
  other way doesn't trigger a wake.
- **Hold "Awake":** only wakes the machine; it never exits.
- **On Decaid without `exitApp()`:** behaves like a normal sleep, so the skin
  change is safe to ship before or after the Decaid change.

It reuses Streamline's existing `setupPressAndHold` and keeps its rules for
when the skin may send sleep and wake commands. The decision logic is a small,
pure, unit-tested function in `screensaver-policy.js`.

**Patch:** [`patches/2-streamline-long-press-sleep.patch`](patches/2-streamline-long-press-sleep.patch)
against [decentespresso/streamline-js](https://github.com/decentespresso/streamline-js)
v0.2.24 (`10d2bc0`). Apply it with `git am`.

The same pattern also works for other skins. An Insight version was written and
tested along the way.

## Verification

- **Decaid:** 241 skin and web UI tests pass, including the new ones.
  `flutter analyze` and `dart format` are clean, on Flutter 3.44.2. The Kotlin
  change was not compiled here (no Android SDK), and the change has **not yet
  run on a tablet**.
- **Streamline.js:** 1315 of 1316 tests pass, including the new ones. The one
  failure, a chart test that needs the `echarts` dev dependency, also fails
  without this change. In a headless browser against a stubbed Decaid API:
  - A tap, or a hold under 0.8 s, sleeps without exiting.
  - A long press sleeps and then exits.
  - A rejected sleep command doesn't exit.
  - A host without `exitApp()` sleeps without errors.
- **Install:** Decaid's own GitHub-branch installer installs this repo as a
  separate skin.
- **On a real tablet with current Decaid:** the skin installs and a long press
  sleeps the machine and shows the screensaver, as expected without the Decaid
  change.

## Next steps

1. ~~Open an issue proposing `exitApp()`~~: accepted in decentespresso/decaid#1035.
2. ~~Decaid pull request~~: merged in decentespresso/decaid#1134, with the
   maintainer's requested nonce protection.
3. Streamline.js: the long-press change is proposed in an issue on
   [decentespresso/streamline-js](https://github.com/decentespresso/streamline-js/issues),
   following its issue-first process.

## Trying the skin (preview)

The repo root is an installable Streamline.js skin with the long-press change,
for anyone who wants to see it or test it against a Decaid build that includes
the patch. On current Decaid releases it behaves exactly like Streamline.js.

1. In Decaid, open **Launcher → Skins**, tap the **cogwheel**, then
   **Install skin → GitHub Branch**. Use the Skins page, not Plugins: installing
   it as a plugin fails with a "no plugin.js" error.
2. Enter `ReubenNinan/Decaid_Long_Press_Sleep_To_Exit_App-Streamline` and leave
   the branch as `main`.
3. Choose **Streamline.js (Long-Press Sleep Exit)**. It installs next to the
   official Streamline.js and doesn't replace it.

## Credits and license

Streamline.js is © Decent Espresso, licensed under GPL-3.0 (see `LICENSE.txt`
and the full license text in `LICENSE`). The skin here is based on
[decentespresso/streamline-js v0.2.24 (`10d2bc0`)](https://github.com/decentespresso/streamline-js/commit/10d2bc0).
Besides the long-press change, it uses its own skin ID (`streamline-sleep-exit`)
and version (`0.2.24-sleepexit.1`) in `skin-manifest.json` and `src/version.js`.
Its update check (`src/settings/settings.js`) points at this repo.
