# android-emulator-demo-video

An [opencode](https://opencode.ai) skill for recording demo videos of Android apps running on an emulator, driven entirely by `adb` — no manual interaction needed.

## What it does

- Records a screen video of an app/feature/flow on a running Android emulator (`adb screenrecord`)
- Drives navigation via `adb` (`input tap`, `input swipe`, `keyevent`)
- Captures a numbered screenshot for each step of the flow
- Finalizes and pulls the output files to your machine

## Install

Copy or clone this skill into your skills directory:

```sh
git clone https://github.com/Quinus/android-emulator-demo-video ~/.config/opencode/skills/android-emulator-demo-video
```

## Requirements

- A running Android emulator (phone, tablet, or Android TV)
- `adb` on your `PATH`

## Usage

Ask opencode to demo, record, or capture a video of a feature, flow, ticket, or bug on a running Android emulator. The skill handles device selection, scouting the navigation path before recording, capturing video plus per-step screenshots together, and finalizing the output files.
