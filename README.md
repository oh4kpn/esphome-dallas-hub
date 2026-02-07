# ESPHome Dallas 1-Wire Hub (Event-based)

This repository contains an **ESPHome configuration optimized for large Dallas DS18B20 temperature sensor networks**
(e.g. 10+ sensors across one or multiple 1‑Wire buses).

Instead of creating one Home Assistant entity per sensor (which consumes memory and slows down boot),
this hub **reads all sensors and sends raw measurements to Home Assistant as events**
using the `esphome.dallas_raw` event.

On the Home Assistant side, **trigger-based template sensors** receive and store the data.
This enables precise timestamps, low ESP32 memory usage, and a scalable architecture.

---

## ✨ Features

- **Event-based architecture**  
  No heavy Home Assistant entities stored in ESP memory.

- **Accurate UTC timestamps**  
  Each measurement includes `measured_at` (ISO 8601 UTC) generated at measurement time.

- **SNTP synchronization status**  
  Events include a `time_synced` flag to validate clock reliability.

- **Decimal precision control**  
  Values are rounded to two decimals (`%.2f`) at the source.

- **Modular & scalable design**  
  Uses ESPHome `script` and `substitutions` for easy expansion.

---

## 🛠 Hardware

- ESP32 (e.g. ESP32 DevKit V1)
- DS18B20 temperature sensors
- 4.7kΩ pull-up resistor on the data line(s)

All DS18B20 sensors share a 1‑Wire bus connected to a GPIO pin with the pull‑up resistor to 3.3 V.

---

## ⚙️ ESPHome Configuration

The hub uses a centralized script to package and send measurements to Home Assistant.

### 1. Substitutions & Time

Force UTC time to keep Home Assistant handling clean and predictable.

```yaml
substitutions:
  temp_update_interval: "30s"

time:
  - platform: sntp
    id: sntp_time
    timezone: "UTC"
```

---

### 2. Event Dispatch Script

```yaml
script:
  - id: send_dallas_event
    mode: queued
    parameters:
      sensor_address: string
      sensor_value: float
    then:
      - homeassistant.event:
          event: esphome.dallas_raw
          data:
            address: !lambda "return sensor_address;"
            value: !lambda 'return str_sprintf("%.2f", sensor_value);'
            hub: dallas-hub-1
            time_synced: !lambda 'return id(sntp_time).now().is_valid() ? "true" : "false";'
            measured_at: !lambda 'return id(sntp_time).now().strftime("%Y-%m-%dT%H:%M:%SZ");'
      - delay: 50ms
```

---

### 3. Dallas Sensor Example

```yaml
sensor:
  - platform: dallas_temp
    one_wire_id: ow_bus_1
    index: 0
    id: b1_i0
    name: "Bus 1 Sensor 0"
    update_interval: ${temp_update_interval}
    filters:
      - filter_out: nan
    on_value:
      then:
        - if:
            condition:
              lambda: 'return !std::isnan(x);'
            then:
              - script.execute:
                  id: send_dallas_event
                  sensor_address: !lambda "return id(b1_i0).get_address_name();"
                  sensor_value: !lambda "return x;"
```

---

## 🏠 Home Assistant Configuration

Use trigger-based template sensors in `configuration.yaml` or `templates.yaml`.

### Raw Sensor (Hardware Address)

```yaml
template:
  - trigger:
      - platform: event
        event_type: esphome.dallas_raw
        event_data:
          address: "0x9c000000847d4828"
    sensor:
      - name: "Dallas Sensor 0"
        unique_id: "dallas_0x9c000000847d4828"
        state: "{{ '%.2f' | format(trigger.event.data.value | float) }}"
        unit_of_measurement: "°C"
        device_class: temperature
        state_class: measurement
        attributes:
          address: "{{ trigger.event.data.address }}"
          hub: "{{ trigger.event.data.hub }}"
          time_synced: "{{ trigger.event.data.time_synced == 'true' }}"
          measured_at: "{{ as_datetime(trigger.event.data.measured_at) }}"
          last_event_received: "{{ now() }}"
```

---

### Logical Abstraction Layer:

```yaml
sensor:
  - name: "Heat Pump Supply Water"
    unit_of_measurement: "°C"
    state: "{{ states('sensor.dallas_0x66012211281e8a28') }}"
```

This allows sensor hardware replacement without changing automations.

---

## 📊 Included Files

- `esphome.yaml` – ESPHome hub configuration
- `dallas_listener_in_ha.yaml` – Home Assistant template listeners
- `ha_debug_card.yaml` – Lovelace debug card
- `ha_debug_card.png` – Lovelace debug card UI preview

---

## 🔍 Troubleshooting

**Clock not synced?**  
Check `time_synced` attribute.  
If false, `measured_at` may be invalid (e.g. 1970).

**No updates in HA?**
1. Developer Tools → Events
2. Listen to `esphome.dallas_raw`
3. Verify ESP logs, Wi‑Fi, and sensor address match

---

## 📄 License

MIT
