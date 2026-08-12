# Sevcon TPDO Configuration

This document captures the Sevcon Gen4 TPDO setup used by the Buddy DA64 firmware.

Source references:

- Object dictionary export: `docs/Master_Object_Dictionary_Database.xhtml`
- Screenshots: `docs/tdpo1.PNG` through `docs/tdpo5.PNG`
- Firmware consumer: `src/CarComputer/da64/da64.cpp`
- CAN constants: `src/CarComputer/da64/da64_config.h`

The Sevcon motor CAN bus is classic CAN at `250 kbps`. DA64 listens and transmits on its second MCP2517.

| Frame | COB-ID | Purpose |
|---|---:|---|
| TPDO3 | `0x455` | Motor temperature and vehicle speed. |
| TPDO4 | `0x368` | Capacitor voltage, heatsink temperature, motor battery current, traction drive state, and switch feedback flags. |

All configured TPDO screenshots show:

| Setting | Value |
|---|---|
| PDO type | Cyclic synchronous, SYNC |
| Syncs per transmit | `1` |
| Local node sync interval | `20.0 ms` |
| Inhibit timer | `0 ms` |
| Event timer | `0 ms` |

Multi-byte PDO values are little-endian.

## TPDO Summary

| TPDO | Communication object | Mapping object | Configured COB-ID | Bits used |
|---|---:|---:|---:|---:|
| TPDO1 | `0x1800` | `0x1A00` | `0x339` | 64 |
| TPDO2 | `0x1801` | `0x1A01` | `0x362` | 64 |
| TPDO3 | `0x1802` | `0x1A02` | `0x455` | 64 |
| TPDO4 | `0x1803` | `0x1A03` | `0x368` | 52 |
| TPDO5 | `0x1804` | `0x1A04` | `0x343` | 64 |

## TPDO1 `0x339`

Screenshot: `docs/tdpo1.PNG`

| Byte | Bits | Object | Signal |
|---:|---:|---|---|
| 0..1 | 16 | `0x4600:05` | Target Id |
| 2..3 | 16 | `0x4600:06` | Target Iq |
| 4..5 | 16 | `0x4600:07` | Id |
| 6..7 | 16 | `0x4600:08` | Iq |

DA64 does not currently consume this TPDO.

## TPDO2 `0x362`

Screenshot: `docs/tdpo2.PNG`

| Byte | Bits | Object | Signal |
|---:|---:|---|---|
| 0..1 | 16 | `0x4600:09` | Ud |
| 2..3 | 16 | `0x4600:0A` | Uq |
| 4..5 | 16 | `0x4600:0B` | Voltage modulation |
| 6..7 | 16 | `0x4602:1D` | Measured inductance |

DA64 does not currently consume this TPDO.

## TPDO3 `0x455`

Screenshot: `docs/tdpo3.PNG`

This frame is consumed by DA64.

| Byte | Bits | Object | Signal | Firmware type | Firmware scale |
|---:|---:|---|---|---|---|
| 0..1 | 16 | `0x4600:03` | Motor Temperature 1, measured TH1 | `int16 LE` | `1 degC/bit` |
| 2..3 | 16 | `0x2220:00` | Throttle Input Voltage | `int16 LE` | `0.00390625 V/bit` |
| 4..5 | 16 | `0x2620:00` | Throttle Value | ignored | ignored |
| 6..7 | 16 | `0x2721:00` | Vehicle Speed | `int16 LE` | `0.0625 km/h/bit` |

DA64 handling:

- Accepts standard 11-bit CAN frame `0x455`.
- Requires at least 8 bytes.
- Reads bytes `0..1` as signed motor temperature.
- Reads bytes `2..3` as signed throttle input voltage for physical footswitch relay gating.
- Reads bytes `6..7` as signed speed.
- Converts speed with `0.0625 km/h/bit`.
- Uses absolute value for dashboard speed, then clamps to `0..255 km/h`.

## TPDO4 `0x368`

Screenshot: `docs/tdpo4.PNG`

This frame is consumed by DA64.

| Byte/bit | Bits | Object | Signal | Firmware type | Firmware scale |
|---:|---:|---|---|---|---|
| 0..1 | 16 | `0x5100:03` | Capacitor Voltage | `uint16 LE` | `0.0625 V/bit` |
| 2 | 8 | `0x5100:04` | Heatsink Temperature | `int8` | `1 degC/bit` |
| 3..4 | 16 | `0x5100:02` | Battery Current | `int16 LE` | `0.0625 A/bit` |
| 5 | 8 | `0x2720:02` | Traction Drive State | cached | raw enum/status byte |
| 6 bit 0 | 1 | `0x2130:00` | Footbrake switch | bit | `1=active` |
| 6 bit 1 | 1 | `0x2121:00` | Forward switch | bit | `1=active` |
| 6 bit 2 | 1 | `0x2122:00` | Reverse switch | bit | `1=active` |
| 6 bit 3 | 1 | `0x2123:00` | FS1 switch | bit | `1=active` |
| 6 bits 4..7 | 4 | unused | unused | ignored | ignored |
| 7 | 8 | unused | unused | ignored | ignored |

DA64 handling:

- Accepts standard 11-bit CAN frame `0x368`.
- Requires at least 7 bytes.
- Reads capacitor voltage from bytes `0..1`.
- Reads heatsink temperature from byte `2`.
- Reads battery current from bytes `3..4`.
- Caches traction drive state from byte `5`.
- Reads switch feedback flags from byte `6`, low nibble only.
- Ignores byte `6` high nibble and byte `7`.

The low-nibble switch flags are bridged to vehicle CAN and dashboard diagnostics:

| TPDO4 flag | Meaning |
|---:|---|
| bit 0 | Footbrake active |
| bit 1 | Forward switch active |
| bit 2 | Reverse switch active |
| bit 3 | Footswitch active |

## TPDO5 `0x343`

Screenshot: `docs/tdpo5.PNG`

| Byte | Bits | Object | Signal |
|---:|---:|---|---|
| 0..3 | 32 | `0x6080:00` | Maximum motor speed |
| 4..7 | 32 | `0x606C:00` | Velocity |

DA64 does not currently consume this TPDO.

## Physical Footswitch Relay Pedal Gating

DA64 does not transmit Sevcon RPDO FS1. The physical `FOOTSWITCH_RELAY` remains the only FS1/pedal enable path.

Relay behavior in Sevcon mode:

- Uses TPDO3 `0x2220:00` throttle input voltage from bytes `2..3`.
- The throttle voltage scale is `0.00390625 V/bit`.
- Relay threshold defaults to raw `236`, equal to `0.921875 V`.
- The threshold is configurable with `CONFIG PEDAL=<V>` and stored in DA64 EEPROM.
- The physical footswitch relay is allowed only when:
  - All existing DA64 footswitch safety logic allows it.
  - TPDO3 throttle data is fresh.
  - `throttle_raw > configured_threshold_raw`.
- If the pedal voltage is stale or at/below the threshold, the physical footswitch relay is de-energized.

Debug with the DA64 serial command `MOTORREPORT`. It prints:

- `ready`: motor MCP2517 initialized.
- `tpdo3_age_ms` and `tpdo4_age_ms`: freshness of incoming Sevcon PDOs.
- `tpdo3_raw`: raw TPDO3 payload bytes.
- `throttle_raw` and `throttle_v`: TPDO3 throttle input used for the FS1 threshold.
- `throttle_le_raw`/`throttle_le_v` and `throttle_be_raw`/`throttle_be_v`: little-endian and big-endian interpretations of TPDO3 bytes `2..3`.
- `pedal_relay_threshold_raw` and `threshold_v`: configured relay threshold.
- `pedal_relay_voltage_allowed`, `throttle_fresh`, and `footswitch_relay`: relay gating state.
- `pedal_relay_block_mask`: bit 0 means TPDO3 throttle is stale, bit 1 means throttle is at/below the threshold.

Serial config command:

```text
CONFIG PEDAL=<V>
```

Example:

```text
CONFIG PEDAL=0.921875
OK CONFIG PEDAL=0.9219V raw=236
```

`CONFIG PEDAL_RAW=<raw>` is also available for service tooling that wants to write the exact TPDO3 raw threshold.

## Firmware Constants

DA64 expects these IDs and scales:

```cpp
MOTOR_CAN_BITRATE = 250000
MOTOR_CAN_TPDO3_ID = 0x455
MOTOR_CAN_TPDO4_ID = 0x368
MOTOR_CAN_SPEED_SCALE_KMH_PER_BIT = 0.0625
MOTOR_CAN_THROTTLE_V_SCALE_V_PER_BIT = 0.00390625
MOTOR_CAN_CAP_V_SCALE_V_PER_BIT = 0.0625
MOTOR_CAN_BATT_I_SCALE_A_PER_BIT = 0.0625
MOTOR_CAN_PEDAL_RELAY_ON_ABOVE_RAW_DEFAULT = 236
```

If Sevcon COB-IDs or mappings are changed, update both this document and the DA64 firmware constants/parsing logic.

## Sevcon Fault / EMCY Integration

Sevcon Gen4 faults can be integrated through CANopen EMCY frames on the motor CAN bus.

Observed example:

```text
0x0081  11312558  EMCY  Generic Error. Sevcon code: 0x4681
0x00000081  0.0ms  emcy  0x00 0x00 0x00 0x81 0x46 0x00 0x00 0x98
```

Interpretation:

| Field | Meaning |
|---|---|
| `0x0081` | CANopen EMCY COB-ID for node `1` (`0x80 + node_id`). |
| `EMCY` | CANopen emergency message. |
| `Generic Error` | CANopen emergency category decoded by the bus tool. |
| `0x4681` | Sevcon manufacturer fault code. This is the value Buddy PC should map to text/details. |

CANopen EMCY payload layout:

| Byte | Example | Meaning |
|---:|---:|---|
| 0..1 | `00 00` | CANopen emergency error code, little-endian. |
| 2 | `00` | CANopen error register. |
| 3..4 | `81 46` | Sevcon/BorgWarner fault ID, little-endian: `0x4681`. |
| 5..7 | `00 00 98` | Sevcon manufacturer-specific debug/status bytes. Preserve for PC diagnostics. |

For Buddy PC decoding:

```text
sevcon_fault_id = emcy.data[3] | (emcy.data[4] << 8)
```

The current fault table entry for `0x4681` is:

| Field | Value |
|---|---|
| Sevcon fault ID | `0x4681` |
| Fault group | CANopen |
| Severity | Warning |
| DTC | `P0A78-93` |
| Summary | Unit in pre-operational |
| Description | Controller is in pre-operational state |
| Recommended action | If configured and ready for use, change state to operational. If controller is slave, check master state. |

Recommended integration:

| Layer | Responsibility |
|---|---|
| DA64 firmware | Listen for motor CAN EMCY ID `0x081`, capture the raw CANopen EMCY payload and decoded Sevcon fault code, set DA64 fault code `40` only when ignition is on and charger is disconnected, and forward compact raw fault data on vehicle CAN `0x012`. |
| Buddy PC software | Decode Sevcon code `0x4681` and other Gen4 Size 6 fault IDs using a generated lookup table from `docs/Fault_codes_sevcon.xhtml`. |

Do not store the full Sevcon fault-code text table in DA64 firmware. Keep DA64 as the raw fault forwarder and keep the human-readable database in Buddy PC software.

DA64 publishes the cached Sevcon EMCY as:

| Vehicle CAN | Rate | Meaning |
|---:|---:|---|
| `0x010 byte 7 bits 0..6` | 4 Hz | Sevcon traction drive state from TPDO4 byte `5`. |
| `0x010 byte 7 bit 7` | 4 Hz | Quick flag: Sevcon EMCY active as a display/vehicle fault. Suppressed while charging or ignition off. |
| `0x012` | 4 Hz | Compact Sevcon EMCY bridge. Byte `0 bit 0` is suppressed while charging or ignition off, but cached raw fault bytes may remain for diagnostics. |
| Fault code `40` | 2 Hz/event | `DA64 SEVCON EMCY FAULT`, severity `3`. Only active when ignition is on and charger is disconnected. Event detail bytes are Sevcon fault low/high. |

`0x012` layout:

| Byte | Meaning |
|---:|---|
| 0 | Bit 0: active Sevcon EMCY display/vehicle fault. Suppressed while charging or ignition off. |
| 1 | Sevcon CANopen node ID. |
| 2..3 | CANopen emergency error code, little-endian. |
| 4 | CANopen error register. |
| 5..6 | Sevcon/BorgWarner fault ID, little-endian. |
| 7 | Raw EMCY byte 7, manufacturer/status byte. |

The DA64 serial command `FAULTS` also prints `sevcon_emcy_active`, `sevcon_emcy_node`, `sevcon_emcy_canopen`, `sevcon_emcy_error_register`, `sevcon_emcy_fault`, `sevcon_emcy_age_ms`, and `sevcon_emcy_raw`.

For node IDs other than `1`, the EMCY COB-ID is:

```text
0x080 + node_id
```

## Gen4 Size 2/4/6 Fault Code Lookup

This table is generated from `docs/Fault_codes_sevcon.xhtml` and includes the rows marked for Gen4 Size 2/4/6. Buddy PC should use `0x012 byte 5..6` as the lookup key.

| Sevcon fault ID | Level | LED | SmartView | CANopen error | DTC | Module | Type | Message | Cause | Recommended action |
|---:|---:|---:|---|---|---|---|---|---|---|---|
| 0x4481 | 1 | 2 | F12001 | 0x1000 | P05E3-62 | TracApp | Warning | Handbrake Fault (warn) | Handbrake is active when a direction selected | Warning to release the handbrake before driving |
| 0x4542 | 1 | 5 | F15002 | 0x9000 |  | IO | Warning | Low Oil | Oil level is low | Check oil level |
| 0x4543 | 1 | 5 | F15003 | 0x9000 |  | IO | Warning | Hydraulic Filter | Hydraulic Filter | Check hydraulic filter |
| 0x4544 | 1 | 5 | F15004 | 0x2300 |  | PumpApp | Warning | Pump Current Low | Pump motor is not drawing sufficient current | Check pump motor wiring and operation. Ensure an internal cutback is not stopping operation. |
| 0x4545 | 1 | 5 | F15005 | 0x1000 | P0AA6-00 | HW Protection | Warning | Isolation Fault | Isolation failure detected between logic and power frame | Check isolation between low and high voltage circuits on controller and vehicle systems |
| 0x4547 | 1 | 5 | F15007 | 0x9000 |  | TracApp | Warning | Tow Mode Active | Tow mode has been activated | Disable tow mode if not required |
| 0x4548 | 1 | 5 | F15008 | 0x9000 | C0051-29 | TracApp | Warning | Steer Sensor warning | Invalid steer sensor state | Check steer sensor wiring and configuration |
| 0x454C | 1 | 5 | F15012 | 0x9000 | P0A7F-7B | BattApp | Warning | Electrolyte Low Level | Battery Electrolyte low level detected | Check battery electrolyte level |
| 0x454D | 1 | 5 | F15013 | 0x9000 | P0A7F-7B | BattApp | Warning | Electrolyte Cutout Level | Battery Electrolyte low level detected and cutback | Check battery electrolyte level |
| 0x458A | 1 | 6 | F16010 | 0x1000 | B00B0-00 | TracApp | Warning | Seat (warning) | Seat switch open so configured regen settings in 0x2928 are being applied | Close the seat switch and recycle the selected direction |
| 0x458B | 1 | 6 | F16011 | 0x1000 | P057A-64 | TracApp | Warning | Footbrake (warning) | Customer specific fault. Raised when analog footbrake voltage does not match digital footbrake switch state | Check footbrake wiring and installation |
| 0x45C1 | 1 | 7 | F17001 | 0x3000 | P0A7D-00 | BattApp | Warning | BDI (battery discharge) warning | BDI remaining charge (0x2790,1) is less than configured BDI Warning level (0x2C30,5). No action is taken by the controller. For information only. | Charge battery (or if commissioning controller and battery is charged check BDI configuration in 0x2C30) |
| 0x45C2 | 1 | 7 | F17002 | 0x3000 | P0A7D-00 | BattApp | Warning | BDI (battery discharge) cutout | BDI remaining charge (0x2790,1) is less than configured BDI Cutout level (0x2C30,4). The low BDI drivability profile is applied if configured in 0x2931. | Charge battery (or if commissioning controller and battery is charged check BDI configuration in 0x2C30) |
| 0x45C3 | 1 | 7 | F17003 | 0x3000 | P0AF8-A2 | BattApp | Warning | Low Battery cutout | Battery voltage is less than the configured Under Voltage limit (0x2C02,2) for longer than the protection delay (0x2C03,0) | Charge battery (or if commissioning controller and battery is charged check BattApp configuration in 0x2C01, 0x2C02, 0x2C03) |
| 0x45C4 | 1 | 7 | F17004 | 0x3000 | P0AF8-A3 | BattApp | Warning | High Battery cutout | Battery voltage is greater than the configured Over Voltage limit (0x2C01,2) for longer than the protection delay (0x2C03,0) | Charge battery (or if commissioning controller and battery is charged check BattApp configuration in 0x2C01, 0x2C02, 0x2C03) |
| 0x45C5 | 1 | 7 | F17005 | 0x3000 | P0A78-17 | BattApp | Warning | High Capacitor cutout | Capacitor voltage is greater than the configured Over Voltage limit (0x2C01,2) for longer than the protection delay (0x2C03,0) | Charge battery (or if commissioning controller and battery is charged check BattApp configuration in 0x2C01, 0x2C02, 0x2C03) |
| 0x45C6 | 1 | 7 | F17006 | 0x3000 | P0AF8-A2 | HW Protection | Warning | Vbat below rated min | Battery voltage is less than rated minimum voltage for controller for longer than 1sec | Charge battery or check DC link voltage is within controller operating range (NOTE: This fault is sometimes seen at power down) |
| 0x45C7 | 1 | 7 | F17007 | 0x3000 | P0AF8-A3 | HW Protection | Warning | Vbat above rated max | Battery voltage is greater than rated maximum voltage for controller for longer than 1sec | Charge battery or check DC link voltage is within controller operating range |
| 0x45C8 | 1 | 7 | F17008 | 0x3000 | P0A78-A3 | HW Protection | Warning | Vcap above rated max | Capacitor voltage is greater than rated maximum voltage for controller for longer than 1sec | Charge battery or check DC link voltage is within controller operating range |
| 0x45C9 | 1 | 7 | F17009 | 0x3000 | P0A78-16 | MotorCtrl | Warning | Vcap cutback for motoring torque | Capacitor voltage is in low voltage cutback region configured in 0x4612 for motor control / torque conditioner drive voltage cutback map | Charge battery and DC link voltage is within configured range (if commissioning controller check drive torque cutback map 0x4612 is correct) |
| 0x45CA | 1 | 7 | F17010 | 0x3000 | P0A78-17 | MotorCtrl | Warning | Vcap cutback for regen torque | Capacitor voltage is in high voltage cutback region configured in 0x4619 (or 0x4612 if 0x4619 not present) for motor control / torque conditioner regen voltage cutback map | Charge battery and DC link voltage is within configured range (if commissioning controller check regen torque cutback map 0x4619 is correct, use 0x4612 if 0x4619 not present) |
| 0x45CB | 1 | 7 | F17011 | 0x3100 |  | HW Protection | Warning | Mains Under Voltage | Mains voltage is below minimum | Check mains supply is within controller limits |
| 0x45CC | 1 | 7 | F17012 | 0x3100 |  | HW Protection | Warning | Mains Over Voltage | Mains voltage is above maximum | Check mains supply is within controller limits |
| 0x4601 | 1 | 8 | F18001 | 0x4200 | P0A78-9D | HW Protection | Warning | Device too cold | Low heatsink temperature has reduced available power to motor | Allow controller to warm up to normal operating temperature |
| 0x4602 | 1 | 8 | F18002 | 0x4200 | P0A78-98 | HW Protection | Warning | Device too hot | High heatsink / junction / PCB track / capacitor temperature has reduced available power to motor | Allow controller to cool down to normal operating temperature. Check coolant flow rate / air flow is sufficient for required motor current |
| 0x4603 | 1 | 8 | F18003 | 0x4000 | P0A90-98 | Motor Thermistor | Warning | Motor in thermal cutback | High measured (0x4600,3) or estimated (0x4602,8) motor temperature has reduced available power to motor as configured in 0x4620, 0x4621, 0x461E, 0x461F | Allow motor to cool down to normal operating temperature. Check motor cooling or motor thermistor configuration and wiring. If not using a thermistor and this fault is set make sure motor estimate is configured correctly (or disabled in 0x4621,1 = 0 and 0x4621,2 = 100) |
| 0x4604 | 1 | 8 | F18004 | 0x4000 | P0A90-9D | Motor Thermistor | Warning | Motor too cold | Motor measured temperature is below -30degC | Check motor thermistor connection or allow motor to warm up. Check motor thermistor configuration and wiring |
| 0x4681 | 1 | 10 | F10101 | 0x1000 | P0A78-93 | CANopen | Warning | Unit in pre-operational | Controller is in pre-operational state | If configured and ready for use, change state to operational. If controller is slave check master state |
| 0x4682 | 1 | 10 | F10102 | 0x8100 | U01B7-87 | CANopen | Warning | IO can't initialise | Controller has not received all configured RPDOs at power up | Check PDOs on all other CANbus nodes are configured correctly and match up with controller RPDO configuration. Check CAN wiring. |
| 0x4683 | 1 | 10 | F10103 | 0x8250 | U0259-87 | CANopen | Warning | RPDO Timeout (warning) | One or more configured RPDOs not received with 3sec at start up or 500ms during normal operation. | Check status of all nodes on CANbus. Check PDOs on all CANbus nodes are configured correctly and match up with controller RPDO configuration |
| 0x46C1 | 1 | 11 | F11101 | 0x6300 | P0A3F-99 | Encoder | Warning | Encoder Alignment Warning | Encoder is not aligned properly with motor | Ensure encoder offset is configured correctly set or repeat encoder alignment procedure |
| 0x4741 | 1 | 13 | F13101 | 0x6100 | P0607-42 | SW Internal | Warning | Scheduler stack overflow warning | This warning is set if any task has between 10% and 20% of its stack free. | Internal software warning. Contact BorgWarner. |
| 0x4781 | 1 | 14 | F14101 | 0x1000 | U0001-09 | CANopen | Warning | CANopen anon EMCY level 1 | EMCY message received from non-BorgWarner node and anonymous EMCY level (0x2830,0) is set to 1. | Check status of non-BorgWarner nodes on CANbus |
| 0x47C1 | 1 | 15 | F15101 | 0x1000 |  | VehApp | Warning | Vehicle Service Required | Vehicle service next due time (0x2850,5) has expired. If supported Service driveability profile (0x2925) will activate. | Service vehicle and reset service hours counter. Or check service configuation if service indication not required. |
| 0x47C7 | 1 | 15 | F15107 | 0x9000 |  | VehApp | Warning | Pump oil level low | Pump oil level is too low | Add oil to the hydraulic pump |
| 0x47C8 | 1 | 15 | F15108 | 0x9000 |  | VehApp | Warning | Pump oil temperature | Pump temperature is too high | Check for blockage in oil filter screen and/or oil cooler |
| 0x4881 | 2 | 2 | F22001 | 0x1000 | B00B0-63 | TracApp | Drive Inhibit | Seat Fault | Valid direction selected with operator not seated or operator is not seated for a user configurable time in drive. | Must be seated with switches inactive |
| 0x4882 | 2 | 2 | F22002 | 0x1000 | P2E00-24 | TracApp | Drive Inhibit | Two Direction Fault | Both the forward and reverse switches have been active simultaneously for greater than 200 ms. | If both switches are selected then release both, then select required direction. If fault persists check vehicle wiring and configuration of mapping for digital inputs. |
| 0x4883 | 2 | 2 | F22003 | 0x1000 | P0510-62 | TracApp | Drive Inhibit | SRO Fault | FS1 (foot switch / throttle) active for user configurable delay (0x2914,2) without a direction selected. | Deselect FS1 (foot switch / throttle) and select direction first |
| 0x4884 | 2 | 2 | F22004 | 0x1000 | P05D0-67 | TracApp | Drive Inhibit | Sequence Fault | Any drive switch active at power up as configured in 0x2918 | Deselect all drive switches and then reselect required switch. |
| 0x4885 | 2 | 2 | F22005 | 0x1000 | P0510-67 | TracApp | Drive Inhibit | FS1 Recycle Fault | FS1 (foot switch / throttle) active after a direction change and FS1 recycle function enabled (0x2914,1,bit1) | Deselect FS1 / throttle pedal |
| 0x4886 | 2 | 2 | F22006 | 0x1000 | P05D4-64 | TracApp | Drive Inhibit | Inch Fault | Inch switch active along with any drive switch active (excluding inch switches), seat switch indicating operator present or handbrake switch active. | Correct conditions need to be met for inching functionality (drive not selected, operator not present and handbrake released) |
| 0x4887 | 2 | 2 | F22007 | 0x9000 |  | TracApp | Drive Inhibit | Overload Fault | Vehicle overloaded | Remove overload condition |
| 0x4888 | 2 | 2 | F22008 | 0x9000 |  | TracApp | Drive Inhibit | Raised and Tilted Fault | Scissor lift platform raised and tilted | Lower platform |
| 0x4889 | 2 | 2 | F22009 | 0x9000 |  | TracApp | Drive Inhibit | Pothole Fault | Scissor lift pothole protection active | Lower platform to allow moving vehicle out of pot hole. |
| 0x488A | 2 | 2 | F22010 | 0x9000 |  | TracApp | Drive Inhibit | Traction Inhibit Fault | Traction function inhibited using traction inhibit switch (0x2137) | Deselect traction inhibit switch if vehicle drive is required |
| 0x488B | 2 | 2 | F22011 | 0x9000 |  | TracApp | Drive Inhibit | Illegal Mode Change Fault | Vehicle changed from traction mode to pump mode (or vice versa) when direction selected | Deselect all drive switches |
| 0x488C | 2 | 2 | F22012 | 0x9000 |  | TracApp | Drive Inhibit | Tilt Sensor Fault | Scissor lift error code (0x3802,0) set to 0x02 | Check tilt sensor |
| 0x488F | 2 | 2 | F22015 | 0x1000 |  | MotorCtrl | Drive Inhibit | Sensorless Startup | Sensorless has failed to start up. | Check motor parameters are correct for motor |
| 0x4942 | 2 | 5 | F25002 | 0x9000 |  | HW Internal | Drive Inhibit | PST Fault | An issue has occurred with the PST unit | Check PST unit faults |
| 0x4943 | 2 | 5 | F25003 | 0x3200 |  | HW Internal | Drive Inhibit | PFC Fault | Fault with the PFC stage | Replace unit if fault persists |
| 0x4944 | 2 | 5 | F25004 | 0x3200 |  | HW Internal | Drive Inhibit | Boost Fault | Fault with the Boost Stage | Replace unit if fault persists |
| 0x4981 | 2 | 6 | F26001 | 0x1000 | P0120-1C | TracApp | Drive Inhibit | Throttle Fault | Throttle value (0x2620,0) is greater than 20% at power up. | Release throttle and re-apply. Check wiring |
| 0x4982 | 2 | 6 | F26002 | 0x9000 | P2887-18 | TracApp | Drive Inhibit | E-Brake Wire off | Wire-off detected in electrobrake circuit | Check electrobrake wiring, ensure current flows when energised |
| 0x4983 | 2 | 6 | F26003 | 0x1000 | P2E00-9A | TracApp | Drive Inhibit | Direction Change | Direction is changed and vehicle speed is greated than allowed in 0x2929,2 | Neutral switch should be selected to clear the fault |
| 0x4A81 | 2 | 10 | F20101 | 0x8250 | U025A-87 | CANopen | Drive Inhibit | RPDO Timeout (drive inhibit) | One or more configured RPDOs not received with 3s at start up or 500ms during normal operation. | Check status of all nodes on CANbus. Check PDOs on all CANbus nodes are configured correctly and match up. |
| 0x4B01 | 2 | 12 | F22101 | 0x8140 | U0001-88 | CAN | Drive Inhibit | CAN bus-off (drive inhibit) | CANbus off fault condition detected on multinode system. NOTE: normally this is a Very Severe CAN off fault | Check CANbus wiring and configuration (no duplicate COB-IDs, correct baud rate). If controller is standalone check configuration in 0x5901,0 |
| 0x4B81 | 2 | 14 | F24101 | 0x1000 | U0001-09 | CANopen | Drive Inhibit | CANopen anon EMCY level 2 | EMCY message received from non-BorgWarner node and anonymous EMCY level (0x2830,0) is set to 2. | Check status of non-BorgWarner nodes on CANbus |
| 0x4C41 | 3 | 1 | F31001 | 0x6300 | P0607-56 | CANopen | Severe | Too many slaves | Number of slaves (0x2810,0) set higher than maximum allowed number of slaves (typically 7) | Check 0x2810,0 setting is correct |
| 0x4DC3 | 3 | 7 | F37003 | 0x3000 | P030A-16 | HW Protection | Severe | Power Supply (keyswitch) Critical | Battery voltage has dropped below critical level | Check controller voltage supply |
| 0x4E81 | 3 | 10 | F30101 | 0x8250 | U025B-87 | CANopen | Severe | RPDO Timeout (severe) | One or more configured RPDOs not received with 3s at start up or 500ms during normal operation. | Check status of all nodes on CANbus. Check PDOs on all CANbus nodes are configured correctly and match up. |
| 0x4F01 | 3 | 12 | F32101 | 0x8200 | U0001-68 | CANopen | Severe | CANopen unexpected slave state | CANopen slave has changed to unexpected state | Check status of all nodes on CANbus. |
| 0x4F02 | 3 | 12 | F32102 | 0x8200 | U0001-08 | CANopen | Severe | CANopen EMCY send failed | Unable to transmit EMCY message | Contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F41 | 3 | 13 | F33101 | 0x6100 | P0607-04 | SW Internal | Severe | Internal SW Fault | Internal software fault - typically set in combination with another fault | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F42 | 3 | 13 | F33102 | 0x6100 | P0607-42 | SW Internal | Severe | Out of memory | Out of memory - internal software fault | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F43 | 3 | 13 | F33103 | 0x6100 | P0607-04 | SW Internal | Severe | General DSP error | Unknown error raised by motor model code - internal software fault | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F44 | 3 | 13 | F33104 | 0x6100 | P0607-42 | SW Internal | Severe | Timer Error | Unable to allocate timer - internal software fault | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F45 | 3 | 13 | F33105 | 0x6100 | P0607-42 | SW Internal | Severe | Queue Error | Unable to post message to queue - internal software fault | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F46 | 3 | 13 | F33106 | 0x6100 | P0607-42 | SW Internal | Severe | Scheduler Error | Unable to create task in scheduler - internal software fault | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F48 | 3 | 13 | F33108 | 0x6100 | P0607-04 | SW Internal | Severe | I/O Internal SW Error | Input Output subsytem fault - internal software fault or configuration error | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F49 | 3 | 13 | F33109 | 0x6100 | P0607-04 | SW Internal | Severe | GIO Internal SW Error | Generic IO subsystem fault - internal software fault or configuration error | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F4C | 3 | 13 | F33112 | 0x6100 | P0607-04 | SW Internal | Severe | OBD Internal SW Error | Object Dictionary subsystem fault - internal software fault or parameter range error fault | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F4D | 3 | 13 | F33113 | 0x6100 | P0607-04 | SW Internal | Severe | VehApp Internal SW Error | Vehicle Application subsystem fault - internal software fault or configuration error | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F4E | 3 | 13 | F33114 | 0x6100 | P0607-04 | SW Internal | Severe | DMC Internal SW Error | Drives and Motion Control subsystem fault - internal software fault or configuration error | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F4F | 3 | 13 | F33115 | 0x6100 | P0607-04 | SW Internal | Severe | TracApp Internal SW Error | Traction Application subsystem fault - internal software fault or configuration error | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F53 | 3 | 13 | F33119 | 0x6100 | P0607-04 | SW Internal | Severe | App Manager Internal SW Error | Application Manager subsystem fault - internal software fault or configuration error | If other faults are set, resolve the conditions for these faults. Check configuration is valid and recycle keyswitch. If this is the only fault set contact BorgWarner with the fault data bytes, controller report and dcf configuration file |
| 0x4F54 | 3 | 13 | F33120 | 0x2200 | P0A51-28 | HW Protection | Severe | Autozero Range Error | Current sensor auto-zero value outside of allowed range | Internal hardware fault or check configuration in 0x4654,0 (if available) |
| 0x4F55 | 3 | 13 | F33121 | 0x6300 | P0607-55 | Configuration | Severe | DSP motor parameter error | Internal software fault or configuration object is out of range: Also may indicate issue with IPM 3D tables | Set configuration object to valid value. Out of range object can be identified using 0x5621 or DVT CLI window (if enabled) |
| 0x4F56 | 3 | 13 | F33122 | 0x1000 | P0A90-62 | TracApp | Severe | Motor in wrong direction | Motor rotation detected as wrong direction according to TracApp | Check motor wiring and configuration |
| 0x4F57 | 3 | 13 | F33123 | 0x1000 | P0A90-36 | TracApp | Severe | Motor stalled | Motor rotation stalled according to TracApp | Check motor wiring and configuration |
| 0x4F81 | 3 | 14 | F34101 | 0x1000 | U0001-09 | CANopen | Severe | CANopen anon EMCY level 3 | EMCY message received from non-BorgWarner node and anonymous EMCY level (0x2830,0) is set to 3. | Check status of non-BorgWarner nodes on CANbus |
| 0x5041 | 4 | 1 | F41001 | 0x6300 | P0607-44 | HW Internal | Very Severe | Bad NVM Data | EEPROM or flash configuration data corrupted and data can not be recovered. | If firmware has recently been updated, revert to previous version. Contact BorgWarner for support. |
| 0x5042 | 4 | 1 | F41002 | 0x6300 | P0607-56 | Configuration | Very Severe | VPDO Out of Range | VPDO mapped to non-existent or invalid object | Check all VPDO mappings (0x3000 to 0x3400) |
| 0x5043 | 4 | 1 | F41003 | 0x6300 | P0607-56 | Configuration | Very Severe | Static Range Error | At least one configuration object is out of range | Set configuration object to valid value. Our of range object can be identified using 0x5621 or DVT CLI window. |
| 0x5044 | 4 | 1 | F41004 | 0x6300 | P0607-56 | Configuration | Very Severe | Dynamic Range Error | At least one configuration object is out of dynamic range. This is where one objects range depends on another object or hardware variant specific level. | Check all dynamic range objects. DVT CLI window indicates type of object which is out of range. |
| 0x5045 | 4 | 1 | F41005 | 0x6300 | P0607-64 | Configuration | Very Severe | Auto-configuration Fault | Unable to automatically configure I/O and vehicle setup. | Check auto configuration objects (0x5810 and 0x5811) |
| 0x5081 | 4 | 2 | F42001 | 0x9000 | C0052-29 | TracApp | Very Severe | Invalid Steer Switches | Steering switches are in an invalid state | Check steering switches and wiring. Check steering map 0x2913 is valid |
| 0x5101 | 4 | 4 | F44001 | 0x9000 | P0AA0-72 | BattApp | Very Severe | Line Contactor o/c | Line contactor open circuit - contactor did not close when the coil is energized. | Check line contactor operation and wiring. DC link fuse blown |
| 0x5102 | 4 | 4 | F44002 | 0x9000 | P0AA0-73 | BattApp | Very Severe | Line Contactor welded | Line contactor appears to be closed when the coil is NOT energized. | Check line contactor hasn't welded closed and the wiring is correct. Make sure the line contactor is not mapped if there is not one connected and controlled by the motor inverter |
| 0x5103 | 4 | 4 | F44003 | 0x5000 | P0AA0-01 | BattApp | Very Severe | Contactor Drive Fault | Fault with contactor drive | Check contactor and wiring for short circuits. If fault persists replace the controller |
| 0x5181 | 4 | 6 | F46001 | 0x9000 | P0853-13 | IO | Very Severe | Digital Input Wire Off | Digital input wire-off | Check wiring and configuration |
| 0x5182 | 4 | 6 | F46002 | 0x9000 | P0853-1C | IO | Very Severe | Analog Input Wire Off | Analog input voltage outside of configured range (0x46C0 - 0x46C7) | Check wiring and configuration is correct in 0x46c0 to 0x46c7. If analog input is not used the range should be set to the minimum and maximum limits |
| 0x5183 | 4 | 6 | F46003 | 0x2300 | P2C92-19 | IO | Very Severe | Analog Output Over Current | Contactor driver over current protection has activated | Ensure contactor doesn't exceed maximum current and check contactor wiring for short circuits. Make sure contactor or load is NOT capacitive such as contactors with economizer circuits |
| 0x5184 | 4 | 6 | F46004 | 0x1000 | P2C92-9E | IO | Very Severe | Analog Output On with No Failsafe | Internal hardware failsafe circuitry not working | Contact driver failed startup checks. Internal circuit has failed or external wiring issues with contactor connected to B- / logic ground. For Gen4 DC the contactor output used needs to be configured in 0x46A5, if none are used a resistor needs to be fitted in place of a contactor for the test to complete - see Gen4DC manual / AppNote |
| 0x5185 | 4 | 6 | F46005 | 0x1000 | P2C92-9F | IO | Very Severe | Analog Output Off with Failsafe | Contactor driver circuit has failed start up checks | Contact driver couldn't activate for startup checks. Internal circuit has failed or external coil short circuit or wiring issue connecting contactor driver to keyswitch |
| 0x5186 | 4 | 6 | F46006 | 0x4200 | P2C92-4B | IO | Very Severe | Analog Output Over Temperature | Contactor driver over temperature | Ensure contactor doesn't exceed maximum current and check contactor wiring |
| 0x5187 | 4 | 6 | F46007 | 0x2300 | P2C92-18 | IO | Very Severe | Analog Output Under Current | Contactor driver unable to achieve current target in current mode | Ensure contactor driver current target is within range for available voltage |
| 0x5188 | 4 | 6 | F46008 | 0x2300 | P2C92-12 | IO | Very Severe | Analog Output Short Circuit | Contactor driver MOSFET short circuit detected | Check contactor and wiring for short circuits. If fault persists replace the controller |
| 0x51C2 | 4 | 7 | F47002 | 0x3200 | P0C78-00 | HW Internal | Very Severe | Capacitor Precharge Failure | Capacitor voltage (0x5100,3) did not rise above 5V at power up | Check battery connection for reverse polarity, or check internal / external short circuit across the DC link.Potential failure of pre-charge circuit or relay. |
| 0x5201 | 4 | 8 | F48001 | 0x4200 | P0A78-98 | HW Protection | Very Severe | Heatsink / device overtemp | Controller heatsink (or junctions, capacitors, PCB) has reached critical high temperature, and the controller has shut down. | Allow controller to cool down to normal operating temperature. If liquid cooled check coolant flow. |
| 0x52C1 | 4 | 11 | F41101 | 0x9000 | P0A3F-02 | Encoder | Very Severe | Encoder Fault | Encoder input wire-off is detected. | Check encoder wiring - especially shielding and routing of encoder cables. Check configuration is valid and peak / trough correct for sincos |
| 0x52C2 | 4 | 11 | F41102 | 0x2300 | P0A51-1D | HW Protection | Very Severe | Motor Overcurrent Fault | Motor current exceeded controller rated maximum | Check motor configuration and wiring. Typically caused by current control gains or motor parameters incorrect. |
| 0x52C3 | 4 | 11 | F41103 | 0x2300 | P0A78-06 | MotorCtrl | Very Severe | Current Control Fault | Motor controller unable to maintain control of motor currents | Check motor configuration and wiring. Typically caused by current control gains or motor parameters incorrect. For PMAC check encoder alignment |
| 0x52C4 | 4 | 11 | F41104 | 0x1000 | P0A3F-37 | Encoder | Very Severe | Motor Overspeed Fault | Motor speed has exceeded overspeed threshold in 0x4624 | Check speed limit gains and configuration. Ensure motor speed is not too high for configuration. |
| 0x52C5 | 4 | 11 | F41105 | 0x6300 | P0A3F-99 | Encoder | Very Severe | Encoder Alignment Severe | Encoder is not aligned properly. | Ensure encoder offset and sequence is correct |
| 0x5301 | 4 | 12 | F42101 | 0x8100 | U0001-89 | CAN | Very Severe | CAN bus Fault | CANbus fault condition detected. | Check CANbus wiring and configuration |
| 0x5302 | 4 | 12 | F42102 | 0x8200 | U0001-08 | CANopen | Very Severe | CANopen Bootup not received | CANopen slave has not transmitted boot up message at power up | Check status of all nodes on CANbus and multinode configuration. |
| 0x5303 | 4 | 12 | F42103 | 0x8110 | U0001-89 | CAN | Very Severe | CAN Lo-Pri Rx queue overrun | Low priority CAN receive queue overflow. | Check CANbus wiring and configuration. Ensure transmitting node is sending at correct rate |
| 0x5304 | 4 | 12 | F42104 | 0x8110 | U0001-89 | CAN | Very Severe | CAN Lo-Pri Tx queue overrun | Low priority CAN transmit queue overflow. | Check CANbus wiring and configuration. Ensure messages are being acknowledged and bus load is not too high |
| 0x5305 | 4 | 12 | F42105 | 0x8110 | U0001-89 | CAN | Very Severe | CAN Hi-Pri Rx queue overrun | High priority CAN receive queue overflow. | Check CANbus wiring and configuration. Ensure transmitting node is sending at correct rate |
| 0x5306 | 4 | 12 | F42106 | 0x8110 | U0001-89 | CAN | Very Severe | CAN Hi-Pri Tx queue overrun | High priority CAN transmit queue overflow. | Check CANbus wiring and configuration. Ensure messages are being acknowledged and bus load is not too high |
| 0x5309 | 4 | 12 | F42109 | 0x8130 | U0001-08 | CANopen | Very Severe | Nodeguarding Failed | Not used | Not used |
| 0x530C | 4 | 12 | F42112 | 0x8200 | U0001-68 | CANopen | Very Severe | CANopen device in wrong state | CANopen slave has changed to unexpected state (typically failed to go operational) | Check status of all nodes on CANbus. Configuration issues such as multiple masters on bus, duplicate node ID or heartbeat COBIDs can trigger the fault. |
| 0x530E | 4 | 12 | F42114 | 0x8200 | U0001-08 | CANopen | Very Severe | CANopen SDO Internal Error | Internal CANopen fault | Check SDO client and server COBID are set correctly |
| 0x530F | 4 | 12 | F42115 | 0x8200 | U0001-08 | CANopen | Very Severe | CANopen SDO Timeout Error | Internal CANopen fault | Check SDO client and server COBID are set correctly |
| 0x5319 | 4 | 12 | F42125 | 0x8200 | P0A78-67 | VehApp | Very Severe | Motor slave in wrong state | Motor control slave has not responded as expected to controlword sent by TracApp or PumpApp. This can either be the same controller for single node systems or a slave node connected over CAN | Typically caused when the motor controller can't enable due to being unable to complete power on checks due to spinning motors or insufficient DC link voltage. Bridge enable timeout is also known to trigger this fault. |
| 0x5343 | 4 | 13 | F43103 | 0x6100 | P0607-42 | SW Internal | Very Severe | Fault List Overflow | Attempting to set too many faults. | Internal software fault. Ensure correct firmware and settings are loaded. Logic supply is above critical level. |
| 0x5345 | 4 | 13 | F43105 | 0x6100 | P0607-42 | SW Internal | Very Severe | Scheduler stack Overflow | Less than 10% of the stack is free on one of the RTOS tasks. | Internal software fault. Check configuration and contact BorgWarner |
| 0x5381 | 4 | 14 | F44101 | 0x1000 | U0001-09 | CANopen | Very Severe | CANopen anon EMCY level 4 | EMCY message received from non-BorgWarner node and anonymous EMCY level (0x2830,0) is set to 4. | Check status of non-BorgWarner nodes on CANbus |
| 0x5389 | 4 | 14 | F44109 | 0x4000 | P0A90-98 | Motor Thermistor | Very Severe | Motor Over temperature | Motor temperature critical | Allow to cool to normal operating temperature and check motor cooling. Also check thermistor configuration and wiring |
| 0x538A | 4 | 14 | F44110 | 0x4200 | P0A78-9D | HW Protection | Very Severe | Device under temperatrue | Device temperature reached critical low temperature | Heat the controler or with the logic enabled wait for it to warm to the required minimum temperature |
| 0x5441 | 5 | 1 | F51001 | 0x5000 | P0607-57 | HW Internal | Return to Base | Incompatible hardware version | Detected controller hardware version incompatible with software | Check correct software is programmed into controller. Reprogram if necessary. Contact BorgWarner |
| 0x5442 | 5 | 1 | F51002 | 0x5000 | P0607-54 | HW Internal | Return to Base | Calibration Fault | Calibration settings in controller are out of range or not set | Controller current sensors require calibration. Contact BorgWarner |
| 0x54C1 | 5 | 3 | F53001 | 0x3000 | P0A78-A3 | HW Protection | Return to Base | PowerFrame Overvoltage Fault | DC link voltage exceeds rated maximum for controller | Check battery condition and wiring. Ensure battery voltage is not too high for the controller operational rating |
| 0x54C2 | 5 | 3 | F53002 | 0x2300 | P0A78-49 | HW Protection | Return to Base | PowerFrame Fault | Current flow exceeds controller maximum. Caused by motor current too high, configuration error causing overcurrent, wiring fault/short or internal controller failure | Check motor control configuration and wiring. If fault is set persistently, disconnect motor cables to see if fault clears |
| 0x54C3 | 5 | 3 | F53003 | 0x5000 | P0A78-12 | HW Protection | Return to Base | PowerFrame s/c M1 upper | MOSFET / IGBT short circuit detection on M1 top devices. Fault condition is checked when powerframe disabled by comparing M1 terminal voltage to B+ | Check motor wiring. Check controller condition with motor disconnected |
| 0x54C4 | 5 | 3 | F53004 | 0x5000 | P0A78-11 | HW Protection | Return to Base | PowerFrame s/c M1 lower | MOSFET / IGBT short circuit detection on M1 bottom devices. Fault condition is checked when powerframe disabled by comparing M1 terminal voltage to B- | Check motor wiring. Check controller condition with motor disconnected |
| 0x54C5 | 5 | 3 | F53005 | 0x5000 | P0A78-12 | HW Protection | Return to Base | PowerFrame s/c M2 upper | MOSFET / IGBT short circuit detection on M2 top devices. Fault condition is checked when powerframe disabled by comparing M2 terminal voltage to B+ | Check motor wiring. Check controller condition with motor disconnected |
| 0x54C6 | 5 | 3 | F53006 | 0x5000 | P0A78-11 | HW Protection | Return to Base | PowerFrame s/c M2 lower | MOSFET / IGBT short circuit detection on M2 bottom devices. Fault condition is checked when powerframe disabled by comparing M2 terminal voltage to B- | Check motor wiring. Check controller condition with motor disconnected |
| 0x54C7 | 5 | 3 | F53007 | 0x5000 | P0A78-12 | HW Protection | Return to Base | PowerFrame s/c M3 upper | MOSFET / IGBT short circuit detection on M3 top devices. Fault condition is checked when powerframe disabled by comparing M3 terminal voltage to B+ | Check motor wiring. Check controller condition with motor disconnected |
| 0x54C8 | 5 | 3 | F53008 | 0x5000 | P0A78-11 | HW Protection | Return to Base | PowerFrame s/c M3 lower | MOSFET / IGBT short circuit detection on M3 bottom devices. Fault condition is checked when powerframe disabled by comparing M3 terminal voltage to B- | Check motor wiring. Check controller condition with motor disconnected |
| 0x54C9 | 5 | 3 | F53009 | 0x5000 | P0A78-67 | HW Protection | Return to Base | PowerFrame s/c checks incomplete | Unable to complete MOSFET / IGBT short circuit tests at power up | Ensure pre-charge has increased DC link voltage high enough for tests to be carried out. |
| 0x5781 | 5 | 14 | F54101 | 0x1000 | U0001-09 | CANopen | Return to Base | CANopen anon EMCY level 5 | EMCY message received from non-BorgWarner node and anonymous EMCY level (0x2830,0) is set to 5. | Check status of non-BorgWarner nodes on CANbus |

## Validation Checklist

After programming Sevcon TPDOs:

1. Confirm motor CAN bus is `250 kbps`.
2. Confirm TPDO3 transmits on `0x455` and TPDO4 transmits on `0x368`.
3. Confirm both frames are standard 11-bit IDs, not extended IDs.
4. Confirm TPDO3 has DLC 8 and TPDO4 has DLC 7.
5. Confirm TPDO3 byte `0..1` changes with motor temperature.
6. Confirm TPDO3 byte `6..7` changes with vehicle speed.
7. Confirm TPDO4 byte `0..1` follows capacitor voltage.
8. Confirm TPDO4 byte `2` follows heatsink temperature.
9. Confirm TPDO4 byte `3..4` follows motor-controller battery current.
10. Confirm TPDO4 byte `5` follows traction drive state.
11. Confirm TPDO4 byte `6` low nibble follows footbrake, forward, reverse, and footswitch inputs.
12. Confirm Sevcon EMCY faults appear on `0x081` for node `1`.
