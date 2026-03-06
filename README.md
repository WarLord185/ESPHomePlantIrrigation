# ESPHomePlantIrrigation

> Automated plant irrigation system built on the NanoESP32-C6, managed through ESPHome and Home Assistant.

---

## Overview

ESPHomePlantIrrigation is an embedded systems project that delivers automated, schedule- and sensor-driven irrigation control using the **NanoESP32-C6** microcontroller. The firmware is defined as an ESPHome YAML configuration with custom `lambda` C++ components, and integrates natively with **Home Assistant** for monitoring, automation, and manual override.

---

## Hardware Requirements

| Component | Details |
|---|---|
| Microcontroller | NanoESP32-C6 |
| Connectivity | Wi-Fi 6 (802.11ax), Bluetooth 5 LE (on-chip) |
| Power Supply | 3.3 V / 5 V (board-dependent) |
| Actuators | Solenoid valve(s) or water pump relay module |
| Sensors (optional) | Soil moisture sensor, DHT22 / AHT10 for temperature & humidity |

> **Note:** Specific wiring diagrams and GPIO pin mappings are documented in [`docs/wiring.md`](docs/wiring.md).

---

## Installation

### Prerequisites

Ensure the following are installed and configured before proceeding:

- [Home Assistant](https://www.home-assistant.io/installation/) (2024.1 or later recommended)
- [ESPHome Add-on](https://esphome.io/guides/getting_started_hassio.html) installed in Home Assistant
- Python 3.10+ and `esphome` CLI (optional, for local compilation)

```bash
pip install esphome
```

### 1. Clone the Repository

```bash
git clone https://github.com/<WarLord185>/ESPHomePlantIrrigation.git
cd ESPHomePlantIrrigation
```

### 2. Configure Secrets

Copy the secrets template and populate it with your credentials:

```bash
cp secrets.yaml.example secrets.yaml
```

Edit `secrets.yaml`:

```yaml
wifi_ssid: "YourNetworkSSID"
wifi_password: "YourNetworkPassword"
api_encryption_key: "YourBase64EncryptionKey"
ota_password: "YourOTAPassword"
```

> **Important:** `secrets.yaml` is listed in `.gitignore` and must never be committed to version control.

### 3. Flash the Device

#### Via ESPHome Dashboard (recommended)

1. Open the ESPHome Dashboard in Home Assistant.
2. Click **+ New Device** and select your `irrigation.yaml` configuration.
3. Connect the NanoESP32-C6 via USB and click **Install**.

#### Via ESPHome CLI

```bash
esphome run irrigation.yaml
```

Subsequent updates can be deployed over-the-air (OTA) once the device is on the network:

```bash
esphome run irrigation.yaml --device <device-ip>
```

---

## Usage & Examples

### Basic Irrigation Schedule

The configuration below defines a valve that opens for 10 minutes every morning at 07:00:

```yaml
switch:
  - platform: gpio
    pin: GPIO5
    name: "Irrigation Valve Zone 1"
    id: valve_zone_1

time:
  - platform: homeassistant
    on_time:
      - cron: "0 0 7 * * *"
        then:
          - switch.turn_on: valve_zone_1
          - delay: 10min
          - switch.turn_off: valve_zone_1
```

### Soil Moisture-Triggered Irrigation (Lambda Example)

The following lambda reads an analog soil moisture sensor and activates the valve only when the soil is dry:

```yaml
sensor:
  - platform: adc
    pin: GPIO2
    name: "Soil Moisture"
    id: soil_moisture
    update_interval: 60s
    on_value:
      then:
        - lambda: |-
            if (x < 30.0) {  // threshold in percent
              id(valve_zone_1).turn_on();
            } else {
              id(valve_zone_1).turn_off();
            }
```

### Home Assistant Automation

Once the device is adopted, entities are automatically exposed. A simple HA automation to send a notification when irrigation starts:

```yaml
automation:
  - alias: "Notify on Irrigation Start"
    trigger:
      - platform: state
        entity_id: switch.irrigation_valve_zone_1
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          message: "Irrigation Zone 1 has started."
```

---

## Project Structure

```
ESPHomePlantIrrigation/
├── irrigation.yaml          # Main ESPHome configuration
├── secrets.yaml.example     # Secrets template
├── docs/
│   └── wiring.md            # GPIO pinout and wiring diagrams
└── README.md
```

---

## License

This project is licensed under the [MIT License](LICENSE).

---

## Contributing

Contributions are welcome. Please open an issue to discuss proposed changes before submitting a pull request. Ensure any new YAML or lambda code is tested on hardware prior to submission.

---

## Acknowledgements

- [ESPHome](https://esphome.io/) — firmware framework
- [Home Assistant](https://www.home-assistant.io/) — automation platform
- [Espressif Systems](https://www.espressif.com/) — NanoESP32-C6 hardware
