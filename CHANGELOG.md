# Changelog

## 0.2.0 — 2026-07-09

- Onboarding step 7: offer (never auto-create) to generate `run_<Sketch>.sh`, a
  standalone helper script committed to the project root, named after the active
  sketch, one per sketch.
- The generated script resolves paths relative to its own location and re-reads
  `fqbn`/`port`/`monitorBaud` from `.esp32/config.json` on every run via
  `grep`+`sed` (no `jq`/`python3` dependency), so it never goes stale when the
  port changes.
- Three subcommands — `upload` (compile then upload), `monitor` (live serial,
  since the script runs in a real TTY unlike Claude's shell), `check` (bounded
  10s capture, same technique as the skill's own internal sanity check). Bare
  invocation shows an explained menu; invalid args or `-h`/`--help` show the same
  as usage text.

## 0.1.0 — 2026-07-09

Initial release.

- Onboarding: prerequisite check (arduino-cli + esp32 core), board/port detection via
  `arduino-cli board list`, preferences for code comments / tip verbosity / learning
  notes, optional starter-sketch scaffold, optional `.gitignore` entry for `.esp32/`.
- Natural-language compile/upload/monitor, always showing the literal `arduino-cli`
  command and its full output.
- Sketch/folder name mismatch check before compiling.
- Recognizes missing-library and partition-overflow compile failures and explains
  the fix.
- Bounded (10s) serial capture after upload via raw `stty`+`cat`, not
  `arduino-cli monitor` — confirmed by testing that `monitor` exits immediately with
  no output when stdin isn't an interactive TTY.
- Contextual ESP32-specific tips (not a general linter), gated by tip verbosity,
  logged to an append-only `.esp32/NOTES.md`.
- Lazy port re-validation (only re-resolves on an actual upload/monitor failure).
- Multi-sketch-per-project resolution, persisted once resolved.

Tested end-to-end (compile, upload, serial capture, missing-library error text)
against a real classic ESP32 DevKitC / WROOM-32 board (ESP32-D0WD-V3). Other ESP32
variants (S2/S3/C3) are not yet verified — see `reference.md`.
