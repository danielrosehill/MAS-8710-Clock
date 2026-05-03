# WiFi & NTP Setup via Captive Portal

When the unit is unconfigured, factory-reset, or otherwise unable to associate with a WiFi network, it broadcasts its own access point named `Config-XXXXXX`. This is the *only* way to onboard the older firmware revision (`MAS8710BN-` hostname units), since once those units join a WiFi network they firewall TCP/80 from the LAN side.

The newer-firmware revision (`ESP-` hostname units) leaves the same web UI listening on TCP/80 from the LAN side after WiFi join, so for those, see [`http-api.md`](http-api.md) — but the *captive-portal* form is identical.

## Captive-portal credentials

| Field | Value |
|---|---|
| SSID | `Config-XXXXXX` (per-device suffix) |
| Password | `33669999` |
| Gateway / config URL | `http://192.168.4.1/` |
| Phone DHCP lease | `192.168.4.X` |

The `XXXXXX` suffix is **not** a MAC tail — it's an opaque per-device identifier set at manufacture. Treat it as a unit serial.

## Procedure (vendor)

1. Power on the clock; wait for the blue LED on the back to flash.
2. **Turn off mobile data on the phone.** If the phone keeps mobile data on, it may abandon the captive AP (which has no upstream).
3. Connect the phone to `Config-XXXXXX` using password `33669999`. If prompted *"this WiFi has no internet, use anyway?"* — choose **Use** and disable auto-switch.
4. Browse to `http://192.168.4.1`. Three buttons:
   - **Configure WiFi** — set home SSID + password (case-sensitive).
   - **NTP Server** — set the NTP host the clock syncs against.
   - **System Setup** — DST toggle and integer UTC offset.
5. After SAVE the clock joins the home WiFi; blue LED goes solid when connected.

The home page shows `lastSync: YYYY.MM.DD HH:MM:SS` — useful for confirming sync after changes.

## Procedure (Linux desktop with WiFi NIC)

If you have a USB or built-in WiFi adapter on a workstation, you can drive the captive portal with `nmcli` + `curl` instead of using a phone. Useful for batch reconfigurations and for units that are otherwise inaccessible from the LAN.

```bash
# Confirm the device is broadcasting its captive AP
nmcli -t -f IN-USE,SSID,SIGNAL,SECURITY device wifi list --rescan yes | grep -i 'config-'

# Connect (the clock acts as a DHCP server, you'll get 192.168.4.X)
nmcli dev wifi connect "Config-XXXXXX" password "33669999" ifname <wifi-iface>

# Read current state
curl -s http://192.168.4.1/ | grep -oE 'lastSync:[^<]+'

# Set NTP server
curl -s "http://192.168.4.1/ntpSave?ntpServer=<your-ntp-server>"

# Set DST + timezone (integer hour offset, -12..+12)
curl -s "http://192.168.4.1/setupSave?summerTime=1&timeZone=2"   # UTC+2 with DST on
curl -s "http://192.168.4.1/setupSave?summerTime=0&timeZone=0"   # UTC, no DST

# Done
nmcli dev disconnect <wifi-iface>
```

## Captive AP intermittency

Empirically the captive AP is **not always broadcasting** even on a unit that's already on the LAN. Observations on a unit running the older firmware:

- Sometimes visible from a nearby scanner at -60 dBm; sometimes not visible at all.
- Re-appearance does not require power-cycling — wait, re-scan, retry.
- After a successful `setupSave`, the unit appears to drop the AP for a short period before re-broadcasting.

If the captive AP is genuinely gone and the unit is on the LAN, on older-firmware units there is no way back into configuration without a factory reset (long-press SET — see [`osd-buttons.md`](osd-buttons.md)).

## Dual-band caveat

If the home WiFi uses a single SSID for both 2.4 GHz and 5 GHz, the clock will fail to associate (it is 2.4 GHz only). Vendor recommends splitting the SSIDs and pointing the clock at the 2.4 GHz band specifically.
