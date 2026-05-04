# OSD Button Reference

Settings that are **not** exposed via the web UI (LCD brightness, the three alarms, alarm chime, time format, manual date/time) are reachable only via the physical buttons on the rear panel.

This document supersedes the earlier two-button (`SET` / `UP`) draft, which was based on the closest matching vendor manual ([DIY ESP8266 Networking Clock Kit, manuals.plus](https://manuals.plus/ae/1005006339767938)) but did not reflect the actual hardware. Bench observation across three MAS-8710 units confirms **four** buttons.

## Button layout

Four physical buttons on the rear panel, top-to-bottom on the right edge. See [`images/rear-panel.png`](images/rear-panel.png).

| Label (CN) | Label (EN) | Function |
|---|---|---|
| 设置 | **SET** | Manual date/time edit |
| 增加 | **+** | Cycle display modes / increment values |
| 减少 | **−** | Parameter-config menu / decrement values |
| 闹钟 | **alarm** | Alarm-time setup (3 slots) |

> When the unit is online and NTP-synced, `SET` is rarely useful — NTP authoritatively writes year/month/day/hour/minute/second and overwrites any manual entry. `SET` is mainly relevant for offline use.

## SET — manual date & time

In normal display, press `SET` to enter the edit cascade. Each `SET` press advances to the next field; `+` and `−` adjust the flashing value:

```
year → month → day → hour → minute → second → save & exit
```

For **seconds**, pressing `+` resets the value to `00` (a "snap-to-zero" trick for sub-second alignment against a reference). Final `SET` press saves and returns to the time display.

## `+` — display mode

In normal display, `+` cycles the display mode. The clock briefly flashes a `dp:N` prompt to confirm:

| Code | Display behaviour |
|---|---|
| `dp:0` | Time only (continuous) |
| `dp:1` | Date only (continuous) |
| `dp:2` | Time and date alternating |

## `−` — parameter menu

In normal display, `−` enters the parameter-config menu. Each `−` press advances to the next parameter; `+` changes the value of the currently shown parameter. Final `−` press exits and saves.

| Code | Meaning | Values |
|---|---|---|
| `LP` | Manual brightness | `1` (darkest) – `8` (brightest). **Only effective when `AL:0`.** |
| `AL` | Auto-brightness | `0` = off (use `LP`), `1` = auto (LDR-driven on units with the sensor populated) |
| `BP` | Alarm sound | `0` = silent, `1` = ring |
| `FF` | Time format | `12` = 12-hour, `24` = 24-hour |

When `AL:1`, the unit dims itself based on ambient light. The 19:00–07:00 schedule that some vendor manuals describe is a fallback heuristic; actual behaviour on units with the LDR populated is light-driven, not time-driven.

## `alarm` — three alarms

In normal display, press `alarm` to enter alarm-edit mode. The unit shows `-1-`, `-2-`, `-3-` to indicate which of the three independent alarm slots is being edited. For each slot:

- Hour, then minute (use `+` / `−` to adjust, `alarm` or `SET` to advance — depends on firmware revision).
- Each slot has its own enable flag.

Master sound on/off is the `BP` parameter in the `−` menu, **not** per-alarm.

**Behaviour at fire time:** alarm rings for 30 seconds; any key press silences it.

**Vendor tip:** to ring only once per day, set all three slots to the same time — the firmware doesn't deduplicate, but a single 30-second ring covers all three identical triggers.

## SET — long-press

Behaviour varies by firmware revision. The two known assignments:

- **Factory reset** (clears WiFi credentials, NTP server, DST/timezone, and re-enables the `Config-XXXXXX` captive-portal AP). Long-press duration typically ≥ 5 seconds.
- **Manual time adjustment mode** on offline units (no NTP). Not relevant when the clock is online.

If a long-press unexpectedly enters manual-time mode rather than factory reset, exit by pressing `SET` repeatedly to walk through year/month/day/hour/minute/second without changing them, or wait for the timeout.

## What the OSD does *not* expose

- WiFi credentials — captive portal only.
- NTP server hostname — captive portal or LAN HTTP only.
- DST flag and timezone — captive portal or LAN HTTP only.

## Quick recovery

Common "the clock is showing something I didn't expect" cases:

1. **Unexpected `HH:MM` value displayed.** Most likely you bumped `alarm` and are looking at alarm slot 1's set time. Press `alarm` repeatedly to step 1 → 2 → 3 → return to time display.
2. **Digits flashing.** You're mid-edit (`SET` cascade or `−` parameter menu). Press the entry button repeatedly to walk to the end and save out.
3. **Wholly wrong time.** Power-cycle (USB-C re-plug). The clock boots back to time display and re-syncs NTP within seconds.
4. **Avoid long-pressing `SET`** — that's the factory reset on most revisions, and clears WiFi/NTP config.

## Observed quirks

- The `systemSetup.html` page on both firmware revisions renders `summerTime` with `value='0' checked` regardless of the actual stored value. The form HTML reflects the static page template, **not** the device's current state. To confirm the saved DST value, observe displayed time against a known reference, or watch the `lastSync` timestamp.
- After a `setupSave` HTTP request the LCD has been observed to briefly flash an unrelated `HH:MM`-shaped value (e.g. `22:18`) before settling. This is a render glitch, not a stored alarm, and clears on the next NTP tick.
