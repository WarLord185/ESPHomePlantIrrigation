# ESPHomePlantIrrigation — Troubleshooting Log

A record of all issues encountered and their solutions during development.


## 1. SDA/SCL Swapped

**Problem:** SHT40 failed to communicate entirely.

**Cause:** Colour-coded wires were physically connected to the wrong pins — SDA and SCL were swapped at the breadboard.

**Solution:** Swapped the physical wires to match the config (`sda: GPIO6`, `scl: GPIO7`).


## 2. SHT40 Communication Failed

**Error:**
```
[E][sht4x:078]: Communication failed
[W][sht4x:081]: Get serial number failed
[E][component:255]: sht4x.sensor is marked FAILED: unspecified
```

**Cause:** I2C bus frequency was too aggressive at the default 400 kHz for the SHT40 on the ESP32-C6 with esp-idf. Root cause was also compounded by issue 1 (swapped wires).

**Solution:** Added `frequency: 100000Hz` to the `i2c:` block and `precision: High` to the sensor config.

```yaml
i2c:
  sda: 6
  scl: 7
  frequency: 100000Hz

sensor:
  - platform: sht4x
    precision: High
```


## 3. DHT11 → SHT40 Migration

**Problem:** Original firmware used a DHT11 on a single GPIO data pin. The SHT40 uses I2C.

**Solution:** Replaced `platform: dht` / `pin` / `model: DHT11` with `platform: sht4x`, and added an `i2c:` bus block.

```yaml
# Before
sensor:
  - platform: dht
    pin: 13
    model: DHT11

# After
i2c:
  sda: 6
  scl: 7

sensor:
  - platform: sht4x
    precision: High
```


## 4. Circular Dependency (SSD1306 + SHT40 on esp-idf)

**Error:**
```
ERROR Circular dependency detected! Please run with -v option to see what functions failed to complete.
```

**Cause:** The SSD1306 display and SHT40 sensor both attempted to claim the I2C bus during the same setup phase, causing ESPHome's code generator to deadlock.

**Solution:** Added `setup_priority: -100` to the display so it initialises after the sensor. Also split buffer writes so the temperature and humidity `on_value` callbacks no longer cross-reference each other's sensor state.

```yaml
display:
  - platform: ssd1306_i2c
    setup_priority: -100
```


## 5. GPIO10 Has No ADC on ESP32-C6

**Error:**
```
sensor.adc: ESP32C6 doesn't support ADC on this pin.
pin: 10
```

**Cause:** ADC1 on the ESP32-C6 is only available on GPIO0–GPIO6. GPIO10 has no ADC capability.

**Solution:** Moved the capacitive soil moisture sensor from GPIO10 to **GPIO3**.

```yaml
sensor:
  - platform: adc
    pin: 3  # was 10
```


## 6. GPIO8 and GPIO9 Are Strapping Pins

**Warning:**
```
WARNING GPIO9 is a strapping PIN and should only be used for I/O with care.
WARNING GPIO8 is a strapping PIN and should only be used for I/O with care.
```

**Cause:** Strapping pins are sampled by the ESP32-C6 bootloader at power-on to determine boot mode. External components attached to these pins can cause unpredictable startup behaviour or boot failures.

**Solution:** Moved affected components to safe GPIO pins.

| Component | Was | Now |
|
| Pump indicator LED | GPIO9 | GPIO18 |
| Pump manual button (TTP223) | GPIO8 | GPIO19 |


## 7. `attenuation: 11db` Deprecated

**Warning:**
```
WARNING `attenuation: 11db` is deprecated, use `attenuation: 12db` instead
```

**Cause:** ESPHome 2026.x renamed the ADC attenuation option for the ESP32.

**Solution:** Updated the ADC sensor config.

```yaml
sensor:
  - platform: adc
    attenuation: 12db  # was 11db
```


## 8. Double Line on Soil Moisture Graph

**Problem:** After the 128-sample circular buffer wrapped around, a second spurious line appeared at the bottom of the graph on the OLED display.

**Cause:** Uninitialised buffer slots contained `0.0` by default. The graph skip condition `if (isnan(v) || v == 0)` was intended to skip these, but once the buffer contained a real reading of `0.0` (valid data), that slot was also being skipped, and the wraparound revealed the old zeroed slots plotting at the graph baseline — creating a second line.

**Solution:** Two changes applied together:

1. Buffers initialised to `NAN` at boot so unwritten slots are unambiguously empty.
2. Graph skip condition changed from `if (isnan(v) || v == 0)` to `if (isnan(v))` so zero is treated as valid data.

```cpp
// Before
if (isnan(v) || v == 0) continue;

// After
if (isnan(v)) continue;
```


## 9. `initial_value` Does Not Support Lambdas in Globals

**Error:**
```
This option is not templatable!
initial_value: !lambda |-
  float arr[128];
  for (int i = 0; i < 128; i++) arr[i] = NAN;
  return *arr;
```

**Cause:** ESPHome does not allow `!lambda` expressions for `initial_value` in the `globals:` block — it only accepts scalar literal values.

**Solution:** Moved the buffer initialisation loop into `esphome: on_boot:` with `priority: 600`, which runs early enough that all buffers are filled with `NAN` before any sensor callbacks fire.

```yaml
esphome:
  on_boot:
    priority: 600
    then:
      - lambda: |-
          for (int i = 0; i < 128; i++) {
            id(temp_buf)[i] = NAN;
            id(hum_buf)[i]  = NAN;
            id(soil_buf)[i] = NAN;
          }
```