# vl-sim: simulated data sources for VizuaLog

`vl-sim` is a small docker image that provides an external test source for VizuaLog, currently UDP and MQTT. It´s main purpose is to have test tool to connect to over local network to test connection, data aquisition and logging long term.

| Sender | Sends | VizuaLog plugin |
|---|---|---|
| `vludpsender-cli` | Binary packets over UDP, laid out by a schema file | UdpCom |
| `vlmqttsender-cli` | JSON messages to an MQTT broker, grouped into topics | MQTT |

Image: `ghcr.io/omigatech/vl-sim` (linux/amd64 and linux/arm64, so it runs on Intel and
Apple Silicon Macs, Windows and Raspberry Pi).

## Quick start

**Docker:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) on macOS
or Windows, or Docker Engine on Linux. Download [`compose.yml`](compose.yml) into an empty
folder and run:

```bash
docker compose up -d
docker compose logs -f
```

That starts:

- `udp-vehicle`: 50 packets per second of vehicle telemetry to UDP port 45454 on your computer.
- `mosquitto`: an MQTT broker on port 1883.
- `mqtt-robot` and `mqtt-imu`: JSON messages to that broker on `sim/robot/...` and `sim/imu/...`.

Stop it all with `docker compose down`. Update to the latest image with
`docker compose pull && docker compose up -d`.

### In VizuaLog: UDP

Add a UdpCom interface listening on port **45454** with **Byte Order = Little** and
**Packet Size = 13**. VizuaLog doesn't read these from the schema file, so they must match
by hand. The sender prints them at startup (`docker compose logs udp-vehicle`).

Load the signal list from the same schema the sender uses. Copy it out of the image:

```bash
docker run --rm ghcr.io/omigatech/vl-sim cat /config/udp-config/vehicle_telemetry.json > vehicle_telemetry.json
```

### In VizuaLog: MQTT

Add an MQTT interface with broker `localhost`, port `1883`, and run discovery. Topics appear
as messages arrive, with one signal per JSON field: `sim/robot/joints.Joint1Angle`,
`sim/imu/status.cal.sys` and so on. No schema file is needed.

## Included schemas

List them with `docker run --rm ghcr.io/omigatech/vl-sim`.

| File | Signals | Use with |
| --- | --- | --- |
| `/config/udp-config/vehicle_telemetry.json` | 8: speed, RPM, fuel, coolant, 4 flags. 13-byte packet | UDP |
| `/config/udp-config/weather_station.json` | 3: temperature, humidity, rain. 9-byte packet | UDP |
| `/config/udp-config/industrial_robot.json` | 13: joint angles, gripper, cycle count, state flags | UDP |
| `/config/mqtt-config/industrial_robot_mqtt.json` | the robot, on 3 topics at 20 / 5 / 1 Hz | MQTT |
| `/config/mqtt-config/imu_mqtt.json` | 9-DOF IMU: euler angles, gyro, acceleration at 50 Hz; temperature, calibration, magnetometer at 1 Hz | MQTT |
| `/config/sim-config/*.json` | robot signal lists without a layout (no byte offsets) | MQTT |

Any schema works with the MQTT sender. The UDP sender needs one with byte offsets.

## Your own schema

Put a JSON file in a `schemas` folder next to `compose.yml` and uncomment the `my-device`
service in it. [`schemas/example_device.json`](schemas/example_device.json) is a starting point
that works with both senders.

```json
{
  "endianness": "Little",
  "mqtt": { "topics": [
    {"topic": "sim/pump/process", "rateHz": 10, "signals": ["Pressure", "FlowRate", "MotorSpeed"]},
    {"topic": "sim/pump/status",  "rateHz": 1,  "signals": ["*"]}
  ]},
  "dataSignals": [
    {"name": "Pressure", "dataType": "float32", "byteOffset": 0, "unit": "bar", "minValue": 0, "maxValue": 16},
    {"name": "Running",  "dataType": "bool",    "byteOffset": 12, "bit": 0}
  ]
}
```

### Signals (`dataSignals`)

| Field | | |
|---|---|---|
| `name` | required | Also the JSON key on MQTT and the signal name in VizuaLog |
| `dataType` | required | See below. A signal with a missing or unknown type is skipped, never guessed |
| `minValue`, `maxValue` | recommended | The range the simulated value moves in |
| `unit`, `description` | optional | Shown in VizuaLog |
| `byteOffset` | UDP only | Where the field starts in the packet |
| `bit` | UDP bools | 0–7; several bools can share one byte |

| `dataType` | Bytes | |
|---|---|---|
| `bool` | 1 | one bit of the byte, selected by `bit` |
| `int8`, `uint8` | 1 | |
| `int16`, `uint16` | 2 | |
| `int32`, `uint32` | 4 | |
| `float32` | 4 | IEEE-754 single |
| `float64` | 8 | IEEE-754 double |

`float`/`double`/`int`/`boolean` are accepted as `float32`/`float64`/`int32`/`bool`. Names
are case-insensitive.

The simulation needs no settings. Numbers wander as a smooth random walk inside
`minValue`..`maxValue`, integers are rounded, and bools hold for a few seconds, then flip.

### UDP layout

- `endianness`: `"Little"` (default) or `"Big"`. It's applied to every multi-byte field.
- `packetSize` (optional): otherwise the packet ends where the last field ends.
- Fields are written at their `byteOffset`; unused bytes are zero.

### MQTT topics (`mqtt`)

| Field | Default | |
|---|---|---|
| `topic` | required | Published to as-is (no `+` or `#`) |
| `signals` | `["*"]` | Signal names or `*` / `?` patterns, case-insensitive |
| `rateHz` | 10 | Messages per second on this topic, 0.01 to 1000 |
| `qos` | 0 | 0, 1 or 2 |
| `retain` | false | Retained values outlive the sender and show in VizuaLog as if live |

- A signal goes to the **first** topic that matches it, so a last topic with `"*"` takes the
  rest.
- Without an `mqtt` section, every signal goes to one topic, `vlsim/<file name>`.
- Each message is one JSON object, for example `{"Pressure":7.43,"FlowRate":95.06,"MotorSpeed":2398}`.
- A **dot in a name nests**. `cal.sys` is sent as `{"cal":{"sys":3}}`, which is how VizuaLog
  reads JSON paths, so the signal is found as `<topic>.cal.sys`.

Check a schema before running it: `-l` prints the layout (UDP) or every topic with a sample
message (MQTT), then starts sending.

```bash
docker run --rm -v "$PWD/schemas:/config/custom:ro" ghcr.io/omigatech/vl-sim \
  vlmqttsender-cli /config/custom/example_device.json --broker=host.docker.internal -l
```

## Sender options

Run either sender with `--help` for the full list, for example
`docker run --rm ghcr.io/omigatech/vl-sim vlmqttsender-cli --help`.

| `vludpsender-cli` | Default | |
|---|---|---|
| `-a, --address` | `127.0.0.1` | IP address or host name of the VizuaLog computer. A name is looked up once, at start |
| `-p, --port` | `45454` | |
| `-r, --rate` | `50` | Packets per second |
| `-l, --list-signals` | | Print the packet layout first |

| `vlmqttsender-cli` | Default | |
|---|---|---|
| `-b, --broker` / `-p, --port` | `localhost` / `1883` | |
| `-u, --username` | anonymous | Password from `VL_MQTT_PASSWORD` (or `--password`) |
| `-t, --topic-prefix` | | Prepended to every topic, for running one schema twice |
| `-r, --rate` / `--qos` | `10` / `0` | For topics that don't set their own |
| `-c, --client-id` | `vlmqttsender-<schema>-<random>` | Must be unique on the broker |
| `-l, --list-signals` | | Print topics and sample messages first |

Both senders take `--status-interval` (seconds between status lines in the log, default 60)
and `-q` (no status lines: startup, errors and, for MQTT, connection events only).

## Running it on another machine

A Linux server or Raspberry Pi can run the senders while VizuaLog runs on your desktop:

- **UDP:** set `--address` to the desktop's IP address.
- **MQTT:** in VizuaLog, use the server's address as the broker instead of `localhost`.

On Linux, `network_mode: host` instead of the default network makes the senders use the
server's own network directly. This is simplest when the broker or VizuaLog is reached over a
VPN such as Tailscale.

## Good to know

- **The broker in `compose.yml` has no password** and listens on port 1883 of your computer.
  Fine on a home or office network, but don't expose it to the internet.
- **While the broker is unreachable, MQTT messages are dropped, not queued.** VizuaLog stamps
  values with arrival time, so a replayed backlog would show up as a burst of stale data. The
  sender reconnects on its own (retrying after 1 s, then up to 30 s), and its status line
  counts what was dropped.
- **`docker compose stop` / `down` stop the senders cleanly.** Each prints a summary line.
- **Nothing in VizuaLog?** Check `docker compose logs <service>` first. On macOS, VizuaLog
  needs *Local Network* permission (System Settings → Privacy & Security). On Windows, allow
  VizuaLog through the firewall for UDP 45454. If port 1883 is taken by a broker you already
  run, remove the `mosquitto` service and point `--broker` at yours.

## Licence

The senders are MIT licensed. The image also contains Qt (LGPL-3.0), Eclipse Paho
(EPL-2.0/EDL-1.0) and Debian packages under their own licences. The full text is in the image
at `/usr/share/doc/vl-sim/`.
