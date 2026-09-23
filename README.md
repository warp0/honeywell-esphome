# honeywell-esphome

ESPHome integration notes for the Honeywell HTRAM air quality monitor.

## Known CO2 packet

The built-in CO2 sensor reports frames in the following format:

```text
FE 04 02 XX XX CRC CRC
```

`XX XX` is the CO2 value in ppm encoded as a big-endian 16-bit integer. The
bootstrap configuration treats the trailing bytes as a Modbus RTU CRC16 stored in
little-endian order and validates them before publishing a reading.

## ESPHome bootstrap configuration

Use `esphome/honeywell_htram.yaml` as the bootstrap configuration for an ESP32
that is connected to the HTRAM UART output. The example:

- listens on `GPIO16` at `9600` baud,
- resynchronizes on the `FE 04 02` frame header,
- validates the trailing Modbus RTU CRC16 bytes, and
- publishes the decoded CO2 value to Home Assistant as `CO2 Level`.

## Notes

- Connect the HTRAM UART TX line to the ESP32 RX pin configured above.
- The sample focuses on CO2 because that packet layout has been reverse
  engineered already.
- This repository is intended to collect practical integration notes for these
  inexpensive legacy Honeywell HTRAM monitors and provide a useful starting
  point for ESPHome-based firmware.