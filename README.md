# ESPHome Dallas 1-Wire Hub (Event-based)

This project is an **ESPHome configuration optimized for large Dallas DS18B20 temperature sensor networks**
(e.g. 10–64+ sensors on a multiple 1-Wire bus).

Instead of creating a separate **Home Assistant entity for every sensor** (which consumes memory and slows down boot),
this hub reads all sensors and sends the **raw measurements to Home Assistant as events**
(`esphome.dallas_raw`).

On the Home Assistant side, **lightweight template sensors** pick up and store the data.
This approach enables **precise timestamps**, **low ESP32 memory usage**, and a **clean, scalable data flow**.

---

## ✨ Features

- **Event-based architecture**  
  No heavy Home Assistant entities stored in ESP memory.

- **Accurate UTC timestamps**  
  Every measurement includes a `measured_at` field (ISO 8601 UTC), generated at measurement time.

- **SNTP synchronization status**  
  Events include a `time_synced` flag to verify clock reliability.

- **Decimal precision control**  
  Values are rounded to two decimals (`%.2f`) at the source.

- **Modular & scalable design**  
  Uses ESPHome `script` and `substitutions`, making it easy to add more sensors.

---

## 🛠 Hardware

- ESP32 (e.g. ESP32 DevKit V1)
- DS18B20 temperature sensors
- 4.7kΩ pull-up resistors on the data lines

---

## ⚙️ ESPHome Configuration

This setup uses a centralized script to package and send data to Home Assistant.

---

### 1. Definitions (Substitutions & Time)

Force UTC time to keep Home Assistant handling clean and predictable.

```yaml
substitutions:
  temp_update_interval: "30s" # Global update interval

time:
  - platform: sntp
    id: sntp_time
    timezone: "UTC" # Force UTC, HA converts to local time
```

---

### 2. Script (Event dispatch logic)

This script packages the measurement and fires a Home Assistant event.

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
            hub: kellari-onewire-hub-1
            # Is SNTP time valid?
            time_synced: !lambda 'return id(sntp_time).now().is_valid() ? "true" : "false";'
            # Exact measurement time (UTC ISO 8601)
            measured_at: !lambda 'return id(sntp_time).now().strftime("%Y-%m-%dT%H:%M:%SZ");'
      - delay: 50ms # Small delay to avoid network congestion
```

---

### 3. Sensor definition

Each sensor calls the script above.  
Note the use of `id(...)` to fetch the sensor address.

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

Use **trigger-based template sensors** in `configuration.yaml` or `templates.yaml`.

---

### Example: Raw sensor (hardware ID based)

This sensor listens for `esphome.dallas_raw` events and updates only when the correct
64-bit address appears.

```yaml
template:
  - trigger:
      - platform: event
        event_type: esphome.dallas_raw
        event_data:
          address: "0x9c000000847d4828"
    sensor:
      - name: "Monster Test 1"
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

### Logical abstraction layer (Optional)

You can create a clean logical sensor that references the raw one.
This makes hardware replacement painless.

```yaml
sensor:
  - name: "Heat Pump Supply Water"
    unit_of_measurement: "°C"
    state: "{{ states('sensor.dallas_0x9c000000847d4828') }}"
```

---

## 🔍 Troubleshooting

### How do I know if the clock is synced?
Check the `time_synced` attribute.

- `false` → `measured_at` is unreliable (often year 1970)
- `true` → timestamps are valid

`last_event_received` always shows when Home Assistant received the data.

### Sensors not updating in Home Assistant?

1. Go to **Developer Tools → Events**
2. Listen to: `esphome.dallas_raw`
3. If no events appear:
   - Check ESP logs
   - Verify Wi-Fi connectivity
4. If events appear:
   - Make sure the `address` in the template sensor matches exactly

---

## 📄 License

MIT
