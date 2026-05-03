# LAN HTTP API (newer firmware)

The newer firmware revision of the MAS-8710 (units that announce as `ESP-XXXXXX` via DHCP) leaves the captive-portal web UI **listening on TCP/80 from the LAN side** after WiFi join. Older firmware (`MAS8710BN-` hostname) closes/filters the port and is only reachable via the captive portal.

The two UIs serve identical HTML and accept the same parameters. This document is for the LAN-side variant.

## Discovery

```bash
# Confirm port 80 is listening
nmap -Pn -p 80 <clock-ip>

# A reachable unit returns:
#   80/tcp open  http
```

If port 80 is `closed` or `filtered`, you're looking at older firmware — fall back to the captive-portal flow in [`wifi-setup.md`](wifi-setup.md).

## Endpoints

No authentication on any path. All forms are HTML `<form action="…">` with no method, so the browser submits as `GET` with query-string parameters. `curl` works directly.

| Path | Purpose | Notable response |
|---|---|---|
| `GET /` | Home page. Three buttons + `lastSync: YYYY.MM.DD HH:MM:SS`. | HTML with the lastSync string in a `<div style='color:yellowgreen'>`. |
| `GET /configWiFi.html` | WiFi setup page. Below the form, returns a **live nearby-SSID scan** with signal levels in dBm. | Useful for placement/diagnostics — see "Scanning" below. |
| `GET /wifiSave?wifiName=<ssid>&wifiPassword=<pw>` | Save new WiFi credentials. Case-sensitive. | Confirmation page that auto-redirects. |
| `GET /configNtp.html` | NTP setup page. Includes a vendor list of public pools (Aliyun, Apple, NIST, etc.). | — |
| `GET /ntpSave?ntpServer=<host-or-ip>` | Set the NTP server. Hostname or IP both accepted. | `<h3>NTP Server: <value></h3>` echoed back, then redirect. lastSync update visible within seconds if the new server is reachable. |
| `GET /systemSetup.html` | DST toggle + integer UTC offset. | `summerTime` is a radio (0/1); `timeZone` is a `<select>` (-12..12). |
| `GET /setupSave?summerTime={0\|1}&timeZone={-12..12}` | Apply DST and timezone. | Always returns `<h3>Setup Saved!</h3>` regardless of which (or how many) parameters are sent — see "Caveats" below. |

There is no enumerable `Index` and no other paths. Confirmed by enumerating common ESP/Arduino paths (`/status`, `/info`, `/version`, `/api`, `/admin`, `/reboot`, `/reset`, `/ota`, `/update`, etc.) — all return `404`.

## Caveats

### `setupSave` is opaque

The endpoint returns `<title>Setup Saved!</title>` for **any** request, including:

- Empty (`/setupSave`)
- Unknown parameters (`/setupSave?nonsense=xyz`)
- Plausible-but-undocumented parameters (`/setupSave?brightness=5`, `/setupSave?alarm=0`, etc.)

This means response inspection cannot tell you whether a parameter is accepted. Worse, **bare or partial requests appear to apply default values** — sending `/setupSave` with no params has been observed to momentarily set `summerTime=0, timeZone=0` on the device, briefly displaying UTC before the next NTP sync corrects the time component (the timezone setting itself persists).

**Always send both `summerTime` and `timeZone` together.** Never call `setupSave` without parameters.

### Brightness, alarm, and chime are not exposed

The vendor's English manual documents three independent alarms, manual + auto LCD brightness, and a temperature offset — all settable via the OSD `SET` / `UP` buttons. None of these are reachable through the HTTP UI on either firmware revision. See [`osd-buttons.md`](osd-buttons.md).

### Timezone model is integer hours only

Half-hour and 45-minute zones (India, Nepal, parts of Australia) are unrepresentable. DST is a global on/off — not rule-aware. For a UTC+2 zone with DST: `summerTime=1, timeZone=2` during DST; flip `summerTime=0` when the country exits DST. For a fixed UTC display: `summerTime=0, timeZone=0`.

## Useful invocations

```bash
# Last sync timestamp
curl -s http://<clock-ip>/ | grep -oE 'lastSync:[^<]+'

# Point at LAN NTP server
curl -s "http://<clock-ip>/ntpSave?ntpServer=<your-ntp-server>"

# Israel local with DST on
curl -s "http://<clock-ip>/setupSave?summerTime=1&timeZone=2"

# UTC, no DST
curl -s "http://<clock-ip>/setupSave?summerTime=0&timeZone=0"
```

## Scanning nearby APs through the device

The WiFi-config page returns the device's own neighbour scan in HTML — handy for placement debugging or for inventorying other captive-portal SSIDs broadcast by sibling clocks:

```bash
curl -s http://<clock-ip>/configWiFi.html \
  | grep -oE '<tr><td>[^<]+</td><td>-?[0-9]+dBm</td></tr>'
```

This was how we confirmed that two sibling clocks were within range and continuously broadcasting their `Config-XXXXXX` APs.
