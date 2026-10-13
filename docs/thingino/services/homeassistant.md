# Home Assistant Integration

Thingino includes a native Home Assistant integration via MQTT auto-discovery.
When enabled, the camera registers itself as a HA device automatically — no
manual YAML configuration required.

## Prerequisites

- Home Assistant with the **MQTT integration** enabled (Settings → Devices &
  Services → Add Integration → MQTT)
- An MQTT broker reachable by both the camera and HA (e.g. Mosquitto add-on)
- The MQTT broker address, set on the **Services → Home Assistant** page of
  the camera Web UI (`ha.mqtt.*`). `ha.mqtt.host` is required: `S93ha`
  refuses to start without it. Other empty `ha.mqtt` fields, except
  `tls_skip_verify`, fall back to the matching `mqtt_sub` value.

## Quick start

Open **Services → Home Assistant** in the camera Web UI, fill in the broker,
tick **Enable Home Assistant integration**, pick the entities and save.

Or from the camera shell:

```sh
jct /etc/thingino.json set ha.mqtt.host <broker>
jct /etc/thingino.json set ha.enabled true
/etc/init.d/S93ha restart
```

Within a few seconds HA will show a new device containing all enabled entities.

## Configuration keys (`ha.*` in `/etc/thingino.json`)

| Key | Default | Description |
|-----|---------|-------------|
| `enabled` | `false` | Start the HA daemon on boot |
| `discovery_prefix` | `"homeassistant"` | MQTT discovery prefix (match HA setting) |
| `state_interval` | `30` | Seconds between state polls |
| `discovery_interval` | `3600` | Re-publish discovery every N seconds (survives HA restarts) |
| `camera_interval` | `60` | Seconds between live view images (minimum 5) |
| `ota_check_interval` | `21600` | Seconds between GitHub firmware release checks (Firmware update entity) |
| `device_name` | `""` | Override device name (defaults to hostname) |
| `device_model` | `""` | Override device model string |
| `mqtt.host` | `""` | MQTT broker host |
| `mqtt.port` | `1883` | MQTT broker port |
| `mqtt.username` | `""` | MQTT username |
| `mqtt.password` | `""` | MQTT password |
| `mqtt.client_id_prefix` | `"thingino-ha"` | Prefix of the MQTT client IDs |
| `mqtt.use_ssl` | `false` | Connect to the MQTT broker with TLS using the system CA bundle |
| `mqtt.tls_skip_verify` | `false` | Skip broker certificate verification when TLS is enabled. Insecure; use only for self-signed or otherwise untrusted broker certificates |
| `enable_motion` | `true` | Binary sensor: motion detected |
| `enable_doorbell` | `false` | Binary sensor: doorbell button pressed |
| `enable_motion_guard` | `true` | Switch: enable/disable motion detection |
| `enable_ircut` | `true` | Switch: IR cut filter |
| `enable_daynight` | `true` | Select: day / night mode |
| `enable_daynight_state` | `true` | Sensor: day / night mode actually applied |
| `enable_privacy` | `true` | Switch: privacy screen |
| `enable_color` | `true` | Switch: color vs monochrome |
| `enable_ir850` | `true` | Switch: 850 nm IR LED |
| `enable_ir940` | `true` | Switch: 940 nm IR LED |
| `enable_white_light` | `true` | Switch: white light |
| `enable_floodlight` | `false` | Light: Wyze Floodlight v1 on/off + 1–100% brightness |
| `enable_lamp_socket` | `true` | Switch: Wyze Lamp Socket (also requires `lamp_socket.enabled`) |
| `enable_firmware_version` | `true` | Sensor: firmware version (e.g. "ciao+9a88c72") |
| `enable_firmware_timestamp` | `true` | Sensor: firmware build timestamp (e.g. "2026-07-24 03:04:32 UTC") |
| `enable_gain` | `false` | Sensor: ISP gain (proxy for lux) |
| `enable_brightness` | `true` | Sensor: scene brightness in % from `daynightd` (created only when `daynightd` is installed) |
| `enable_rssi` | `true` | Sensor: WiFi signal strength (dBm) |
| `enable_live_view` | `false` | Camera: still image refreshed every `camera_interval` |
| `enable_snapshot` | `true` | Button: take snapshot |
| `enable_reboot` | `false` | Button: reboot camera |
| `enable_ota` | `false` | Update: OTA firmware update |

`enable_daynight_state` and `enable_brightness` are not in the default
config and have no Web UI toggle. They are on unless set to `false`. The
Camera Config diagnostic sensor (`IMAGE_ID` from `/etc/os-release`) is always
published.
| `enable_ptz` | `false` | Buttons: PTZ up/down/left/right/home |

The firmware update entity (`enable_ota`) uses a cached GitHub release lookup:
`firmware/state` is still published every `state_interval`, but
`firmware/latest` is refreshed on `ota_check_interval` cadence with per-camera
jitter to avoid synchronized bursts across multiple cameras.

Disable individual entities you don't want:

```sh
jct /etc/thingino.json set ha.enable_reboot false
jct /etc/thingino.json set ha.enable_ptz true   # enable PTZ buttons
```

Floodlight brightness and motion activation are configured on the
[Floodlight](../floodlight.md) page (Settings → Floodlight).

## MQTT topic layout

All topics are scoped to `cameras/<id>/` where `<id>` is the camera's hostname
(e.g. `ing-wyze-cam3-2937`). The default hostname includes the SoC serial suffix
for uniqueness; override the hostname to retain HA configuration across hardware
swaps.

| Entity | State topic | Command topic |
|--------|-------------|---------------|
| Motion detected | `cameras/<id>/motion/state` | — |
| Doorbell | `cameras/<id>/doorbell/state` | — |
| Motion Guard | `cameras/<id>/motion_guard/state` | `cameras/<id>/motion_guard/set` |
| IR Cut Filter | `cameras/<id>/ircut/state` | `cameras/<id>/ircut/set` |
| Day/Night Mode | `cameras/<id>/daynight/state` | `cameras/<id>/daynight/set` |
| Day/Night State | `cameras/<id>/daynight_state/state` | — |
| Privacy Screen | `cameras/<id>/privacy/state` | `cameras/<id>/privacy/set` |
| Color Mode | `cameras/<id>/color/state` | `cameras/<id>/color/set` |
| IR LED 850 nm | `cameras/<id>/ir850/state` | `cameras/<id>/ir850/set` |
| IR LED 940 nm | `cameras/<id>/ir940/state` | `cameras/<id>/ir940/set` |
| White Light | `cameras/<id>/white/state` | `cameras/<id>/white/set` |
| Floodlight | `cameras/<id>/floodlight/state` | `cameras/<id>/floodlight/set` |
| Floodlight brightness | `cameras/<id>/floodlight_brightness/state` | `cameras/<id>/floodlight_brightness/set` |
| Lamp Socket | `cameras/<id>/lamp_socket/state` | `cameras/<id>/lamp_socket/set` |
| Live view | `cameras/<id>/live_view/image` (base64 JPEG) | — |
| Firmware version | `cameras/<id>/firmware_version/state` | — |
| Firmware build | `cameras/<id>/firmware_timestamp/state` | — |
| Gain | `cameras/<id>/gain/state` | — |
| Brightness | `cameras/<id>/brightness/state` | — |
| Camera config | `cameras/<id>/camera_config/state` | — |
| WiFi RSSI | `cameras/<id>/rssi/state` | — |
| Snapshot | — | `cameras/<id>/snapshot/set` |
| Reboot | — | `cameras/<id>/reboot/set` |
| Firmware installed | `cameras/<id>/firmware/state` | — |
| Firmware latest | `cameras/<id>/firmware/latest` | — |
| Firmware install | — | `cameras/<id>/firmware/set` (payload: `install`) |
| PTZ | — | `cameras/<id>/ptz/{up,down,left,right,home}/set` |
| Availability | `cameras/<id>/status` | — (`online` / `offline`) |

## Accepted command payloads

- **Switches**: `on` / `off` (also accepts `ON`/`OFF`, `1`/`0`)
- **Floodlight**: `on` / `off`; send `1`–`100` to the brightness command topic
- **Day/Night select**: `day` / `night` / `toggle`
- **Snapshot / reboot / PTZ buttons**: `1`
- **Firmware install**: `install`

## Gain sensor

The gain sensor (`enable_gain`) reads the ISP total gain from
`/proc/jz/isp/isp-m0` when available. It is disabled by default because the
proc interface is SoC-specific. Enable it after confirming the value makes
sense on your hardware:

```sh
jct /etc/thingino.json set ha.enable_gain true
/etc/init.d/S93ha restart
```

## Troubleshooting

Run discovery manually and watch the output:

```sh
ha-discovery
```

Watch all HA-related MQTT traffic:

```sh
mosquitto_sub -h <broker> -v -t 'homeassistant/+/thingino_#' -t 'cameras/<id>/#'
```

Check the daemon log:

```sh
logread | grep ha-
```

Force a state refresh without restarting the daemon:

```sh
ha-state
```
