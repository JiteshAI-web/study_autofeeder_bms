# study_autofeeder_bms

# AutoFeeder - Battery Management System (BMS)

**ESP32-based battery management firmware for AutoFeeder automated feeding system.**

## Overview

This firmware manages a 12V lead-acid battery charging system with SMPS backup, monitors battery health, and publishes telemetry via MQTT. It provides intelligent charging control, safety protection, and integration with the main AutoFeeder controller.

## Hardware

- **MCU**: ESP32 (Arduino framework)
- **Power System**:
  - 24V SMPS (AC mains backup)
  - 12V Lead-Acid Battery
  - Relay for charge control
- **Sensors**:
  - Voltage Sensor 1 (GPIO2) - SMPS line monitoring
  - Voltage Sensor 2 (GPIO3) - Battery voltage
  - ACS712-5A (GPIO4) - Charge current sensor (via voltage divider)
- **Control**:
  - Relay (GPIO5) - SMPS-to-battery connection

## Features

### Smart Charging Logic
- **Start Charging**: Battery ≤ 12.2V and AC present (>18V)
- **Stop Charging**: Battery ≥ 13.79V AND current < 0.4A (proper lead-acid termination)
- **Hard-Stop Failsafe**: Battery ≥ 13.87V (any current) — stops charging regardless of current sensor, protects against ACS712 failure
- **Safety Timeout**: 8-hour maximum charge time
- **AC Loss Protection**: Stops charging if SMPS drops below 15V (4-second debounce to avoid false trips from brief SMPS dips)

### Battery Protection
- **Critical Shutdown**: ≤10.5V - Emergency stop all operations
- **Resume Operations**: ≥11.0V - Restart after recovery
- **Deep Discharge Prevention**: Sends shutdown command to main controller

### MQTT Integration
- **Real-time Telemetry**: Publishes battery voltage, SMPS voltage, charge current every 2 seconds
- **State Reporting**: Battery states (idle/charging/full/discharged)
- **Event Notifications**: AC loss, charging start/stop, critical battery warnings
- **Command Integration**: Sends STOP/SHUTDOWN/RESUME commands to AutoFeeder main controller

## MQTT Topics

### Published Topics

| Topic | Payload | Description |
|-------|---------|-------------|
| `bms/DEVICE_ID/online` | `online`/`offline` | Connection status |
| `bms/DEVICE_ID/v_smps` | `23.45` | SMPS voltage (V) |
| `bms/DEVICE_ID/v_batt` | `12.67` | Battery voltage (V) |
| `bms/DEVICE_ID/current` | `1.234` | Charge current (A) |
| `bms/DEVICE_ID/ac` | `ON`/`OFF` | AC mains status |
| `bms/DEVICE_ID/battery_state` | `idle`/`charging`/`full`/`low`/`critical` | Battery state |
| `bms/DEVICE_ID/event` | Text | Event messages |
| `bms/DEVICE_ID/event` | `charge_stopped_voltage_hard_stop` | Hard-stop triggered (ACS712 failsafe) |

### Command Topics (Sent to Main Controller)

| Topic | Payload | When Sent |
|-------|---------|-----------|
| `feeder/DEVICE_ID/cmd` | `STOP` | Battery < 11.0V (low) |
| `feeder/DEVICE_ID/cmd` | `SHUTDOWN` | Battery ≤ 10.5V (critical) |
| `feeder/DEVICE_ID/cmd` | `RESUME` | Battery > 11.0V (recovered) |

## Configuration

### WiFi & MQTT Settings

Edit `BMS_MQTT.ino` and set **two defines** that must both match the paired AutoFeeder device ID:

```cpp
// ── CHANGE BOTH OF THESE to match your AutoFeeder device ID ──────────────────
#define DEVICE_ID         "BFL_FdtryA001"   // BMS identity — used in bms/DEVICE_ID/... topics
#define TARGET_STEPPER_ID "BFL_FdtryA001"   // Paired AutoFeeder ID — used in feeder/TARGET_STEPPER_ID/cmd
//                         ↑ MUST BE IDENTICAL ↑
// ─────────────────────────────────────────────────────────────────────────────
```

> **Important:** `DEVICE_ID` and `TARGET_STEPPER_ID` must always be the same value and must match the `DEVICE_ID` configured in the paired AutoFeeder Raspberry Pi firmware. If they differ, SHUTDOWN/STOP/RESUME commands will be sent to the wrong device and battery protection will not work.

Also set your network credentials:

```cpp
const char* WIFI_SSID = "YourWiFiSSID";    // WiFi network name
const char* WIFI_PASS = "YourPassword";     // WiFi password
const char* MQTT_BROKER = "mqttbroker.bc-pl.com";  // MQTT broker address
const char* MQTT_USER = "mqttuser";         // MQTT username
const char* MQTT_PASSWD = "YourMQTTPass";   // MQTT password
```

### Voltage Divider Calibration

Adjust resistor values to match your hardware:

```cpp
// SMPS Sensor (example: 22kΩ + 1kΩ)
const float R1_SMPS = 22000.0f;
const float R2_SMPS = 1000.0f;
const float CAL_SMPS = 0.998;  // Fine-tune with multimeter

// Battery Sensor (example: 10kΩ + 1.5kΩ)
const float R1_BATT = 10000.0f;
const float R2_BATT = 1500.0f;
const float CAL_BATT = 0.998;  // Fine-tune with multimeter
```

### Current Sensor Calibration

For ACS712-5A with voltage divider:

```cpp
const float ZERO_MV = 599.0f;  // Measured voltage at 0A (calibrate with AC off)
const float CURRENT_CAL_FACTOR = 34.0f;  // Adjust to match multimeter reading
```

### Battery Thresholds

Customize for your battery type:

```cpp
const float BATT_LOW_V = 12.2f;       // Start charging (25% SoC)
const float BATT_FULL_V = 13.79f;     // Stop charging — primary termination (current-based)
const float BATT_HARD_STOP_V = 13.87f; // Stop charging — hard-stop failsafe (voltage only, ignores current sensor)
const float BATT_CRITICAL_V = 10.5f;  // Emergency shutdown
const float BATT_RESUME_V = 11.0f;    // Resume operations
```

## Installation

### Hardware Setup

1. **Power Connections**:
   - Connect 24V SMPS output to voltage divider → GPIO2
   - Connect 12V battery to voltage divider → GPIO3
   - Connect ACS712 output (via voltage divider) → GPIO4
   - Connect relay control → GPIO5

2. **Wiring Diagram**:
   ```
   24V SMPS ----[R1=22kΩ]----+----[R2=1kΩ]---- GND
                             |
                           GPIO2

   12V Battery -[R1=10kΩ]---+----[R2=1.5kΩ]-- GND
                             |
                           GPIO3

   ACS712 Out --[1kΩ]-------+----[2kΩ]------- GND
                             |
                           GPIO4

   Relay Signal ─────────────────────────────── GPIO5
   ```

3. **Safety**: Use proper voltage dividers to bring all sensor inputs to 0-3.3V range for ESP32.

### Software Setup

1. **Install Arduino IDE**:
   - Download from [arduino.cc](https://www.arduino.cc/en/software)
   - Install ESP32 board support via Board Manager

2. **Install Libraries**:

   - `WiFi.h` / `WiFiClient.h` — **built into the ESP32 board package** (no separate install needed; installed automatically when you add ESP32 board support in step 1)
   - `PubSubClient` — **must be installed manually**:
     1. Open `Sketch → Include Library → Manage Libraries`
     2. Search for **PubSubClient**
     3. Install **PubSubClient by Nick O'Leary**

3. **Upload Firmware**:
   - Open `BMS_MQTT.ino` in Arduino IDE
   - Configure WiFi/MQTT settings
   - Select board: `ESP32 Dev Module`
   - Select COM port
   - Click Upload

4. **Verify Operation**:
   - Open Serial Monitor (115200 baud)
   - Check WiFi connection
   - Verify MQTT publishing with `mosquitto_sub -h broker -t "bms/DEVICE_ID/#"`

## Operation

### Normal Charging Cycle

1. **Idle**: Battery at 12.5V, AC present, no charging (battery healthy)
2. **Low Battery**: Battery drops to 12.2V
3. **Start Charging**: Relay ON, current flows into battery
4. **Charging**: Battery voltage rises, current gradually decreases
5. **Full Detection**: Battery reaches 13.79V AND current < 0.4A for 60 seconds
6. **Stop Charging**: Relay OFF, charging complete
7. **Monitor**: System continues monitoring and repeats if needed

### Emergency Protection

If battery drops to **10.5V** (critical):
- Immediately sends `SHUTDOWN` command to main AutoFeeder controller
- Publishes critical battery event to MQTT
- Waits for battery to recover above 11.0V before resuming

### AC Loss Handling

If AC power fails (SMPS < 15V):
- Immediately turns OFF relay (no charging)
- Publishes AC loss event
- System runs on battery until AC returns

## Monitoring

### Serial Console

```
WiFi connected: 192.168.1.100
MQTT connected
Publishing metrics...
SMPS: 23.45V | Battery: 12.67V | Current: 1.23A | Charging: ON
AC State: ON | Battery State: charging
```

### MQTT Dashboard

Subscribe to topics for real-time monitoring:

```bash
# All BMS topics
mosquitto_sub -h mqttbroker.bc-pl.com -u mqttuser -P password -t "bms/DEVICE_ID/#"

# Battery state only
mosquitto_sub -h mqttbroker.bc-pl.com -u mqttuser -P password -t "bms/DEVICE_ID/battery_state"

# Events only
mosquitto_sub -h mqttbroker.bc-pl.com -u mqttuser -P password -t "bms/DEVICE_ID/event"
```

## Troubleshooting

### Incorrect Voltage Readings

1. Measure actual voltage with multimeter
2. Adjust `CAL_SMPS` or `CAL_BATT` calibration factors
3. Formula: `CAL_FACTOR = Actual_Voltage / Measured_Voltage`

### Incorrect Current Readings

1. Turn off AC power (no current flowing)
2. Measure actual voltage at ACS712 output with multimeter
3. Update `ZERO_MV` with measured value (e.g., 599mV)
4. Apply known load and adjust `CURRENT_CAL_FACTOR` to match multimeter

### MQTT Connection Issues

1. Check WiFi credentials
2. Verify MQTT broker is reachable: `ping mqttbroker.bc-pl.com`
3. Test MQTT credentials with mosquitto_pub
4. Check firewall/port 1883

### Charging Not Starting

1. Verify AC present (SMPS > 18V)
2. Check battery voltage < 12.2V
3. Verify relay wiring
4. Check `RELAY_ACTIVE_HIGH` setting matches your relay module

## Related Repositories

This BMS firmware is a sub-component of the AutoFeeder ecosystem. The main controller runs on Raspberry Pi and communicates with this ESP32 BMS over MQTT.

- **[auto-feeder-rpi](https://github.com/Control-System-Bfl/auto-feeder-rpi)** - Main AutoFeeder controller (Raspberry Pi) — integrates with this BMS
- **[AutoFeeder-Firmware-BMS](https://github.com/Control-System-Bfl/AutoFeeder-Firmware-BMS)** - This repository (ESP32 BMS firmware)

## Specifications

- **Input Voltage**: 24V SMPS, 12V Battery
- **Charge Current**: Up to 5A (ACS712-5A limit)
- **Battery Type**: 12V Lead-Acid (SLA/AGM)
- **Communication**: WiFi + MQTT
- **Telemetry Rate**: 2 seconds
- **Safety**: 8-hour timeout, deep discharge protection, AC loss detection, hard-stop failsafe at 13.87V

## License

Part of AutoFeeder automated feeding system.

---

**Last Updated**: March 3, 2026
**Platform**: ESP32 (Arduino Framework)
**Communication**: MQTT over WiFi
