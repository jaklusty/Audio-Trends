# Soundbites desktop app: Phase 0 design note

Written 10 October 2026. This is the "read and plan" deliverable. Nothing is built yet.
It ends with the questions I need answered before Phase 1 starts.

## 1. The one thing I could not do

The brief says the existing system in `~/Movies/Soundbites/_library/` is the spec and must be
read first. That folder is on your Mac. This session runs in a cloud container that only holds
the Audio-Trends feed repository, so I have not been able to read `README.md`, `ingest.py`,
`libstore.py`, `hook.py`, `promote.py`, `index.html` or the state files.

That matters most for hard requirement 10. The TypeScript `slugify` must give byte-identical
ids to the Python one, and I can only prove that against the real function and the real
`library.json`. Everything in this note about the existing code is therefore inferred from the
brief and from the feed files, not from reading it.

**What I need from you:** a copy of the code and state, not the audio. Either zip these and
attach them, or push them to the new app repository (see section 7):

- `README.md`, `ingest.py`, `libstore.py`, `hook.py`, `promote.py`, `catchup.py`, `serve.py`, `index.html`
- `library.json`, `favourites.json`, `profile.json`, `promote_queue.json`, `health.json`, `last_run.json`
- a file listing of `clips/` and `full/` (for example the output of `ls -R clips full > listing.txt`)
- any run files in `pending/` and `pending/ingested/` if they exist

Total size should be under a few megabytes.

## 2. The three version-dependent facts, confirmed

### 2a. yt-dlp needs a JavaScript runtime for YouTube

Confirmed in the yt-dlp README and its EJS wiki page (read today).

- Since release 2025.11.12, full YouTube support needs an external JavaScript runtime. Deno is the
  default and the recommended one. Node, Bun and QuickJS also work but are opt-in.
- Minimum Deno version is 2.3.0. The current Deno release is v2.9.7.
- The option is `--js-runtimes deno:/path/to/deno`. The path may be the binary or its folder.
- Without a runtime, yt-dlp drops the `web` player client and falls back to the `visionos`
  client only. It still often works, but with fewer formats and it is officially deprecated.
  We will not rely on that.
- The standalone builds (`yt-dlp_macos`, `yt-dlp.exe`) bundle the challenge-solver scripts
  (`yt-dlp-ejs`). No "remote components" download is needed and we will leave that off.
- `yt-dlp_macos` is a single universal binary (Intel and Apple Silicon, macOS 10.15 or later).
  `yt-dlp.exe` is the Windows x64 build. Both are listed in `SHA2-256SUMS` on the same release.
- Deno ships as `deno-aarch64-apple-darwin.zip`, `deno-x86_64-apple-darwin.zip` and
  `deno-x86_64-pc-windows-msvc.zip`, each containing one file (`deno` or `deno.exe`). Each has a
  `.sha256sum` file next to it. The Mac one is a plain "hash  filename" line; the Windows one is
  PowerShell's `Algorithm / Hash / Path` layout with an upper-case hash. The verifier handles both.
- Deno runs the scripts with no file or network access, which is why yt-dlp recommends it over
  Node or Bun. Good for us: the feed is untrusted input.
- The current yt-dlp release is 2026.08.19. Latest-release URLs resolve fine.
- The `heatmap` field in yt-dlp's JSON output is confirmed: a list of
  `{start_time, end_time, value}` entries with `value` between 0 and 1. That is the
  "most replayed" source for hook detection.

### 2b. Ad-hoc signing in electron-builder

Confirmed against the published package `app-builder-lib@26.15.3`, which is what
`electron-builder@26.15.3` (current) installs. Its own option documentation says:

> Not set (default): electron-builder searches the keychain for a valid signing certificate. If
> none is found, signing is skipped for all architectures: there is no automatic ad-hoc fallback.
> `"-"`: opt in to ad-hoc signing explicitly. Note that `hardenedRuntime: true` (the default)
> combined with ad-hoc signing requires the `com.apple.security.cs.disable-library-validation`
> entitlement to prevent app launch failures; otherwise set `hardenedRuntime: false`.

So the setting is `mac.identity: "-"` with `mac.hardenedRuntime: false`. Two warnings from
the project's issue tracker that shaped this:

- Versions 26.0.13 to around 26.8 ad-hoc signed automatically on Apple Silicon. That was later
  removed, so a config that relies on the fallback silently ships an unsigned app on the next
  upgrade. We set `"-"` explicitly.
- Ad-hoc signing with hardened runtime left on makes the app fail at launch (library
  validation rejects the Electron framework) or lose microphone and camera access. We do not need
  hardened runtime until notarisation is turned on, so it stays off, and the signing switch in
  section 6 turns it on together with the Developer ID identity.
- The bundled ffmpeg sits in `extraResources` (outside the asar) and electron-builder signs
  everything in the bundle, so it gets the same ad-hoc signature. I will verify with
  `codesign --verify --deep --strict` in the build and fail the build if it does not pass.

### 2c. Opening an unsigned app on current macOS

Apple's Mac User Guide (the page for macOS Tahoe 26) gives these steps, and they are the only
supported route:

1. Double-click the app. A dialog says it cannot be verified. Click Done (not Move to Bin).
2. Open System Settings, click Privacy & Security, scroll to the Security section.
3. Next to the message about the app, click **Open Anyway**. The button is only there for
   about an hour after the attempt in step 1.
4. Enter the login password. The app is then remembered as an exception.

Control-click then Open stopped working in macOS 15 Sequoia and is still gone in Tahoe. The
"First time opening" guide will use the steps above with screenshots taken on your Mac in
Phase 4, because I cannot screenshot macOS from here.

One caution for Phase 4: the "damaged and can't be opened" message appears when the signature
does not match the bundle, not when it is merely untrusted. If a downloaded build says
"damaged", that is a build fault and we fix the build.

## 3. ffmpeg: a licence problem to decide before anything is bundled

I downloaded the Apple Silicon ffmpeg that the popular `ffmpeg-static` npm package ships and
read its build string. It was configured with `--enable-gpl --enable-nonfree`. "Nonfree" is the
ffmpeg project's term for a build that **may not be redistributed at all**, by anyone. So the
easy route (install `ffmpeg-static`, lipo its two Mac binaries together) is out. The same
package's Windows binary comes from gyan.dev and its Intel Mac binary from evermeet.cx; both
are GPL builds, which are redistributable if we meet the GPL's conditions.

What the options require:

| Build | Mac Apple Silicon | Mac Intel | Windows | Licence | What we must do |
|---|---|---|---|---|---|
| ffmpeg-static / osxexperts.net | yes | no | no | GPL + nonfree | Cannot ship. Ruled out. |
| evermeet.cx | no (Intel only; maintainer has said no ARM builds) | yes | no | GPL | Ship GPL text, link exact source |
| BtbN or yt-dlp FFmpeg-Builds | no | no | yes (`win64-gpl`, with checksums) | GPL | Same as above |
| ffmpeg.martin-riedl.de | yes | yes | no | not confirmed (site unreachable from here) | Check its build flags on your Mac first |
| Build our own in GitHub Actions | yes | yes | yes | LGPL (ffmpeg + LAME only) | Ship LGPL text and a link to the exact source and build script |

The app only needs ffmpeg to decode what yt-dlp downloads (Opus, AAC), cut and encode 30-second
MP3 clips (LAME, which is LGPL), write 16-bit WAV and measure loudness (built in). None of that
needs x264, x265 or any GPL-only part. A small LGPL build is enough, and we would control the
version and the licence completely. Building it is about 20 minutes of runner time per platform,
done once per ffmpeg version and stored as a release asset, so it does not slow the app build.

My recommendation is to build our own LGPL ffmpeg in GitHub Actions (one macOS job producing
arm64 and x86_64 and joining them with `lipo`, one Windows job). The alternative with the least
work is evermeet (Intel) + Martin Riedl (Apple Silicon) + BtbN (Windows), all GPL, provided
Riedl's build flags check out. Either way the About screen carries the ffmpeg licence text and
exact source links, the yt-dlp licence (Unlicense, public domain) and the Deno licence (MIT).

This is a decision for you because it sets what the About screen has to promise.
Section 7, question 3.

## 4. The design, in plain English

**One app, two halves.** The Electron main process is the engine. It owns the library folder,
runs yt-dlp and ffmpeg, keeps the state files safe and pushes changes to the window. The window
(renderer) is the old dashboard page, adapted, running with sandbox on, Node integration off and
context isolation on. It never loads anything from the internet and never navigates. All it can
do is call a short list of named operations through a preload bridge (open folder, star, play,
try again, wrong track, settings) and receive pushed updates.

**Engine modules**, each a direct port of a Python file with the same responsibilities:

- `store` (from `libstore.py`): atomic writes via write-to-temp-then-rename, a `.bak` of the last
  good copy, and a lock file so two operations never write at once. The lock is a directory-style
  lock with a stale timeout (the `proper-lockfile` approach), because Windows has no `flock`.
  Replacing a file on Windows retries for a few seconds before reporting, because antivirus holds
  files open. A corrupt file is copied to `<name>.corrupt-<timestamp>` first, never deleted, then
  restored from `.bak`, and Needs attention says so. With no usable backup, the engine stops and
  changes nothing.
- `ids` (from `slugify` in `ingest.py`): the id function, with Unicode handling matched to
  Python (`casefold`, NFKD for the readable part, NFKC for the hash, Unicode-aware `\w`). It gets
  its own test file of accented, CJK, Cyrillic and emoji-only titles, plus the whole of your
  `library.json`, before anything else is built on it.
- `ingest` (from `ingest.py`): candidate normalisation (`mood` string becomes a list, `suits`
  list becomes one string), new versus known, keep-runs, duplicate ids, filing rows, pruning once
  at the end of a run.
- `hook` (from `hook.py`): explicit `hook_start` first, then the YouTube heatmap, then the
  loudest 30-second window (recorded as method `energy`), then 0:20. The librosa chorus method is
  dropped as the brief says.
- `promote` (from `promote.py`): full-track download when a track is starred, in the format
  `profile.json` names, and "Free that space" reclaiming. Nothing deletes a full track unless asked.
- `feed` (new, replaces `catchup.py`): fetches `index.json` from the configured base (an https
  address or a local folder), sorts runs by date, keeps `feed_state.json` in the library folder,
  downloads audio only for the newest `keep_runs` runs, files older runs as metadata, works
  oldest first, and also picks up run files dropped into `pending/`. A `version` higher than 1
  stops ingest with "this feed needs a newer version of the app".
- `tools` (new): downloads and verifies yt-dlp and Deno into the app-data folder, checks weekly
  and after an extractor failure, uses versioned file names so a running `.exe` is never
  overwritten, re-downloads if a file goes missing, and always runs both with argument arrays and
  `--ffmpeg-location` pointing at the bundled ffmpeg.
- `queue` (new): one background download at a time by default (setting, up to 3), a 2-second
  pause between requests, visible progress, pause and cancel. The window stays usable during a
  one-hour run. Runs file their metadata rows immediately and fill in clips as they arrive, so
  quitting mid-run and reopening resumes without duplicates.

**Audio to the window.** A sandboxed page cannot read files, so the main process registers a
custom scheme (something like `soundbites://clip/<id>` and `soundbites://full/<id>/<file>`). The
handler maps only to files inside the chosen library's `clips/` and `full/` folders, refuses
anything else, and answers byte-range requests itself (206 Partial Content) so scrubbing works.
I am not relying on Electron's file streaming to handle ranges; the handler does it explicitly.

**Settings in two places, on purpose.** Library path, feed address, download concurrency and
window position live in the OS app-data folder. Everything about the library itself
(`library.json`, `favourites.json`, `profile.json`, `feed_state.json`, locks) stays inside the
library folder, so the folder stays portable and the Python files stay readable.

**Locks.** `app.requestSingleInstanceLock()` stops a second copy of the app on the same
machine. A lock file in the library folder stops a second machine or account. A lock older than
its heartbeat is reported as stale and offered for take-over, never silently blocking.

**Saved views.** The old views filter on a `profiles` tag nothing sets, so they never worked. My
proposed display rules, based only on fields the feed actually fills:

| View | Proposed rule |
|---|---|
| High energy | `energy` is "high" |
| Vocal-led | any `mood` tag contains "vocal", or `suits` mentions vocal |
| Instrumental | any `mood` tag contains "instrumental" or "no vocals" |

I need your `profile.json` to see what the three views are actually called before I finalise
this. You can adjust the rules in Settings and the chosen view is remembered in
`profile.json` under `views.active`. Views only change what is shown.

**New in v1** as the brief lists: match transparency fields on each track (upload title,
channel, length), "no confident match" when neither the candidate title nor artist appears in
the upload's title or channel, "Wrong track" with rejected video ids recorded per track, a
user-pasted YouTube link that a later feed run never overwrites, drag to DaVinci Resolve via
`startDrag` from the main process (macOS requires a non-empty drag icon, so one is bundled),
"Show in Finder / Explorer", a feed status line, and a GitHub Releases check for a newer version.

**Tooling.** Electron 44, electron-builder 26.15, electron-vite 5, Vitest 5, TypeScript. The
Mac `.dmg` is a universal build (`arch: universal`). The Windows NSIS installer is built on a
Windows runner in GitHub Actions. ffmpeg is in `extraResources`, outside the asar.

## 5. Where I would differ from the brief, and why

1. **ffmpeg source.** The brief implies a download-and-bundle package. The obvious one is
   unshippable (section 3). I propose building a small LGPL ffmpeg ourselves.
2. **yt-dlp self-update.** The brief's approach (download a new versioned file, verify, switch)
   is right. I would additionally never call yt-dlp's own `--update`, because it replaces the file
   in place, which is exactly the Windows problem the brief describes.
3. **Remote components for yt-dlp.** Left off entirely (`--no-remote-components`). The
   standalone binaries carry what they need, and a feed-driven app should not fetch extra scripts
   from npm or GitHub at run time.
4. **The GitHub Releases version check** needs the app's own repository to exist first, and the
   unauthenticated API allows 60 requests an hour. Once per launch and once a day is plenty.
5. **Testing where I am.** I can run the engine's unit tests, the Phase 1 comparison runs
   (Python versus TypeScript on copies of the library) and headless Electron tests here. I cannot
   play audio, screenshot macOS, or test Windows. Phase 2 playback checks, Phase 4 install tests
   and Phase 5 happen on your machines with me guiding. The brief's rule stands: nothing counts as
   working on Windows until it has run on a real Windows machine or VM.
6. **Where this note lives.** The brief says the app must not live in Audio-Trends. The note is
   committed here only because this session's working branch is in Audio-Trends. The app code
   starts in its own repository as soon as you name it.

Everything else in the brief I plan to follow as written.

## 6. The signing switch, documented now so it is not forgotten

In `electron-builder.yml`, the Mac section will read:

```yaml
mac:
  identity: "-"          # ad-hoc. Replace with Developer ID name when a certificate exists.
  hardenedRuntime: false # must become true for notarisation
  notarize: false        # true + APPLE_API_KEY, APPLE_API_KEY_ID, APPLE_API_ISSUER in CI secrets
```

Turning on real signing later means: set `identity` to the Developer ID name (or leave it unset
and provide `CSC_LINK` / `CSC_KEY_PASSWORD` in CI), set `hardenedRuntime: true`, set
`notarize: true` with the three Apple API secrets, add `electron-updater` with `publish:
github`, and switch the in-app "newer version available" link to a real auto-update prompt. No
code structure changes are needed for any of that; it is configuration plus one module.

## 7. Decisions I need from you

1. **Name.** Working name is Soundbites. Some options, in no order and with no favourite:
   Soundbites, Hooks, Clipbin, Earworm, Trendbin, Sound Shelf, Thirty (as in thirty seconds),
   Audition. Any of these, or your own.
2. **Icon and look.** Your call. I can mock up two or three directions once the name is set, or
   you can send a sketch or a reference.
3. **ffmpeg route** (section 3): build our own LGPL ffmpeg, or use the three third-party GPL
   builds after checking Martin Riedl's flags on your Mac.
4. **App repository.** The app needs its own GitHub repository (the Windows build runs there).
   A name, and whether it is public or private. Public makes the GPL/LGPL source-offer trivial
   and lets the "newer version available" check work without a token.
5. **A Windows 10 or 11 machine or VM** you can reach for Phase 4. Also tell me whether it is a
   Home edition with Smart App Control on, since that blocks unsigned installers with no
   "Run anyway" button.
6. **The library code and state** listed in section 1.

Once I have items 4 and 6, Phase 1 can start: the id function and its tests first, then store,
ingest, hook and promote, each checked against the Python on a copy of the library.

## Sources read today

- yt-dlp README and EJS wiki: https://github.com/yt-dlp/yt-dlp and https://github.com/yt-dlp/yt-dlp/wiki/EJS
- yt-dlp announcement, external JavaScript runtime required: https://github.com/yt-dlp/yt-dlp/issues/15012
- electron-builder: `app-builder-lib@26.15.3` option docs (from the npm package) and current source in https://github.com/electron-userland/electron-builder; issues 9529 and pull request 9007 on ad-hoc fallback
- Apple, Open a Mac app from an unknown developer: https://support.apple.com/guide/mac-help/mh40616/mac
- Electron API docs for `protocol.handle`, `startDrag`, `will-navigate`, `setWindowOpenHandler`: https://github.com/electron/electron/tree/main/docs/api
- ffmpeg-static download script and binaries: https://github.com/eugeneware/ffmpeg-static ; BtbN builds: https://github.com/BtbN/FFmpeg-Builds ; yt-dlp's own builds: https://github.com/yt-dlp/FFmpeg-Builds
- Deno releases: https://github.com/denoland/deno/releases

## Update, 10 October: decisions taken

- ffmpeg: build our own LGPL ffmpeg in GitHub Actions. Done as `.github/workflows/ffmpeg.yml`
  in the app repository.
- Name: Soundbites.
- Logo: a music symbol with a bite taken out of it. Four variants are in
  `design/logo/` in the app repository for Jak to pick from.
- App repository: `jaklusty/soundbites`, private.
- Still open: a Windows machine or VM for Phase 4, and the library code and state files.
