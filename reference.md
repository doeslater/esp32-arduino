# Reference: ESP32 caveats and file layout

Confirmed against a classic ESP32 DevKitC / WROOM-32 board (FQBN `esp32:esp32:esp32`).
S2/S3/C3 have different strapping pins and peripheral maps — don't assume anything
below carries over to those variants; check the specific board's datasheet.

## Boot-strapping pins (classic ESP32)

These pins have a role during boot, before your sketch runs. Driving them the wrong
way externally (a pull-up/pull-down resistor, a connected sensor, etc.) can prevent
the board from booting at all — one of the most common "why won't my board start"
issues for first-time ESP32 users.

| Pin | Role at boot | Notes |
|---|---|---|
| GPIO0 | Boot mode select | HIGH (internal pull-up) = normal boot from flash. LOW = UART download mode. Most dev boards wire this to a BOOT button for exactly this reason. |
| GPIO2 | Must be LOW or floating | Involved in boot-mode selection alongside GPIO0 on some revisions. Often tied to the onboard LED, which is usually fine, but avoid also pulling it HIGH externally at boot. |
| GPIO12 (MTDI) | Flash voltage select | Internal pull-down. If pulled HIGH at boot, the chip selects 1.8V flash timing instead of 3.3V — on a 3.3V flash board (most dev boards), this can prevent boot entirely. Leave floating or LOW unless you specifically know your flash needs 1.8V. |
| GPIO15 (MTDO) | Boot log / boot-mode | Internal pull-up. Pulling it HIGH at boot can suppress boot log output. |

**GPIO6–GPIO11** are wired directly to the integrated SPI flash on WROOM-32 modules.
Never use these as general-purpose GPIO — doing so will make the board unable to
read its own flash.

## ADC2 vs. WiFi

The ESP32's WiFi driver uses ADC2 internally. Any pin on ADC2
(GPIO0, 2, 4, 12, 13, 14, 15, 25, 26, 27) becomes unreliable — often returning noise
or failing outright — the moment WiFi is active (even just connected, not
transmitting). If a sketch needs both WiFi and analog reads, use an ADC1 pin instead
(GPIO32–GPIO39), which is unaffected.

## Deep-sleep wake pins

Only RTC-capable GPIOs can wake the chip from deep sleep as an external wake source:
GPIO0, 2, 4, 12, 13, 14, 15, 25, 26, 27, 32–39. `ext0` wake uses exactly one of these;
`ext1` wake can use several at once as a bitmask. Any other pin simply can't trigger
a wake-up — this is a common surprise when porting a sketch from a non-RTC-capable
pin.

## FQBN reference

- Classic ESP32 DevKitC / WROOM-32: `esp32:esp32:esp32`
- Run `arduino-cli board listall esp32` to see every board definition the installed
  `esp32:esp32` core provides, if targeting a different variant.

## File layout

```
your-project/
├── .esp32/
│   ├── config.json   # board/port/preferences — see SKILL.md for schema
│   └── NOTES.md       # append-only log of tips given across sessions
└── Blink/
    └── Blink.ino
```

`.esp32/config.json` contains a machine-specific serial port path — the skill offers
to add `.esp32/` to your project's `.gitignore` during onboarding for this reason.
