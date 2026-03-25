# LFR Project Guidelines

## Environment & Code Style
- Use C++ within a PlatformIO or ESP-IDF environment.
- Code should be organized into modular files under `src/` and `include/` directories.
- Ensure all modules include `config.h` to access global pin definitions and tuning constants.

## Architecture (ESP32-S3 FreeRTOS)
- **Core 1 (APP_CPU) - "Flight Controller"**: Hard real-time tasks. Run the fast PID loop, rapid MUX switching, sensor reading, and motor PWM. **Rules**: NO `delay()`, NO `Serial.print`, NO `Wire` (I2C) calls on this core.
- **Core 0 (PRO_CPU) - Interface & Overhead**: Non-blocking background tasks. Handle OLED displays, rotary encoders, EEPROM/NVS saving, and battery voltage reading.
- **Inter-Process Communication (IPC)**: Pass telemetry between Core 1 and Core 0 exclusively using FreeRTOS Queues or Mutex-protected global variables to prevent race conditions.
- **Task Wrapping**: Wrap functional modules inside `xTaskCreatePinnedToCore()`.

## Memory & Hardware Conventions
- **NVS over EEPROM**: Use the `#include <Preferences.h>` library instead of legacy EEPROM for robust data saving.
- **PSRAM Allocation**: The system uses an ESP32-S3 N16R8 variant. Explicitly allocate large arrays (like maze history) in 8MB PSRAM using `ps_malloc(size)`. Always check if the returned pointer is `!= NULL`.
- **Logic Level**: Assume all inputs and outputs are 3.3V safe; do not implement software voltage offsets.

## Pin Selection Safety
- **Analog Sensors**: Always place critical analog inputs (e.g., Multiplexer signal, Battery Voltage monitor) on **ADC1** pins (GPIO 1-10) since ADC2 can be locked by Wi-Fi/Bluetooth.
- **Avoid Reserved Pins**: Do not use Boot/Strapping pins (0, 3, 45, 46), Internal Flash/PSRAM pins (26-38), or default Serial/USB pins (19, 20, 43, 44).

## Documentation References
- See `Documentation/Code_Implementation_Plan.md` for algorithm details and the master software blueprint.
- See `Documentation/ESP32_S3_Pinout_Rules.md` for detailed rules on pin selections.