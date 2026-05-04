# Troubleshooting

Diagnostic flows for the most common "the clock is showing the wrong thing" cases.

## Symptom: shows a wrong but plausible-looking `HH:MM` after replug

Example: unit was UTC, you unplug and replug, and a few seconds later it's stuck on `06:28` or similar. Two real causes, ranked by likelihood:

### Cause 1 — Display is sitting in alarm-slot view, not normal time view

On the older `MAS8710BN*` firmware revision there's a known quirk where the unit can land in alarm-edit display after some boot paths and stay there until any button is pressed. The displayed value is the *set time of alarm 1*, not the current time. The CR1220 and the actual time-keeping are fine — only the display state is wedged.

**Test:** press `alarm` once.

- If the displayed value jumps to a *different* `HH:MM` (or `-2-` briefly flashes) — confirmed: it was alarm slot 1. Press `alarm` two more times to walk slot 2 → slot 3 → exit, and the display returns to real time.
- If nothing changes — it's not this; move to Cause 2.

**Workaround:** poke `alarm` after every replug, or live with it. The underlying clock is fine.

### Cause 2 — Config (timezone / DST / NTP server) didn't persist across the power cycle

If `setupSave`-style config was actually wiped, the unit comes back with defaults (`summerTime=0, timeZone=0`) — which means the displayed time is whatever the configured NTP server returned, with no timezone offset applied. On a unit that was deliberately set to UTC this looks like nothing changed; on a unit set to local time it's now showing UTC.

**Test:** open the captive portal (`Config-XXXXXX`, password `33669999`, gateway `http://192.168.4.1/`) — see [`wifi-setup.md`](wifi-setup.md). Check whether NTP server, timezone, and DST are still set as you last configured them.

- If they're blank or defaulted to `0` — flash isn't persisting `setupSave` writes on this unit. Reflashing with a known-good firmware (Tasmota / ESPHome — see [`LAN-Clocks/docs/flashing-guide.md`](https://github.com/danielrosehill/LAN-Clocks/blob/main/docs/flashing-guide.md)) is the realistic fix.
- If they're still set correctly — config persistence is fine; the symptom was something else (probably Cause 1, or a transient NTP miss).

## Symptom: time briefly wrong after replug, then corrects within a minute

Normal behaviour for a unit with a missing or dead **CR1220 backup cell**. The RTC starts from a default value (often `00:00` or a stale stored value) on cold boot, then NTP corrects it within seconds of WiFi association.

**Fix:** replace the CR1220. Note that the cell does *not* affect config persistence (NTP server, WiFi credentials, timezone, DST) — those live in the ESP8285's flash and survive regardless. The cell only keeps the running RTC time warm.

## Symptom: wholly wrong time, no NTP correction

Most likely WiFi association failed. Check:

1. The `WIFI` LED on the rear — solid = associated, blinking = trying, off = no creds saved or hardware fault.
2. The `同步` (sync) LED — fires briefly each time NTP completes successfully. If it never lights, NTP itself is failing (server unreachable, or set to a hostname that doesn't resolve).
3. On older `MAS8710BN*` firmware, LAN-side TCP/80 is closed — diagnose only via captive portal or LEDs.
4. On newer `ESP-*` firmware, fetch `http://<clock-ip>/` and look for the `lastSync:` value — if it's old or missing, NTP isn't reaching.

## Symptom: brownout / random resets under shared power

The ESP8285 pulls ~300–400 mA spikes during WiFi TX. A shared USB hub, splitter cable, or under-spec wall charger can sag below the AMS1117's 4.3 V minimum input during another device's load spike, causing the unit to reset. Repeated brownouts look like "shows random time" because the RTC value gets clobbered before NTP can correct it.

**Fix:** dedicated 5 V / ≥1 A USB-C source per unit. The reliable product class is a **multi-port GaN charger** with isolated per-port regulation (Anker, UGREEN, Baseus desktop chargers). Avoid bus-powered USB hubs and 1-to-N splitter cables.

## Symptom: digits are flashing

You're mid-edit somewhere — `SET` cascade, `−` parameter menu, or alarm-edit. Press the entry button (`SET` / `−` / `alarm`) repeatedly to walk through the remaining fields and save out, or wait for the timeout. Power-cycling is also safe — it boots back to time display.

## Quick recovery cheat sheet

| Display shows | First thing to try |
|---|---|
| Plausible `HH:MM` but wrong | Press `alarm` once; if it changes, walk slots and exit |
| Flashing digits | Press the entry button repeatedly to save out |
| Wholly wrong, no correction | Check `WIFI` and `同步` LEDs, then captive portal |
| `00:00` briefly after replug | CR1220 is dead — replace |
| Random changes under load | Brownout — dedicated power supply |
| Anything else genuinely stuck | USB-C unplug for 30 s, replug |

## When to suspect the firmware vs the hardware

Use this rough split:

- **Hardware suspects** (CR1220, power supply, brownout): symptoms are *transient* — wrong briefly, then corrects, or correlated with other devices' load.
- **Firmware suspects** (config not persisting, alarm-view wedge): symptoms are *deterministic* — same wrong state on every replug, doesn't self-correct without intervention.

Hardware fixes are cheap and reversible. Firmware fixes mean reflashing and accepting that the LCD-driver protocol is currently undecoded, so the display will go dark until a custom ESPHome component is written.
