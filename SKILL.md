---
name: android-emulator-demo-video
description: >
  Record a demo video of an app running on an Android emulator (phone, tablet,
  or Android TV) using adb screenrecord, driven by adb navigation (input tap /
  swipe / keyevent), plus a numbered screenshot for each step of the flow.
  Use when asked to demo, record, or capture a video of a feature/flow/
  ticket/bug on a running Android emulator. Covers picking the right device,
  scouting navigation before recording, capturing the video and per-step
  screenshots together, and finalizing the output files.
---

# Android emulator demo video

Record a running Android emulator to produce a demo of a flow/feature, via
`adb`. No extra tooling required beyond `adb` (on PATH from Android SDK
platform-tools) and `ffprobe`/`ffmpeg` (optional, for verification/post-
processing — install with `brew install ffmpeg` if missing).

**Output for every demo is a pair of deliverables in one folder:** one video
of the full flow, plus one numbered screenshot per step of that same flow,
named in order.

## 1. Identify the target device

```bash
adb devices -l
```

If more than one device/emulator is attached, **always pass `-s <serial>`**
on every subsequent `adb` call — don't rely on the default device. Tell
device types apart from the `model`/`product` field: TV emulators say
`sdk_google_atv*` / `model:...atv...`; phone/tablet emulators say
`sdk_gphone*`. Ask the user which device if it's ambiguous and they didn't
say.

## 2. Plan the steps and output folder

Before touching screenrecord, decide the ordered list of steps the demo
should show (e.g. "01 home", "02 open detail page", "03 start playback",
"04 ad playing", "05 stream playing"). This step list drives both the video
pacing and the screenshot filenames later.

Create the output folder:

```bash
mkdir -p ~/Movies/demo/<ticket-or-feature>_<device>/
```

Default parent directory is `~/Movies/demo/` unless the user names another
location. Use a descriptive folder name (ticket id or feature name + device
type).

## 3. Scout the navigation path

Never improvise taps blind against a scaled-down screenshot — the `Read`
tool downsamples images, so eyeballed coordinates from the displayed image
are wrong. Instead:

- Take a scratch screenshot to see current state: `adb -s <serial> exec-out screencap -p > /tmp/state.png`, then `Read` it.
- Get **exact** tap coordinates with a UI dump instead of guessing from the screenshot:
  ```bash
  adb -s <serial> shell uiautomator dump /sdcard/ui.xml
  adb -s <serial> pull /sdcard/ui.xml /tmp/ui.xml
  grep -o 'text="TV"[^/]*bounds="[^"]*"' /tmp/ui.xml   # find the element, read its bounds
  ```
  `bounds="[x1,y1][x2,y2]"` — tap the center. Cross-check with `adb shell wm size` if unsure the screenshot matches the real resolution.
- Walk the whole planned flow step by step with scratch screenshots between each action, confirming each screen looks right and matches the step list from step 2, **before** touching screenrecord. Getting the sequence right first means one clean take instead of re-recording.
- Reset to a known starting state before the real take: `adb -s <serial> shell am force-stop <package>` then relaunch (`adb shell monkey -p <package> -c android.intent.category.LAUNCHER 1`), or navigate back to the home/start screen.

## 4. Navigation input reference

- **Touch (phone/tablet):** `adb shell input tap <x> <y>`, `adb shell input swipe <x1> <y1> <x2> <y2> <duration_ms>`.
- **D-pad (Android TV):** `adb shell input keyevent KEYCODE_DPAD_UP|DOWN|LEFT|RIGHT|CENTER`, `KEYCODE_BACK`. TV UIs are focus-driven — screenshot after each keyevent to confirm which element gained focus before pressing center/enter.
- Package name (for force-stop/relaunch): read it off a `uiautomator dump` (`package="..."` attribute) or `adb shell dumpsys window | grep mCurrentFocus`.

## 5. Record the video and the per-step screenshots together

Do this in a single take: start screenrecord in the background, then drive
the UI through the planned steps, capturing a numbered screenshot right
after each step lands (screenshots write straight to the final output
folder — no separate pull needed for those).

```bash
adb -s <serial> shell screenrecord --bit-rate 8000000 /sdcard/demo_raw.mp4   # run_in_background: true

# for each planned step, in order:
#   perform the tap/swipe/keyevent for that step
#   sleep briefly for pacing / for the UI to settle
adb -s <serial> exec-out screencap -p > ~/Movies/demo/<name>_<device>/01_<step-description>.png
#   ... repeat for 02_, 03_, ... continuing through the whole flow ...

adb -s <serial> shell pkill -INT screenrecord   # SIGINT, not kill -9 — finalizes the mp4 container cleanly
```

Notes:
- Zero-pad the numeric prefix (`01_`, `02_`, ... `10_`) so filenames sort in order; give each one a short kebab-case description of that step.
- `--bit-rate 8000000` keeps quality decent without huge files; add `--size WIDTHxHEIGHT` if you need to force a resolution (e.g. matching TV output).
- **Always stop with `pkill -INT`** (or let it hit its own time limit). Killing the process harder, or pulling the file while it's still recording, produces a corrupt/unplayable mp4 (missing moov atom).
- Default max duration per invocation is ~180s. For a longer demo, either keep takes tight and focused, or record multiple clips and concatenate with `ffmpeg -f concat`.
- Don't simulate destructive or hard-to-reverse conditions (e.g. tampering with the network stack, force-triggering error states) without checking with the user first — offer it as an option rather than doing it silently.

## 6. Pull and finalize the video

```bash
adb -s <serial> pull /sdcard/demo_raw.mp4 ~/Movies/demo/<name>_<device>/demo_raw.mp4
adb -s <serial> shell rm /sdcard/demo_raw.mp4          # clean up the device
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1 ~/Movies/demo/<name>_<device>/demo_raw.mp4   # sanity-check it's playable and check length
mv ~/Movies/demo/<name>_<device>/demo_raw.mp4 ~/Movies/demo/<name>_<device>/<name>_<device>.mp4   # descriptive final name, not demo_raw.mp4
```

End state is one folder containing the full-flow video plus every
`NN_step-description.png` in order:

```
~/Movies/demo/<name>_<device>/
  <name>_<device>.mp4
  01_home.png
  02_open-detail-page.png
  03_start-playback.png
  04_ad-playing.png
  05_stream-playing.png
```

## 7. Report back

List the output folder contents (video + numbered screenshots). State
plainly what the recording actually shows (the real navigation path taken)
and what it does **not** cover — e.g. if you only captured normal playback
and did not simulate an error/edge case because that needs extra setup
(proxy, network tampering, debug menu) the user hasn't approved yet.
