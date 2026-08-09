<h2 align="center">
  <a href="https://reolink.com"><img src="./logo.png" width="200"></a>
  <br>
  <i>Home Assistant Reolink NVR/cameras custom integration</i>
  <br>
</h2>

<p align="center">
  <a href="https://github.com/custom-components/hacs"><img src="https://img.shields.io/badge/HACS-Custom-orange.svg"></a>
  <img src="https://img.shields.io/github/v/release/materko/reolink_cctv?display_name=tag&include_prereleases&sort=semver" alt="Current version">
</p>

`reolink_cctv` integrates [Reolink](https://www.reolink.com/) NVRs and cameras into Home Assistant. Connecting to an **NVR** gives you a single integration entry with all of its channels as sub-devices, live streams, motion/AI detection sensors, configuration switches, and browsable access to the NVR's recordings.

---

## Why this fork exists: NVR firmware 2.x

Home Assistant ships an official `reolink` integration, and on current firmware it is the better choice — it is maintained by Reolink themselves and supports far more devices and features than this project.

This fork exists for devices the official integration cannot serve: NVRs whose firmware is too old and for which **no newer firmware was ever released**. Its documentation simply says to update the device first, which is not an option when there is nothing to update to. Verified here on an **RLN8-410-E running 2.0.0.269**, where the official integration can neither list nor play the NVR's recordings.

The firmware-2.x behaviours this fork works around, all verified against a real device:

- **`Search` fails across a month boundary.** Any VOD search whose start and end fall into different calendar months is answered with `rspCode -12` ("get config failed"), so searches are split per month and the results merged. The official stack trips over exactly this: to save requests it deliberately queries a range spanning two months.
- **`GetDevInfo` has no `exactType` field**, only `type`, so NVR detection has to accept both. (Upstream `reolink_aio` handles this too.)
- **Recordings play only over raw RTMP.** The `/flv?...playback.bcs` wrapper returns 404 and the `Playback` CGI command answers "not support"; the device's own web UI uses `rtmp://<host>:1935/bcs/playback.bcs?channel=..&type=..&start=<YYYYMMDDHHMMSS>&seek=0&token=..`, where `start` comes from the `PlaybackTime` field of the `Search` result.
- **RTMP accepts token authentication only.** User/password RTMP URLs fail with an I/O error, for live streams as well as for playback.
- **ONVIF motion notifications carry no channel number.** The NVR reports "something moved" without saying which camera, so on such a notification the integration queries the motion state of *all* channels and reports whichever ones are active. This is what makes multi-channel motion detection work here.

Because the upstream `reolink-ip` package is abandoned (its repository is gone and it declared dependencies that no longer install), the library is bundled inside this integration under `custom_components/reolink_cctv/reolink_ip/` and patched there. The integration therefore installs **no pip requirements** at all.

---

## Requirements

- Home Assistant **2025.1** or newer (declared in `hacs.json`). Developed and verified against **2026.8.1** on Python 3.14.
- A camera/NVR user of type **Administrator**. A "Guest" user can read switch states but cannot change them and cannot call the services.
- **ONVIF enabled** on the device. It is off by default on some models and can sometimes only be enabled from a monitor attached to the NVR, not from the web UI or the phone app. A firmware upgrade may reset it.
- Home Assistant's **Internal URL** set to a reachable `http://<ip>:<port>` address — not an mDNS name.

> :warning: **Motion detection needs Home Assistant reachable over plain HTTP on the local network.** Reolink firmware fails to deliver ONVIF notifications to an HTTPS webhook, so if your internal URL is HTTPS, motion events will never arrive.

> :warning: **Do not run this together with the official `reolink` integration for the same device.** Both subscribe to the device's ONVIF events and will fight over them. Disable or remove one of them.

---

## Installation

### HACS ([how to install HACS](https://hacs.xyz/docs/setup/prerequisites))

1. Open **HACS** in the Home Assistant menu
2. Open the top-right menu (three dots) → **Custom repositories**
3. Paste `https://github.com/materko/reolink_cctv`, choose category **Integration**, click **Add**
4. Open the newly listed **Reolink IP NVR/camera** and click **Download**

### Manual

```bash
# Download a copy of this repository
wget https://github.com/materko/reolink_cctv/archive/master.tar.gz

# Unzip the archive
tar -xzf master.tar.gz

# Move the reolink_cctv directory into the custom_components directory of your Home Assistant install
mv reolink_cctv-master/custom_components/reolink_cctv <home-assistant-config-directory>/custom_components/

# Clean up
rm -rf ./reolink_cctv-master ./master.tar.gz
```

> :warning: **Restart Home Assistant after installing, and clear your browser cache** — otherwise the integration may not show up in the list.

Then go to **Settings → Devices & services → Add integration → Reolink IP NVR/camera** and enter username, password and host. Port and HTTPS are detected automatically; if the connection fails, the form is re-shown with explicit **port** and **use HTTPS** fields.

---

## Configuration options

After setup, all of the following are available under **Configure** on the integration entry:

| Option | Default | Description |
| :----- | :------ | :---------- |
| External IP/domain | *(empty)* | Host used to build the `last_record_url` attribute for links reachable from outside your LAN. Unused when empty. |
| External port | *(empty)* | Port used together with the above. |
| Protocol | `rtmp` | `rtmp` or `rtsp` — the protocol used for live streams. |
| Stream | `sub` | `main`, `sub` or `ext` — which stream the recordings and the default camera use. `ext` exists only with RTMP; selecting it under RTSP silently falls back to `sub`. |
| Motion sensor off delay | `5` | Seconds the motion sensor stays "Detected" after the last detection. `0` disables the delay. |
| Motion sensor force-off timeout | `0` | Workaround for camera models that never send the "motion ended" notification: force the sensor back to "Clear" after this many seconds. `0` disables the workaround and saves some resources. |
| ONVIF subscription watchdog interval | `60` | Seconds between subscription health checks (`0`–`180`). `0` disables the watchdog. |
| Playback range (days) | `10` | How far back recordings are searched, and how old thumbnails may get before cleanup removes them. |
| Playback thumbnail path | `<config>/.storage/reolink_cctv/<device-id>` | Where event thumbnails are stored. |
| Timeout | `60` | Device request timeout in seconds (`1`–`120`). |

---

## Entities

Which entities appear depends on what the device reports as supported, so the lists below are the maximum — a plain NVR with non-AI cameras will show only a subset.

### Cameras

For **every channel** an entity is created per stream type: `sub`, `main`, `snapshots`, plus `ext` when the protocol is RTMP. **Only the `sub` camera is enabled by default**; enable the others in the entity registry if you need them.

`snapshots` is not a video stream — it repeatedly fetches still images. It is the fastest and most reliable way to show a camera when the high-resolution stream is too laggy to be usable.

### Binary sensors

| Sensor | Created when |
| :----- | :----------- |
| Motion | always, one per channel |
| Person / Vehicle / Pet / Face detected | the channel reports AI support for that object type |
| Visitor | the channel is a doorbell |

**Availability is used as a status indicator on purpose.** If the ONVIF subscription is not alive, detection sensors show **Unavailable** rather than "Clear", so a broken subscription is visible without reading the log. The watchdog keeps trying to re-subscribe; while it fails, it also polls for motion, so real motion still flips the sensor to "Detected" — and back to "Unavailable" afterwards, to keep signalling the problem.

### Sensor

A **Last record** sensor per channel (only when the device has a hard drive). Its state is the start timestamp of the most recent recorded chunk. It also writes that event's thumbnail automatically.

Attributes: `oldest_day`, `vod_event_id`, `has_thumbnail`, `thumbnail_path`, `last_record_url`, `duration`.

It is called "last record", not "last event", because with 24/7 recording it points at the current recording chunk, which may have started an hour or two ago and is not a motion event as such.

> :warning: `last_record_url` embeds the device login and password, because the Reolink API requires credentials in the URL. Only ever send it over encrypted channels, and only as an HTTPS link.

Example notification automation:

```yaml
automation:
  - alias: Notify mobile app
    triggers:
      - trigger: state
        entity_id: binary_sensor.front_left_motion
        to: "on"
    actions:
      - action: notify.mobile_app_<your_device_id_here>
        data:
          message: "Motion event"
          data:
            video: "{{ state_attr('sensor.front_left_last_record', 'last_record_url') }}"
```

### Switches

| Switch | Scope | Created when the device supports |
| :----- | :---- | :------------------------------- |
| Email | device | email alerts |
| FTP | device | FTP upload |
| Push notifications | device | push to Android/iOS |
| Recording | device | recording to disk |
| Record audio | per channel | audio |
| IR lights | per channel | infrared lights |
| Spotlight | per channel | a white LED spotlight |
| Doorbell light | per channel | a doorbell power LED |
| Siren | per channel | an audio alarm |

Reolink's API does not cleanly separate device-wide from per-camera settings, so the first four are created once for the whole device and the rest per channel.

---

## Recordings (media browser)

Under **Media → Reolink IP NVR/camera** you can browse per camera → per day → per recording, and play a recording back. Playback is re-wrapped into HLS by Home Assistant's stream component, so it plays in any browser.

With continuous 24/7 recording, entries are the NVR's own chunks — typically one hour each — not motion clips. Thumbnails are **deliberately not generated for every entry**: a chunk that has a thumbnail contained a motion event (the Last record sensor wrote one), a chunk without a thumbnail had no motion at all. That is a quick visual filter when scrolling a 24/7 archive.

Old thumbnails are cleaned up while browsing, but browsing alone is not enough — set up a periodic call of the `cleanup_thumbnails` service if you have many motion events across many cameras, or the thumbnails will fill your Home Assistant disk.

---

## Services

All standard camera [services](https://www.home-assistant.io/integrations/camera/#services) work. Additionally:

### `reolink_cctv.set_sensitivity`

Set the motion detection sensitivity. Either all time-schedule presets at once, or one specific preset.

| Field | Required | Description |
| :---- | :------- | :---------- |
| `entity_id` | yes | The camera to control. |
| `sensitivity` | yes | `1` (low) to `50` (high). |
| `preset` | no | Time-schedule preset to change. When omitted, all presets are changed. |

### `reolink_cctv.set_daynight`

| Field | Required | Description |
| :---- | :------- | :---------- |
| `entity_id` | yes | The camera to control. |
| `mode` | yes | `AUTO`, `COLOR` or `BLACKANDWHITE`. |

### `reolink_cctv.set_backlight`

Compensates for differences between dark and bright objects using BLC or WDR. It can improve clarity in high-contrast scenes, but test it at different times of day and night before keeping it.

| Field | Required | Description |
| :---- | :------- | :---------- |
| `entity_id` | yes | The camera to control. |
| `mode` | yes | `BACKLIGHTCONTROL`, `DYNAMICRANGECONTROL` or `OFF`. |

### `reolink_cctv.ptz_control`

Only available on cameras that report PTZ support.

| Field | Required | Description |
| :---- | :------- | :---------- |
| `entity_id` | yes | The camera to control. |
| `command` | yes | `AUTO`, `DOWN`, `FOCUSDEC`, `FOCUSINC`, `LEFT`, `LEFTDOWN`, `LEFTUP`, `RIGHT`, `RIGHTDOWN`, `RIGHTUP`, `STOP`, `TOPOS`, `UP`, `ZOOMDEC`, `ZOOMINC`. |
| `preset` | no | Preset ID for the `TOPOS` command. Available presets are listed in the camera's `ptz_presets` attribute. |
| `speed` | no | Movement speed. Not applicable to `STOP` and `AUTO`. |

**The camera keeps moving until `STOP` is sent.**

### `reolink_cctv.cleanup_thumbnails`

Removes stored VoD thumbnails older than the **Playback range** option, freeing disk space. Only available on cameras with playback capability.

| Field | Required | Description |
| :---- | :------- | :---------- |
| `entity_id` | yes | The camera whose thumbnails to clean. |

> The service schema still accepts an `older_than` field, but the implementation ignores it — the cutoff always comes from the Playback range setting.

---

## Device automations

Besides the entities, the integration registers device-level building blocks usable from the automation editor:

- **Trigger** — *"&lt;camera&gt; last record: new video file"*, fires when a new recording appears.
- **Conditions** — *"&lt;camera&gt; last record: has thumbnail"* / *"no thumbnail"*.
- **Action** — takes a snapshot of a camera and stores it as `snapshot.jpg` in the thumbnail directory. Home Assistant's built-in `camera.snapshot` service does the same thing with more control, so this exists mostly for convenience.

---

## Known device quirks

**Motion notifications do not identify the channel.** The NVR announces motion without saying which camera it came from. The integration handles this by querying every channel's motion state whenever such a notification arrives, so all channels do report motion. Some firmware sends "rich" ONVIF notifications that already contain the detected AI object type; those are used directly, without asking the device anything, which is faster and lighter.

**Channel count can be wrong.** An 8-channel RLN8-410 reports itself as having **12** channels, so do not be surprised by a longer channel list than your NVR has ports.

**No logo in the Home Assistant UI.** Logos for integrations live in Home Assistant's central `brands` repository and are loaded as external links, so a custom integration cannot ship one locally. Nothing is broken; the icon is simply missing.

### Troubleshooting

See [TSHOOT.md](TSHOOT.md) for log-collection instructions. The usual suspects:

- Internal URL not set, or set to an mDNS name instead of an IP.
- Internal URL using HTTPS — ONVIF notifications will not arrive (see Requirements).
- ONVIF disabled on the device, or reset by a firmware upgrade.
- The device user is not an Administrator.

---

## Untested / unsupported

The following were listed as unsupported by the upstream project and have **not** been re-verified in this fork:

- Battery-powered cameras
- B800, B400, D400 (via NVR only)
- Lumus

Direct connections to a **standalone camera** (rather than to an NVR) are implemented but are not what this fork is exercised on — it is developed against an NVR connection.

---

## Credits

Originally written by [JimStar](https://github.com/JimStar) as a rewrite of `reolink_dev`. This fork continues it with support for current Home Assistant releases and for NVR firmware 2.x.
