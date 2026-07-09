---
name: esp32-arduino
version: 0.2.0
description: >
  Build, flash, and debug ESP32 .ino sketches with arduino-cli, including first-time
  environment setup. Trigger on build/upload/setup intent for an ESP32 project —
  "compile this", "flash it to my ESP32", "upload my sketch", "set up my ESP32
  project", "open the serial monitor" — not on casual mentions of .ino files or ESP32
  hardware with no build/upload/setup intent.
allowed-tools:
  - Bash(arduino-cli:*)
  - Read
  - Write
  - Edit
  - Glob
---

# esp32-arduino

Helps someone build, flash, and debug ESP32 sketches via `arduino-cli`, and teaches
ESP32-specific gotchas along the way. Tested against a classic ESP32 DevKitC /
WROOM-32 board (FQBN `esp32:esp32:esp32`). Other variants (S2/S3/C3) differ in
boot-strapping pins and peripherals — see [reference.md](reference.md).

Never run OS-specific shell commands (no `/dev/tty*` globbing, no `open`/`xdg-open`/
`start`). Everything needed — board detection, port detection, core/library
management — goes through `arduino-cli` itself, which is cross-platform.

**Show the literal `arduino-cli` command you're about to run before running it, and
show its full output. Never summarize, truncate, or silently swallow output —
this skill's whole value is transparency over convenience.**

## First step, every time: find or create `.esp32/config.json`

Look for `.esp32/config.json` in the current project. If it exists, read it and skip
onboarding — go straight to "Doing the work" below. If it doesn't exist, run
onboarding.

## Onboarding (first run in a project)

Ask nothing you can detect. Ask exactly what you can't.

1. **Check prerequisites.** Run `arduino-cli version` and `arduino-cli core list`.
   - If `arduino-cli` itself isn't found: tell the user to install it from
     https://arduino.github.io/arduino-cli/latest/installation/ and stop. Do not
     attempt an OS-specific install command yourself.
   - If `esp32:esp32` isn't in the core list: tell them to run
     `arduino-cli core install esp32:esp32` themselves (show the command), and stop.
     Do not run installs on their behalf without asking first.

2. **Detect board and port.** Run `arduino-cli board list`.
   - Exactly one ESP32-like board detected → use its port automatically, no question
     asked.
   - Zero or multiple candidates → show what was found and ask which port/board to
     use.
   - Default FQBN is `esp32:esp32:esp32` (classic DevKitC/WROOM-32) unless the user
     says otherwise or `board list` clearly identifies a different variant.

3. **Ask preferences** (not detectable, always ask):
   - Code comments in generated `.ino` sketches: educational (explains *why*,
     ESP32-specific reasoning) or off. Default recommendation: educational.
   - Tip verbosity: minimal / balanced / tutor. Default recommendation: balanced.
   - Keep a learning-notes log (`.esp32/NOTES.md`)? Default recommendation: yes.

4. **Offer to scaffold a starter sketch** if no `.ino` file exists anywhere under the
   project directory. Ask first — e.g. "want me to create a Blink sketch so we can
   test the full compile → upload → monitor loop?" Never create it unasked.

5. **Offer to gitignore `.esp32/`.** If the project has a `.gitignore` and it doesn't
   already cover `.esp32/`, ask before appending it — the config file contains a
   machine-specific serial port path that shouldn't be committed.

6. **Write `.esp32/config.json`** with the resolved values (schema below).

7. **Offer to generate a run script.** Ask — never auto-create — e.g. "want me to
   drop a script here so you can upload/monitor without going through me each
   time?" Default recommendation: yes. If accepted, write `run_<Sketch>.sh` to the
   project root (see below) and `chmod +x` it. If declined, skip; this can be
   revisited later on explicit request, same as any other reconfiguration.

### Config schema (`.esp32/config.json`)

```json
{
  "fqbn": "esp32:esp32:esp32",
  "port": "/dev/ttyUSB0",
  "monitorBaud": 115200,
  "activeSketch": "Blink/Blink.ino",
  "comments": true,
  "tipVerbosity": "balanced",
  "learningNotes": true
}
```

### Generated script (`run_<Sketch>.sh`)

If step 7 is accepted, generate `run_<SketchBaseName>.sh` in the project root,
named after the currently active sketch (e.g. `run_Blink.sh` for
`Blink/Blink.ino`). **One script per sketch** — if the active sketch later
changes and the user asks for a script again, generate a new
`run_<NewSketch>.sh` alongside it; never rename or delete a script generated for
another sketch. Commit it (don't gitignore it) — unlike `.esp32/config.json`, it
holds no machine-specific values.

The script:

- Resolves its own directory (`dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" &&
  pwd)"`) and reads `.esp32/config.json` relative to that directory, not the
  caller's cwd, so it works no matter where it's invoked from.
- Re-reads `fqbn`, `port`, and `monitorBaud` from `.esp32/config.json` on every
  run via `grep`+`sed` — never bakes those values in at generation time. No
  `jq`/`python3` dependency: the config format is fully controlled by this
  skill, so simple line-based extraction is safe. This also means a port Claude
  re-resolves later is picked up automatically, with no need to regenerate the
  script.
- Exposes three subcommands:
  - `upload` — compile, then only on success, upload:
    `arduino-cli compile --fqbn <fqbn> <sketch-dir>` then
    `arduino-cli upload -p <port> --fqbn <fqbn> <sketch-dir>`. A thin wrapper —
    it shows the commands and lets arduino-cli's own exit code/output speak; it
    does not attempt the smart error diagnosis (missing-library,
    partition-overflow) that the Claude-driven flow does.
  - `monitor` — live `arduino-cli monitor -p <port> -c baudrate=<monitorBaud>`,
    blocking until the user exits. Unlike `check` below, this can run live
    because the script executes in the user's own terminal with a real TTY — the
    constraint that forces the bounded workaround (see "Monitor" below) only
    applies to Claude's own non-interactive shell.
  - `check` — bounded 10s capture, the same technique used internally: `stty -F
    <port> <monitorBaud> raw -echo -hupcl; timeout 10 cat <port>`. A quick "did
    it boot" sanity check.
- Bare invocation (no args) shows a numbered menu with one-line explanations;
  invalid args or `-h`/`--help` print the same lines as usage text:
  ```
  1) Upload   - compile the sketch and flash it to the device
  2) Monitor  - open a live serial connection (Ctrl+C to exit)
  3) Check    - read the serial port for 10s, then stop (quick sanity check)
  ```

## Doing the work

### Resolving the active sketch

`arduino-cli compile` operates on one sketch directory at a time.

- If `config.json` has `activeSketch` and it still exists, use it without asking.
- If there's exactly one `.ino` sketch in the project, use it and save it as
  `activeSketch`.
- If there's more than one and none is set as active (or the user asks for a
  different one by name), ask which one, then persist the answer to `activeSketch`.

### Sketch/folder name check (before compiling)

`arduino-cli` requires the sketch's containing folder to have the exact same name as
the `.ino` file (e.g. `Blink/Blink.ino`, not `Blink/main.ino`). Check this before
compiling. If mismatched, explain the rule and ask before renaming anything — never
rename silently.

### Compiling

Run `arduino-cli compile --fqbn <fqbn> <sketch-dir>`, shown before it runs.

Recognize two ESP32-specific failure patterns in the output and explain the fix
instead of leaving the raw error to speak for itself:

- **Missing library** (`fatal error: X.h: No such file or directory`) — offer
  `arduino-cli lib install <library-name>`, show the exact command, run it only after
  confirming.
- **Partition overflow** (`sketch too big` / `text section exceeds available space`)
  — explain that the default partition scheme is too small for this sketch and
  suggest switching partition scheme (e.g. via `--build-property` or a larger-app
  partition table), pointing at the specific fix rather than just the symptom.

Any other compile error: show arduino-cli's raw message. Don't try to interpret
general C++ errors — that's out of scope.

### Uploading

Before uploading, re-check the configured port only if the *previous* upload/monitor
attempt failed with a port error — don't re-run `board list` before every upload
just to confirm the port is still valid. On a clean run, trust the saved port.

Run `arduino-cli upload -p <port> --fqbn <fqbn> <sketch-dir>`, shown before it runs.

- **On failure:** if the port is no longer in `arduino-cli board list`, re-resolve it
  (ask if ambiguous) and retry once. If the failure looks like it needs manual BOOT-
  button intervention, tell the user, and send a notification (see below) since this
  is a moment the user needs to act before you can continue.
- **On success:** immediately run a short monitor capture (see below) to confirm the
  sketch is actually running.

### Monitor

`arduino-cli monitor` is built as an interactive tool: it reads stdin to detect
exit keystrokes, and **exits immediately with no output the moment stdin isn't a
real TTY** — which is exactly the situation when a command is run from an agent's
shell tool. Confirmed by testing: it returns in well under a second with zero bytes
captured, even while the board is actively printing. Do not use it for the bounded
capture below; only hand it to the user as a literal command to run in their own
terminal, where stdin genuinely is a TTY and it works normally.

For the bounded capture, read the port directly instead:

```sh
stty -F <port> <monitorBaud> raw -echo -hupcl
timeout 10 cat <port>
```

- `-hupcl` matters: without it, closing the port toggles DTR and resets the board
  (common ESP32 auto-reset wiring), which can make the *next* read catch the board
  mid-reboot instead of in steady state. Set it once per port before reading.
- Default behavior: capture 10 seconds this way after an upload, then show exactly
  what was captured. This is for "did my sketch boot correctly," not live debugging.
- For actual live/ongoing debugging, tell the user the literal
  `arduino-cli monitor -p <port> -c baudrate=<monitorBaud>` command and let them run
  it themselves in their own terminal — don't try to keep a long-running stream open
  inside a single turn.

### Notifications

Send a push notification only when an upload fails in a way that needs the user to
physically do something before you can continue (most commonly: hold BOOT and
retry). Don't notify on routine compile/upload/monitor success or failure — that's
already visible on screen and doesn't need to pull the user's attention away.

### Tips and learning

Tips are contextual, not automatic tutorials. Only offer one when, while already
compiling or reading a sketch, you notice a specific ESP32-relevant gotcha:

- Blocking `delay()` calls combined with WiFi/BLE usage (stalls the radio stack).
- Deprecated Arduino-core APIs for this board.
- A software-driven pattern (e.g. manual bit-banging) where a hardware peripheral
  (PWM/RTC/timer) would be simpler or more efficient.

Scope this strictly to ESP32-specific knowledge — this is not a general code
reviewer or linter, and don't recommend "better libraries" speculatively; only
mention an alternative when the current one has a known, concrete ESP32-specific
problem.

Respect `tipVerbosity`:
- `minimal` — only mention something on actual failure.
- `balanced` — failure, plus opportunistic gotchas noticed in passing.
- `tutor` — same as balanced, but lean toward mentioning more of what you notice.

If you reference an external tutorial or doc page, print it as a plain URL — don't
attempt to launch a browser (no cross-platform way to do that without OS-specific
commands).

If `learningNotes` is `true`, append (never overwrite) a short, timestamped entry to
`.esp32/NOTES.md` for each tip/lesson given, e.g.:

```markdown
## 2026-07-09 — blocking delay() with WiFi active
Explanation of what was noticed and why it matters.
```

Do not read `NOTES.md` back before giving a tip — it's a write-only log for the
user's own later reference, not a dedup mechanism; checking it every time would mean
reading a growing file on every invocation for little benefit.

## Reconfiguring later

- **Stale port** (unplugged/replugged, different USB port): detect and re-resolve
  automatically only when an upload/monitor call actually fails — never proactively
  before every command.
- **Anything else** (different board variant, changed preferences): only on explicit
  request (e.g. "reconfigure", "use a different board") — don't guess intent and
  don't silently rewrite settings the user didn't ask to change.
