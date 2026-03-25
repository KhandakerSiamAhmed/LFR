# Competition-Grade Line Following Robot (LFR) - ESP32-S3

A highly advanced, fully autonomous Line Following Robot built for competition. It utilizes an **ESP32-S3** microcontroller, leveraging its dual-core architecture and 8MB PSRAM via **FreeRTOS** to achieve ultra-low latency sensor processing, dynamic PID control, and advanced maze solving.

## 🚀 Key Features

- **Dual-Core FreeRTOS Architecture**:
  - **Core 1 (Flight Controller)**: Hard real-time control loop processing 14-channel analog IR reads, PID error calculations, and motor PWM.
  - **Core 0 (Background Tasks)**: Non-blocking I2C OLED display rendering, rotary encoder UI processing, WebSockets, and Sonar interrupts.
- **Advanced Motion Control**: 
  - Dynamic PID gain scheduling based on track curvature.
  - S-Curve / Trapezoidal motion profiling to prevent traction loss.
- **Auto-Tuning Mathematics**: Employs Relay Feedback mathematical derivation (Ziegler-Nichols) utilizing an **MPU6050** gyroscope to self-tune the PID constants `Kp`, `Ki`, and `Kd` dynamically.
- **Maze Solving**: Left-Hand Method (LSRB) with U-Turn substitution algorithm storing infinite node arrays dynamically in the 8MB PSRAM.
- **Trash Collection (Task)**: Integrated Sonar (HC-SR04+) mapping with an internal lift/grab payload mechanism via MG90 Servos for multi-node trash pickup and dumping.
- **Comprehensive UI & Telemetry**: 
  - Onboard 1.3" OLED Display for sensor viewing, PID editing, and servo calibration.
  - Live WebSockets Telemetry dashboard plotting PID errors, battery voltage, and gyro data to a laptop/phone in real-time. (Note: Network features are strictly disabled by default to comply with competition rules, and can only be activated via the onboard Setup UI).
- **Persistent Memory**: Saves all thresholds, constants, and calibration data automatically to EEPROM. The robot indefinitely remembers its last calibration setup until a new sequence is requested, functioning right out of the box after power cycles.
- **Strict Rulebook Compliance**: Incorporates automated physical payload inspection-retraction sequences and an algorithmic End Box halt detection sequence to guarantee max competition scores.

## 🛠️ Hardware Stack

* **Microcontroller**: ESP32-S3 N16R8 Data Board
* **Motors & Drives**: 2x 25GA 500 RPM DC Geared Motors + 2x IBT2 Motor Drivers
* **Sensors**: 
  * Analog 14 IR Array (Boomerang shape) with a 16-Channel Analog Multiplexer (CD74HC4067)
  * MPU6050 6-Axis Gyroscope/Accelerometer
  * HC-SR04+ Ultrasonic Sensor (3.3V)
* **Power**: 3S 11.1V LiPo Battery with a dedicated DC-DC 3A Buck Step-down converter for a stable 3.3V logic ecosystem.

## 📂 Documentation

The complete breakdown of the mathematical models, hardware circuitry, safety optimizations, and algorithm constraints can be found in the documentation folder:

* 📖 **[Comprehensive Project Guide](Documentation/LFR_Complete_Guide.md)** - Master document covering algorithms, PID theory, hardware setup, and UI routing.
* 🔌 **[ESP32-S3 Pinout Rules](Documentation/ESP32_S3_Pinout_Rules.md)** - Detailed reasoning for the strict pinout architecture to avoid Boot, Memory, and Wi-Fi strapping conflicts.

## 💡 System Architecture Note

To prevent hardware collisions and blocking states, the entire 3.3V ecosystem avoids logic level scaling. The Sonar triggers are handled purely via hardware interrupts on Core 0 to prevent blocking the aggressive Core 1 PID loop. Ensure you review the wiring schema in the main guide before assembly!