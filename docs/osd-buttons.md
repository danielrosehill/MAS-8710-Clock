# OSD Button Reference

Settings that are **not** exposed via the web UI (LCD brightness, the three alarms, hourly chime, temperature offset) are reachable only via the physical buttons on the casework. The button semantics below are taken from the closest matching vendor manual ([DIY ESP8266 Networking Clock Kit, manuals.plus](https://manuals.plus/ae/1005006339767938)) and corroborated by behaviour observed on the units studied.

## Buttons

The casework has two buttons labelled `SET` and `UP` (some revisions also have a separate `DOWN`; if absent, the same `UP` cycles through values and wraps at the end).

## SET — single click

Cycles through the following options. Use `UP` to adjust the highlighted value; press `SET` again to advance to the next.

1. **Manual brightness** — discrete levels (typically 0–7 or 1–8 depending on revision).
2. **Auto brightness toggle** — when enabled, the firmware auto-dims the display to level 1 between **19:00 and 07:00** and restores the manually-set value otherwise. (This behaviour is documented in the system-setup HTML page footer too.)
3. **Temperature offset** — applies a signed offset to the displayed ambient temperature reading, to compensate for sensor placement next to the warm SoC.
4. **Hourly chime / beep** — on/off (revision-dependent; not present on all units).

After the last option, single-clicking `SET` exits setting mode and returns to the normal time display.

## SET — double click

Enters **alarm setting mode**. Three independent alarms are presented in sequence; for each:

1. **Enable flag** — `0` (off) or `1` (on). `UP` toggles.
2. **Hour** — 24h format. `UP` increments.
3. **Minute** — `UP` increments.

Press `SET` to advance to the next field / next alarm. After alarm 3 minute, the unit exits alarm setting mode and returns to the normal time display.

To **disable all alarms** without clearing the times: enter alarm-setting mode (double-click `SET`), and for each of the three alarms set the enable flag to `0` with `UP`, advancing past the hour/minute with `SET` without changing them.

## SET — long press

Behaviour varies by firmware revision. Common assignments seen in this product family:

- **Factory reset** (clears WiFi credentials, NTP server, DST/timezone, and re-enables the `Config-XXXXXX` captive-portal AP). Long-press duration typically ≥ 5 seconds.
- **Manual time adjustment mode** on offline units (no NTP). Not relevant when the clock is online.

If a long-press unexpectedly enters manual-time mode rather than factory reset, exit by pressing `SET` repeatedly to advance through hours/minutes/seconds without changing values, or wait for the timeout.

## What the OSD does *not* expose

- WiFi credentials — captive portal only.
- NTP server hostname — captive portal or LAN HTTP only.
- DST flag and timezone — captive portal or LAN HTTP only.

## Observed quirks

- The system-setup HTML page on both firmware revisions renders `summerTime` with `value='0' checked` regardless of the actual stored value. The form HTML reflects the static page template, **not** the device's current state. To confirm the saved value, observe the displayed time against a known reference, or rely on the `lastSync` timestamp behaviour.
- After a `setupSave` HTTP request the LCD has been observed to briefly flash an unrelated `HH:MM`-shaped value (e.g. `22:18`) before settling. This is a render glitch, not a stored alarm, and clears on the next NTP tick.
