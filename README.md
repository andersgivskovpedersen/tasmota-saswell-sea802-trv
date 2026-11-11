# Saswell SEA802 / Tuya TS0601 radiator valves on a Tasmota Zigbee gateway

Run Saswell SEA802 thermostatic radiator valves (Tuya `TS0601`, `_TZE200_3yp57tby` and `_TZE284_3yp57tby`) on an eWeLink **ZB-GW03** Ethernet Zigbee gateway running **Tasmota**, and control them from **Home Assistant** over MQTT — no Tuya hub, no cloud, no Zigbee2MQTT or ZHA.

What you get per valve in Home Assistant:

- room temperature and target temperature
- setting the target temperature
- "calling for heat" (heating on/off)
- battery-low warning *(unverified, see below)*
- a switch to turn the valve's built-in weekly schedule on or off

> **Status:** working in daily use with four valves. Several datapoints are still unverified and are marked as such. Corrections and confirmations are very welcome — please open an issue.

> **Disclaimer:** This was worked out by reverse engineering (watching the data the valves send and testing what they accept), combined with information from public sources on the internet, mainly the Zigbee2MQTT project (see [Credits and references](#credits-and-references)). None of it comes from manufacturer documentation, and public information may be incomplete or wrong for this exact model. This project is not affiliated with or endorsed by Saswell, Tuya, eWeLink or the Tasmota project. **Use it at your own risk.** It comes with no warranty. Writing wrong values to a valve can leave a room too cold or too warm, so keep an eye on your heating after making changes, especially in frost-prone rooms.

## Contents

- [`tasmota/tuya_trv.zb`](tasmota/tuya_trv.zb) — Tasmota Zigbee plugin that turns the valves' raw Tuya datapoints into named values
- [`homeassistant/trv_package.yaml`](homeassistant/trv_package.yaml) — Home Assistant package with MQTT sensors, binary sensors, switches and numbers for four valves

## How it fits together

```
 Radiator valves  ──Zigbee──>  ZB-GW03 (Tasmota)  ──MQTT──>  Broker  <──MQTT──  Home Assistant
 (Tuya cluster EF00)          tuya_trv.zb plugin                                 trv_package.yaml
```

Tuya valves don't use the standard Zigbee thermostat cluster. They send and receive everything through the Tuya-specific cluster `0xEF00` as numbered **datapoints** (DPs). Tasmota shows these raw as `EF00/xxyy`, where `xx` is the data type and `yy` the DP number in hex:

| `xx` | Type |
| --- | --- |
| `00` | Raw bytes |
| `01` | Boolean |
| `02` | Value (integer) |
| `04` | Enum |
| `05` | Bitmap |

So `EF00/0267` is DP `0x67` = 103, an integer. The plugin gives these codes readable names, and the Home Assistant package reads and writes them over MQTT.

## Hardware

| Part | Details |
| --- | --- |
| Gateway | eWeLink ZB-GW03 (ESP32 + Silicon Labs EFR32 radio, EZSP 6.7.10.0), Ethernet |
| Gateway firmware | Tasmota 14.5.0, the ZB-GW03 build by xsp1989 (comes pre-installed on many units) |
| Valves | Saswell SEA802 (also sold under other names), Tuya model `TS0601` |
| Valve manufacturer IDs | `_TZE200_3yp57tby` (app version 67) and `_TZE284_3yp57tby` (app version 77). Both behave the same. |

## 1. Gateway setup

### Template

This is the GPIO template the xsp1989 build ships with, and it works as is. Check yours under *Configuration → Template* before changing anything.

```json
{"NAME":"ZB-GW03","GPIO":[0,0,3552,0,3584,0,0,0,5793,5792,320,544,5536,0,5600,0,0,0,0,5568,0,0,0,0,0,0,0,0,608,640,32,0,0,0,0,0],"FLAG":0,"BASE":1}
```

| GPIO | Function |
| --- | --- |
| 2 / 4 | Zigbee Tx / Rx |
| 12 / 13 | Zigbee reset 2 / 1 |
| 14 / 15 | LED / link LED |
| 16, 18, 23 | Ethernet power, MDIO, MDC |
| 32 / 33 | I²C (EEPROM holding Zigbee data) |
| 34 | Button |

Note that Tasmota's ESP32 template array is **not** indexed by GPIO number. Edit pins in the web UI rather than in the JSON.

### Basic settings (Tasmota console)

```
MqttHost <your-broker>
Backlog Timezone 99; TimeDST 0,0,3,1,2,120; TimeSTD 0,0,10,1,3,60
```

Set the time zone for your location (the example is Central European Time with daylight saving). **This matters for the valves**: they take their clock from the gateway, and the built-in schedule runs on that clock. The xsp1989 firmware defaults to UTC+8.

## 2. Pair and name the valves

```
ZbPermitJoin 1
```

Put each valve in pairing mode (see its manual). When all are paired, name them and make Tasmota report by name:

```
ZbName 0x1234,TRV1
ZbName 0x5678,TRV2
SetOption83 1
```

Use the short addresses from `ZbInfo`. **Names are tied to the valve's IEEE address and survive re-pairing; short addresses do not.** The Home Assistant package relies on the names.

## 3. Install the Tasmota plugin

Upload [`tasmota/tuya_trv.zb`](tasmota/tuya_trv.zb) via *Consoles → Manage File System*. Tasmota loads every `.zb` file at boot. To load it right away:

```
ZbLoad tuya_trv.zb
ZbLoadDump
```

`ZbLoadDump` shows the mappings as Tasmota parsed them. Check that both temperature lines end in `,mul:10`.

Two pitfalls when editing the file:

- `#Z2Tv1` must be the **very first line**: no blank line, space or byte-order mark before it, or Tasmota reports `invalid header`.
- Multipliers must be written as `,mul:10`. The older plain `,10` form is silently ignored on Tasmota 14.x, and temperatures then show as 2.3 instead of 23.

## 4. Install the Home Assistant package

1. Make sure `configuration.yaml` loads packages:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy [`homeassistant/trv_package.yaml`](homeassistant/trv_package.yaml) to `/config/packages/`.
3. Replace `tasmota_XXXXXX` with your gateway's MQTT topic, and `TRV1`–`TRV4` with your valve names.
4. Restart Home Assistant.

Per valve you get:

| Entity | Type | Source |
| --- | --- | --- |
| `TRVn temperature` | sensor | DP 102 |
| `TRVn target` | sensor | DP 103 |
| `TRVn heating` | binary sensor | DP 3 |
| `TRVn battery low` | binary sensor | DP 105 *(unverified)* |
| `TRVn schedule` | switch | DP 108 (on = follows built-in schedule) |
| `TRVn set target` | number, 5–30 °C | writes DP 103 |

Type the target and press Enter: clicking the arrows sends every 0.5 °C step to the valve as a separate radio message.

## Datapoint reference

| DP | Tasmota code | Plugin name | Meaning | Status |
| --- | --- | --- | --- | --- |
| 3 | `EF00/0403` | `Heating` | 1 = valve calling for heat | Confirmed |
| 8 | `EF00/0108` | `WindowDetection` | Open-window detection on/off | Inferred |
| 10 | `EF00/010A` | `FrostProtection` | Frost protection on/off | Inferred |
| 27 | `EF00/021B` | `TempCalibration` | Temperature offset. Likely whole °C, −6 to +6 | Unverified |
| 40 | `EF00/0128` | `ChildLock` | Child lock on/off | Inferred |
| 101 | `EF00/0165` | `Dp101` | Probably valve on/off | Unverified |
| 102 | `EF00/0266` | `LocalTemperature` | Room temperature, °C × 10 | Confirmed |
| 103 | `EF00/0267` | `OccupiedHeatingSetpoint` | Target temperature, °C × 10, read and write | Confirmed |
| 104 | — | — | Valve position. Never reported to Tasmota | Not seen |
| 105 | `EF00/0569` | `BatteryLow` | Battery low flag (bitmap) | Unverified |
| 106 | `EF00/016A` | `AwayMode` | Away mode on/off | Inferred |
| 107 | — | — | Schedule type | Not seen |
| 108 | `EF00/016C` | `ScheduleEnabled` | 1 = follow built-in schedule, 0 = manual | Tested on one valve |
| 110–122 | `EF00/006E`–`007A` | — | Usage history (see below) | Decoded |
| 123–129 | `EF00/007B`–`0081` | — | Weekly schedule, Monday–Sunday | Decoded |
| 130 | `EF00/0182` | `ScaleProtection` | Anti-scale valve exercise on/off | Inferred |

*Confirmed* = observed and tested. *Inferred* = the value fits and matches the Saswell DP layout in Zigbee2MQTT, but hasn't been toggled to prove it. *Unverified* = from that layout only.

Commands for the writable ones:

```
ZbSend {"Device":"TRV1","Write":{"EF00/0267":215}}   # target 21.5 °C
ZbSend {"Device":"TRV1","Write":{"EF00/016C":0}}     # schedule off (manual)
ZbSend {"Device":"TRV1","Write":{"EF00/021B":-2}}    # offset −2 °C (unverified)
```

What does **not** work:

- Raw frames such as `"Send":"EF00_00/…"`: the valve replies `EF00/0000: 0` and ignores them.
- `"Send":{"<plugin name>":…}`: Tasmota answers *Unrecognized zigbee command*. Use `"Write"`.
- The Tuya data query `"Send":"EF00_03"`: the valve answers *SUCCESS* but sends nothing.

### Built-in weekly schedule (DP 123–129)

One DP per weekday. The factory default is the same every day:

```
04 0168 00C8 01E0 00A0 0438 00C8 0528 00A0
```

| Bytes | Meaning |
| --- | --- |
| `04` | 4 switch points |
| `0168` `00C8` | 360 min (06:00) → 200 (20.0 °C) |
| `01E0` `00A0` | 480 min (08:00) → 160 (16.0 °C) |
| `0438` `00C8` | 1080 min (18:00) → 20.0 °C |
| `0528` `00A0` | 1320 min (22:00) → 16.0 °C |

A valve with the schedule on overrides the target from Home Assistant at each switch point. If your valves drop to 16 °C at odd times, check the schedule switch and the gateway's time zone.

### Usage history (DP 110–122)

Three series, each at four time scales. Values are plain bytes, one per slot.

| Series | Per hour (24 bytes) | Per weekday (7) | Per day of month (31) | Per month (12) |
| --- | --- | --- | --- | --- |
| Target, °C | DP 110 `006E` | DP 112 `0070` | DP 113 `0071` | DP 114 `0072` |
| Room temperature, °C | DP 115 `0073` | DP 116 `0074` | DP 117 `0075` | DP 118 `0076` |
| Valve activity | DP 119 `0077` | DP 120 `0078` | DP 121 `0079` | DP 122 `007A` |

The hourly valve activity matched minutes of heating per hour in one test (3 minutes heating → value 3). Zigbee2MQTT's Saswell driver calls this series "motor history". The valves send the hourly arrays a few times a day. Home Assistant keeps its own history from the sensors, so these are mainly useful for debugging, for example to see what a valve did overnight and on which clock.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| Temperatures show as 2.3 instead of 23 | Plugin uses `,10` instead of `,mul:10`, or did not load (`ZbLoadDump`) |
| `ZbLoad … invalid header` | `#Z2Tv1` is not the very first line of the file |
| Raw `EF00/02xx` values instead of names | The valve's manufacturer ID is missing from the plugin's device lines (`ZbInfo` shows it) |
| `ZbSend: Done` but nothing changes | `Done` only means the radio took the frame. A valve accepting a new target replies with `OccupiedHeatingSetpoint` |
| `MAC_INDIRECT_TIMEOUT` | The sleeping valve didn't collect the message within about 7 s. Turn the knob to wake it and send again |
| Valve changes target at odd hours | Schedule is on (DP 108), and/or the valve clock is wrong. Valves only resync their clock on rejoin: take the batteries out briefly |
| HA entities stopped updating after re-pairing | Short addresses changed. Use `ZbName` + `SetOption83 1` |
| No value appears when requested | The valves don't answer Zigbee read requests. Values arrive when they change, or in a full burst when the valve rejoins |

## Credits and references

- Device identification: [Zigbee2MQTT issue #15766](https://github.com/Koenkk/zigbee2mqtt/issues/15766) (Saswell SEA802, `_TZE200_3yp57tby`)
- Tasmota Zigbee plugin format: [Tasmota PR #16252](https://github.com/arendst/Tasmota/issues/16252) and the [Tasmota Zigbee docs](https://tasmota.github.io/docs/Zigbee/)
- Saswell datapoint layout: Zigbee2MQTT / zigbee-herdsman-converters Saswell thermostat driver

## License

[CC0 1.0 Universal](LICENSE): no conditions. Use, copy, change and share it however you like, without asking and without attribution. Provided as is, without warranty of any kind.
