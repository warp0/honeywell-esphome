# honeywell-esphome

ESPHome integration notes for the Honeywell HTRAM air quality monitor.

## Known CO2 packet

The built-in CO2 sensor reports frames in the following format:

```text
FE 04 02 XX XX CRC CRC
```

`XX XX` is the CO2 value in ppm encoded as a big-endian 16-bit integer. The
sample below treats the trailing bytes as a Modbus RTU CRC16 stored in
little-endian order and validates them before publishing a reading.

## ESPHome bootstrap configuration

The example below reads 7-byte UART frames from the monitor and publishes the
decoded CO2 value to Home Assistant through ESPHome. The same configuration is
available in `esphome/honeywell_htram.yaml`.

```yaml
esphome:
  name: honeywell_htram
  platform: ESP32
  board: esp32dev

wifi:
  ssid: "YOUR_WIFI"
  password: "YOUR_PASS"

logger:

api:

ota:

uart:
  id: uart_bus
  rx_pin: GPIO16
  baud_rate: 9600

globals:
  - id: co2_value
    type: int
    restore_value: no
    initial_value: "0"

sensor:
  - platform: template
    id: co2_sensor
    name: "CO2 Level"
    unit_of_measurement: "ppm"
    accuracy_decimals: 0
    update_interval: never
    lambda: |-
      return id(co2_value);

interval:
  - interval: 1s
    then:
      - lambda: |-
          auto modbus_crc = [](const uint8_t *data, size_t len) {
            uint16_t crc = 0xFFFF;
            for (size_t i = 0; i < len; i++) {
              crc ^= data[i];
              for (size_t bit = 0; bit < 8; bit++) {
                if (crc & 0x0001) {
                  crc = (crc >> 1) ^ 0xA001;
                } else {
                  crc >>= 1;
                }
              }
            }
            return crc;
          };

          static uint8_t frame[7];
          static size_t frame_pos = 0;

          int pending_bytes = id(uart_bus).available();
          while (pending_bytes-- > 0) {
            uint8_t byte;
            if (!id(uart_bus).read_byte(&byte)) {
              continue;
            }

            if (frame_pos == 0 && byte != 0xFE) {
              continue;
            }

            if (frame_pos == 1 && byte != 0x04) {
              frame_pos = 0;
              if (byte == 0xFE) {
                frame[frame_pos++] = byte;
              }
              continue;
            }

            if (frame_pos == 2 && byte != 0x02) {
              frame_pos = 0;
              if (byte == 0xFE) {
                frame[frame_pos++] = byte;
              }
              continue;
            }

            frame[frame_pos++] = byte;

            if (frame_pos == sizeof(frame)) {
              frame_pos = 0;

              const uint16_t expected_crc = modbus_crc(frame, 5);
              const uint16_t reported_crc =
                  static_cast<uint16_t>(frame[5]) |
                  (static_cast<uint16_t>(frame[6]) << 8);

              if (expected_crc == reported_crc) {
                const int value = (frame[3] << 8) | frame[4];
                id(co2_value) = value;
                id(co2_sensor).publish_state(value);
              }
            }
          }
```

## Notes

- Connect the HTRAM UART TX line to the ESP32 RX pin configured above.
- The sample focuses on CO2 because that packet layout has been reverse
  engineered already.
- This repository is intended to collect practical integration notes for these
  inexpensive legacy Honeywell HTRAM monitors and provide a useful starting
  point for ESPHome-based firmware.