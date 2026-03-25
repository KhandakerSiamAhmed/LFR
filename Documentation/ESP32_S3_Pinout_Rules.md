# ESP32-S3 Pin Selection Rules (N16R8 Variant)

When assigning pins for the ESP32-S3 N16R8 (16MB Flash, 8MB PSRAM), it is critical to avoid system-reserved pins to ensure stability, successful booting, and correct peripheral operation.

## 1. Boot / Strapping Pins (DO NOT USE)
These pins dictate the boot mode of the ESP32-S3. If an external component pulls these high or low unexpectedly during startup, the microcontroller will fail to boot or enter the wrong mode.
* **GPIO 0**: Controls Boot Mode (HIGH = Normal Run, LOW = Download/Flash mode).
* **GPIO 3**: Strapping pin. JTAG interface.
* **GPIO 45**: Strapping pin. Dictates VDD_SPI voltage.
* **GPIO 46**: Strapping pin. Dictates log output.

## 2. Memory / Internal Flash Pins (DO NOT USE)
These pins are internally wired to the onboard Flash memory and PSRAM. Touching these will crash the MCU immediately.
* **GPIO 26 to GPIO 38**: Strictly reserved for internal octal SPI Flash and PSRAM. (Note: N16R8 uses Octal PSRAM which requires GPIO 38).

## 3. Serial / USB Pins (AVOID)
To retain the ability to flash code over USB-C and use the Serial Monitor without unplugging components, avoid these:
* **GPIO 19, 20**: Used for hardware USB (D-, D+).
* **GPIO 43, 44**: Used for hardware UART0 (TX/RX Serial Monitor).

## 4. Analog Inputs (ADC1 vs ADC2)
The ESP32-S3 has two Analog-to-Digital converters: ADC1 and ADC2.
* **ADC2 (GPIO 11-20)** is shared with the Wi-Fi/Bluetooth modem. If Wi-Fi is enabled, ADC2 becomes locked, and `analogRead()` returns errors. 
* **RULE**: Always place critical analog sensors (like your Multiplexer signal and Battery Voltage monitor) on **ADC1** pins.
* **ADC1 Pins**: GPIO 1 through GPIO 10.

## 5. Input-Only Pins (No Pull-ups/Pull-downs)
Unlike older ESP32 boards, the ESP32-S3 does not have the classic 34, 35, 36, 39 input-only constraint. Practically all general GPIOs on the S3 can be used for both I/O and have internal pull-ups/pull-downs.

## Summary Checklist for LFR Project
By following the rules above, the project safely allocated:
* **Analog Sensors**: Pins 4, 5 (ADC1 validated).
* **Motors & Data Selectors**: Pins 10-17 (Safe general I/O block avoiding strapping and USB).
* **Hardware Interrupts**: Encoder switch and signals allocated safely on remaining free pins.