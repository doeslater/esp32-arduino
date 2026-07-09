# esp32-arduino

A Claude Code skill for building, flashing, and debugging ESP32 `.ino` sketches with
[`arduino-cli`](https://arduino.github.io/arduino-cli/latest/) — with onboarding for
people new to ESP32 development, not just command execution.

Tested against a classic ESP32 DevKitC / WROOM-32 board. Other ESP32 variants
(S2/S3/C3) should mostly work but have different boot-strapping pins and peripheral
quirks — see [`reference.md`](reference.md) for what's confirmed vs. what differs.

## What it does

- Detects your board and serial port automatically via `arduino-cli board list`,
  falling back to asking only when detection is ambiguous.
- Checks for `arduino-cli` and the ESP32 core before doing anything — points you to
  install docs rather than guessing or silently installing things for you.
- Runs every `compile` / `upload` / `monitor` command from natural language ("flash
  this to my board") — no subcommand syntax to memorize — and always shows you the
  literal `arduino-cli` command and its full output, nothing summarized or hidden.
- Catches the two most common ESP32-specific compile failures — a missing library,
  and sketch-too-big/partition overflow — and explains the fix instead of leaving you
  with a raw compiler error.
- Offers contextual tips when it notices ESP32-specific gotchas (e.g. a blocking
  `delay()` call alongside WiFi/BLE) — not a general code review, and not on every
  successful build.
- Keeps a running, append-only `.esp32/NOTES.md` in your project so tips and lessons
  from past sessions are there for you to reread later.
- Offers to drop a standalone `run_<Sketch>.sh` script into your project, with
  `upload` / `monitor` / `check` subcommands (or a menu if run with no args), so you
  can flash and watch serial output without going through Claude each time.

## Install

Clone (or copy) this folder into your Claude Code skills directory:

```sh
# Personal — available in every project
git clone <this-repo-url> ~/.claude/skills/esp32-arduino

# Project-scoped — only active in one repo
git clone <this-repo-url> .claude/skills/esp32-arduino
```

The first time you ask it to build or flash something in a project, it walks you
through a short onboarding — board/port detection, code-comment style, tip
verbosity, and whether to keep a learning-notes file — and saves your answers to
`.esp32/config.json` in that project so you're not asked again.

## Updating

Skill files are read fresh from disk on every use, so a new version only reaches
you once the files at your install path actually change:

- **Cloned**: `git pull` inside `~/.claude/skills/esp32-arduino` (or the
  project-scoped path) to fetch the latest version.
- **Copied**: there's no update mechanism — re-copy the folder to pick up
  changes.

## Tested devices

- ESP32 DevKitC / WROOM-32, chip ESP32-D0WD-V3 (revision v3.1) — compile, upload, and
  serial capture confirmed working end-to-end against real hardware.

No other board has been tested. S2/S3/C3 and other ESP32 variants are not confirmed —
see [`reference.md`](reference.md) for what's known to differ. If you test this
skill against another board, opening an issue or PR with the result is welcome.

## Disclaimer

This skill flashes firmware to physical hardware based on its own judgment of your
intent and the state of your project. It is provided **as-is, with no warranty**
(see [LICENSE](LICENSE)) — use it at your own risk. Incorrect wiring, incorrect
voltage, or firmware that misuses a boot-strapping pin can damage a board or
anything connected to it; this skill reduces some common mistakes but cannot
guarantee your specific hardware setup is safe. You are responsible for reviewing
what it's about to do (which it always shows before doing it) and for your own
hardware. This project is not affiliated with or endorsed by Espressif Systems,
Arduino, or Anthropic.

## Topics

`esp32` `arduino` `arduino-cli` — the hardware/toolchain this skill targets.
`claude-code` `claude-code-skill` — it's a Claude Code skill, not a standalone tool.
`embedded` `iot` `microcontroller` — the broader category of work it's for.

## Version

0.2.1 — see [CHANGELOG.md](CHANGELOG.md).

## License

MIT — see [LICENSE](LICENSE).
