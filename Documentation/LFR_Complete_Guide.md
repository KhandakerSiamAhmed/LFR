# Competition-Grade Line Following Robot (LFR) - ESP32-S3

This document serves as the master guide for the LFR project, merging rules, component lists, architectures, algorithms, and code.

---

## 1. Rulebook Constraints & Track Characteristics

### Robot Constraints
* **Max Dimensions**: 25cm(L) × 25cm(W) × 15cm(H)
* **Max Weight**: 1kg (16V DC max voltage restriction)
* **Rules**: Fully autonomous, onboard power, physical kill switch required to cut main power.

### Arena & Lines
* **Area**: 15ft x 10ft.
* **Line**: 30mm width, Black on white (or white on black).
* **Intersections**: Min distance between nodes is 25cm, and curve nodes/angles are at least 45 degrees.

### Tasks (Trash Collection - Round 2)
* **Payload**: 5x5x5 cm cubes (Max 15g).
* **Rule**: Stored *internally* inside/top of the robot (no dragging). Must pick up from 5-10 spots and dump entirely in a 30x30x5cm target zone.

---

## 2. Hardware List & Purchasing

### ✅ Currently Owned
1. ESP32-S3 N16R8 Data Board
2. IBT2 Motor Drivers (x2)
3. 25GA 500 RPM DC Geared Motors (x2)
4. Analog 14 IR Array (Curved/Boomerang)
5. 16-Channel Analog Multiplexer (e.g., CD74HC4067)
6. 3.7V 1S 1500mAh LiPo Batteries (x3 to make 11.1V pack)
7. 1.3" I2C OLED Display (SSH1106) & Rotary Encoder
8. Buzzer, MG90 Servos, HC-SR04+ (3.3V/5V compatible) Ultrasonic Sensor

### ❌ Need To Buy (Shopping List)
1. **Power**: DC-DC 3A Buck Step-down Power Supply Module (12V to 3.3V Fixed Output), Filtering Capacitors (2200µF, 1000µF).
2. **Mechanics**: 65mm Rubber Wheels x2, Ball Caster Wheel x1, Assorted standoffs/M3 screws.
3. **Task Additions**: Internal Trash Storage Tray, and an extra Micro Servo (e.g., MG90) for payload lifting and grab/release tasks. (Ensure all servos and sensors purchased can strictly operate on native 3.3V).
4. **Sensors**: MPU6050 6-Axis Gyroscope/Accelerometer module (for wobble detection and PID Auto-Tuning).
5. **Misc**: Voltage Detection Sensor Module (25V).

---

## 3. Wiring & Architecture

> ℹ️ **Note:** For a full, detailed breakdown of why these specific pins were chosen, please consult the independent [ESP32_S3_Pinout_Rules.md](ESP32_S3_Pinout_Rules.md) document. It outlines the avoidance of all Boot, Memory, and USB strapping pins.
> Additionally, while GPIO 10-13 (Motors) exist in the ADC2 range, they are utilized strictly for digital PWM output. This guarantees no conflict when Core 0 activates Wi-Fi, which otherwise locks ADC2 pins from analog reading.

### ESP32-S3 Optimized Pinouts
| Component | Pins (GPIO) | Function |
|:---|:---|:---|
| **MUX Data (Signal)** | 4 | ADC1_CH3. Reads the analog value from the MUX. |
| **Battery Voltage** | 5 | ADC1_CH4. Reads analog signal from Voltage Detection Module. |
| **I2C Bus (OLED & MPU6050)** | 8 (SDA), 9 (SCL) | Shared I2C bus for the OLED display and MPU6050 Gyro/Accel. Safe standard pins. |
| **Motors (Left/Right)** | 10, 11, 12, 13 | IBT2 RPWM/LPWM Controls. Safe standard digital outputs. |
| **MUX Selectors** | 14, 15, 16, 17 | Digital pins (S0-S3) to select which IR sensor is routed. |
| **Rotary Encoder** | 6 (CLK), 7 (DT), 18 (SW)| UI input, Start, and Kill Switch. Safe digital inputs. |
| **HC-SR04+ Ultrasonic**| 21 (Trig), 48 (Echo)| Obstacle detection. Safe digital I/O. |
| **Servos** | 1 (Lift), 2 (Grab/Release)| Actuators for payload picking and lifting. Safe digital outputs. |
| **Interface** | 47 (Buzzer) | Buzzer output. Safe digital output. |

### Battery Voltage Measurement
To safely monitor the 11.1V battery on a 3.3V-limited ESP32-S3 logic pin (GPIO 5):
* **Module**: Use a standard Voltage Detection Sensor Module (which features a built-in 5:1 resistor voltage divider: 30kΩ and 7.5kΩ).
* **Connection**: Connect **VCC/GND** of the module to the 11.1V battery. Connect **S (Signal)** to **GPIO 5** on the ESP32-S3, and **- (Minus)** to the ESP32 GND.
* **Math**: The module scales the 12.6V down by a factor of 5 (to max 2.52V), remaining completely safe for the 3.3V ADC. Compute voltage in code:
`Voltage = (analogRead(5) / 4095.0) * 3.3V * 5.0;`

### Power Distribution Schematic & Circuit Arrangement
* **11.1V Main Bus**: The 3S LiPo battery connects directly to the `VCC` / `GND` inputs of the two IBT2 motor drivers. 
  * **Capacitor Placement**: Place the **2200µF capacitor** in parallel across this 11.1V main bus (between Battery+ and Battery-) close to the motor drivers to absorb high-current voltage spikes.
* **3.3V Step-Down Converter (3A)**: Connected to the 11.1V bus, it drops the voltage to a stable 3.3V. This singular powerful buck converter powers the entirety of the 3.3V ecosystem: the ESP32-S3 (via 3.3V pin, NOT VIN), MG90 Servos (Lift and Grab/Release, verified to run at 3v3), HC-SR04+ Ultrasonic, IR Array, MUX board, OLED display, and all pull-up resistors. 
  * **Capacitor Placement**: Place the **1000µF capacitor** in parallel across the 3.3V output rails of the step-down converter to prevent logic brownouts when the servos actuate and draw sudden current.
*(Note: Because the entire logic system runs uniformly on 3.3V, no logic level shifting or voltage dividers are required on any sensor data lines).*

---

## 4. Systems & Algorithms

### A. Line Detection (Boomerang Geometry)
Due to the swept-back shape of the 14-sensor array, the outer edge sensors read horizontal lines slightly *later* than the forward center sensors.
* **Line Inversion Logic**: To adapt to track color swaps (Black on White vs. White on Black), the system uses a normalized 0-1000 scale for each sensor. A UI toggle inverts this mapping: 1000 always represents the "Line" and 0 always represents the "Background", ensuring the underlying navigation logic remains identical regardless of physical colors.
* **'X' / 'T' Junctions**: Code uses a short buffer (e.g., 20-50ms). When center hits the line color, and outer hits the line color shortly after, register intersection.
* **'Y' Splits**: Triggers immediately on extreme outer left/right sensors. The center stays on the background color. Use priority rule to pick a fork.
* **Broken Lines (Gaps)**: If all sensors see the background color natively in a straight segment, trigger a **coast** state matching previous PWM.
* **Dead Ends**: Rapid 180-degree U-Turn to reset the center sensor lock.

### B. PID Control
Position error utilizes a "weighted average" summation of all 14 analog streams:
```cpp
Weighted_Position = Σ(sensor_value[i] × position_weight[i]) / Σ(sensor_value[i])
Error = Weighted_Position - Zero_Center
MotorSpeed = Base_Speed ± ((Kp * Error) + (Ki * Integral) + (Kd * Derivative))
```
**PID Modes**:
* **Manual Mode**: Uses hardcoded/user-entered `Kp`, `Ki`, and `Kd` values.
* **Auto Mode**: Uses `kp_auto`, `ki_auto`, and `kd_auto` values, which are dynamically generated via MPU6050 analysis in the `Debug` testing session.

### C. Auto-Tuning Mathematics (Åström-Hägglund Relay & Ziegler-Nichols)
The active "Debug" mode employs the **Relay Feedback Auto-Tuning** algorithm (Åström & Hägglund, 1984) combined with **Ziegler-Nichols closed-loop tuning rules**. The MPU6050 acts as the measurement instrument to dynamically calculate the ultimate gain ($K_u$) and ultimate period ($T_u$) of chassis wobble.

1. **Relay Feedback Oscillation**:
   During a straight line segment, the robot temporarily suspends continuous proportional steering. It applies a fixed "Bang-Bang" steering amplitude (relay amplitude, $d$) to force a controlled, consistent wobble across the track line.
2. **Wobble Quantification**:
   The MPU6050's gyroscope (measuring yaw rate $\omega_z$) tracks this forced oscillation. The micro-controller measures:
   * **Peak Amplitude ($a$)**: The physical amplitude of the resulting angular velocity wobble.
   * **Ultimate Period ($T_u$)**: The time (in seconds) between two consecutive oscillation peaks.
3. **Ultimate Gain ($K_u$) Calculation**:
   Using the Describing Function analysis of a relay, the system's ultimate gain is calculated as:
   $$K_u = \frac{4d}{\pi a}$$
   *(Where $d$ is the steering relay magnitude, and $a$ is the measured wobble amplitude).*
4. **PID Parameter Derivation (Z-N Rules)**:
   Once $K_u$ and $T_u$ are acquired from the MPU6050 data stream over several oscillation cycles, the mathematically optimized "No-Wobble" PID tuning parameters are established:
   * **Proportional ($K_p$)**: $$K_p = 0.6 \times K_u$$
   * **Integral ($K_i$)**: $$K_i = 1.2 \times \frac{K_u}{T_u}$$
   * **Derivative ($K_d$)**: $$K_d = 0.075 \times K_u \times T_u$$
5. **Boundary Clamping & Application**:
   These mathematically derived parameters are verified against hard constraints to prevent integer overflow. Once bounded, they update the `kp_auto`, `ki_auto`, and `kd_auto` global variables respectively. Terminating the session guarantees they are saved to EEPROM for the Speed Run phase.

### D. Maze Solving (LSRB Pathing)
- **Primary Rule**: Left-Hand Method (Priority: 1=Left, 2=Straight, 3=Right, 4=Back).
- **Exploration Run**: The robot records the path string at each intersection (e.g., `L-S-R-B-L`).
- **Optimization phase**: After finding the end, the path string is compressed using U-Turn ('B') substitution (e.g., `L + B + L = S` and `L + B + R = B`). This clears all false paths.
- **Speed Run**: During the final measured run, the bot flies through nodes blind to the rule but obedient to the finalized optimized instruction array.

### E. Display UI, Telemetry, & EEPROM Memory
All calibration, tuning, and diagnostics are structured through the OLED display and rotary encoder, featuring a primary 4-option root menu. Additionally, it supports real-time wireless tuning during the test phase.

#### Main OLED Menu Structure:
1. **Run**: 
   * Starts the robot's active line-following routine. 
   * **Sparklines / Micro-charts**: The bottom 16 pixels of the OLED draw a continuous scrolling line chart (sparkline) of the PID Error to provide a fast visual indicator of chassis oscillation without staring at rapidly changing text.
   * **Acceleration/Velocity UI Gauge**: Displays a small visual progress bar on the OLED indicating the current dynamically scheduled $K_p$ or the base PWM power scale during S-Curve/Trapezoidal profiling.
   * Pressing the encoder button during a run triggers a **Pause State**. Motors halt immediately.
   * From Pause State, you can choose to **Resume** or **Terminate Session**. 
   * Crucially, while paused, you can exit to the main menu, navigate to `Setup` to tweak variables, and then return to `Run -> Resume` without losing your session state.
2. **Debug (Auto-Tune)**: 
   * Functions identically to **Run**, but actively relies on the **MPU6050** to monitor chassis stability.
   * **Wobble Analysis**: On straightaways or smooth curves, the bot analyzes the MPU6050's gyroscope ($ω_z$) and accelerometer ($a_y$) data to quantify high-frequency lateral oscillation (wobbling).
   * **Self-Tuning Math**: Triggers complex derivation/PID-auto-tuning logic on the fly. It adjusts Kp, Kd, and Ki in real-time specifically to dampen the detected wobble. (Note: Only executed during `Debug` mode to save processing clock-cycles otherwise).
   * **Web-Based Telemetry Dashboard (WebSockets)**: Exclusively in Debug mode, the ESP32 hosts a lightweight WebSocket server (leveraging Core 0). It serves a sleek HTML/JS page (stored in SPIFFS/LittleFS) to a laptop/phone, plotting live PID Errors, Battery Voltage, and Gyro dynamics on a real-time running chart (using Chart.js).
   * **Save & Apply**: Terminating the debug session permanently updates the stored variables `kp_auto`, `ki_auto`, and `kd_auto` for future auto runs.
3. **Calibrate**: 
   * Automates sensor tuning. The bot pauses for 1 second, then auto-rotates (spins in place) for 5 seconds.
   * While rotating, it takes continuous raw readings from all 14 sensors to find sweeping minimum (lowest) and maximum (highest) values.
   * Thresholds are calculated and **saved automatically and permanently to EEPROM**. The robot will remember its last calibration setup indefinitely and load it instantly on boot, until a new calibration sequence is strictly triggered by the user.
4. **Setup**: 
   * Select **PID Mode**: Toggle between **Auto PID** (utilizes the tuned `kp_auto` set), **Manual PID**, and **Dynamic PID** (gain scheduling based on track curvature).
   * Adjust **PID values** (`Kp`, `Ki`, `Kd`) explicitly for Manual mode.
   * Toggle **Dynamic Calibration**: Turn real-time adaptive thresholding on or off during runs.
   * **Servo Calibration**: Explicit OLED menu options to set and store neutral, minimum, and maximum payload servo positions.
   * Set **Base Speeds** independently for Left (`Base_L`) and Right (`Base_R`) motors.
   * Change hardware **Pinouts** dynamically.
   * Manually override **Thresholds** for individual sensors.
   * Toggle **Track Color Mode** ("Dark Line on Light" vs "Light Line on Dark").
   * **WiFi/Bluetooth Settings**: Enable/disable transceivers and set pairing modes.
5. **View**: 
   * **Analog Values**: Real-time live view of raw inputs from each IR sensor. **Graphical Sensor Array Visualization:** Instead of displaying a plain array of 14 numbers, the SSD1106 graphical capabilities draw a **live 14-bar histogram**, allowing you to instantly see which sensors are active and verify the "curve" of your analog reads visually.
   * **Path Replay Visualization**: Log and instantly replay the completed string-based maze path on the OLED or the WebSockets dashboard.
   * **Digital Values**: Real-time 0/1 binary view based on current thresholds.
   * **Thresholds**: Display currently saved threshold values.
   * **Wireless Status**: Check WiFi/Bluetooth connection status and see exactly what data is currently being received.

#### Wireless Live-Tuning (Testing Phase)
During the testing phase, the ESP32-S3 utilizes its onboard WiFi/Bluetooth to connect to a host interface (e.g., phone app or laptop). The robot can receive real-time packets containing new `Kp`, `Kd`, `Ki`, and `Base Speed (L/R)` values physically *during* a run. This allows adaptive tuning on the fly without ever stopping the robot.

**EEPROM Save/Load**: All adjusted PID constants, separated motor speeds, menu configurations, and sensor calibration arrays are automatically saved to the ESP32's non-volatile storage (NVS/EEPROM). Restarting the robot instantly reloads the perfect profile.

### F. Advanced Motion Control & Sensor Fusion
To push beyond standard line-following limits, the architecture incorporates real-time dynamics control:
- **Sensor Fusion (Gyro + IR)**: Actively utilizes the MPU6050 yaw ($\omega_z$) data during standard runs. If the robot encounters a significant line gap where all IR sensors read the background, the gyroscope enables closed-loop dead-reckoning to confidently maintain the precise heading until the line is reacquired.
- **Turn Angle Detection via MPU6050**: Validates mathematically true 90° and 180° U-Turns through raw yaw integration. This completely replaces standard time/delay-based turn calculations, dramatically increasing accuracy.
- **Trapezoidal / S-Curve Motion Profiling**: Implements algorithmic acceleration and deceleration curves rather than instantaneous PID base speed jumps. Ramping the motor PWMs prevents traction loss (wheel slip) and chassis jolts when accelerating out of corners or hard braking into acute intersections.
- **PID Gain Scheduling**: Replaces static PID tuning with dynamic arrays. The system smoothly interpolates between different $K_p$, $K_i$, and $K_d$ sets depending on the track curvature and forward velocity. High-speed straightaways utilize high $K_d$ and low $K_p$ for stability, while sharp, slow corners seamlessly shift to a high $K_p$ for aggressive turning.

### G. Sonar Sensor (HC-SR04+) Implementation
Crucial for the NRF26 rulebook trash collection mechanics:
- **Non-Blocking FreeRTOS on Core 0**: The standard `pulseIn()` function inherently blocks Core 1's PID controller loop. To maintain flight-controller speeds, Sonar runs entirely asynchronously via a hardware interrupt + timer architecture mapped to Core 0.
- **Detection Logic**: Safely identifies general obstacles, triggers the payload grab sequence when a target cube enters the 3–5cm physical striking boundary, and validates physical proximity against the 30x30x5cm end target zone wall.

### H. Payload Handling & Trash Dump Logic
The complete system for the Round 2 trash payload task:
- **Tracking (Multi-Payload Memory)**: Uses a payload counter and a dynamically allocated visited-node memory array to track whether a trash cube has been gathered from the 5–10 possible pick-up spots.
- **Pickup Algorithm**: Standard PID logic is preempted; the robot halts precisely via Sonar + IR, deploys the grabber, lifts the payload smoothly into internal chassis storage via GPIO 1 & 2, and restarts forward acceleration profiling.
- **Dump Zone Arrival Algorithm**: Leverages a dual-sensor condition logic. IR verifies the bold black target line boundary, whilst the Sonar verifies distance to the drop-off gate. Servos autonomously reverse to empty storage and finish the course.

---

## 5. Assembly & Best Practices

1. **Isolation & Power**: Even though the external 3.3V buck converter can supply 3 Amps, ensure the servos are powered directly from the buck converter's output pads, *not* daisy-chained through the ESP32-S3 development board. This prevents servo stall-currents from browning out the microcontroller.
2. **IR Array Height**: The 14 IR Array should sit around **3mm - 5mm** off the floor. 
3. **PID Tuning Flow**:
   - Set Kp, Ki, Kd to 0.
   - Ramp `Kp` up until the chassis visibly oscillates/wiggles over the line.
   - Reduce `Kp` slightly, then ramp `Kd` to smooth/dampen those wobbles out.
   - Increase `Ki` slightly only if the robot drifts heavily on long wide curves.

---

## 6. Advanced ESP32-S3 Features: Dual-Core & PSRAM

### FreeRTOS Dual-Core Utilization
The ESP32-S3 features two Xtensa LX7 cores. To prevent time-consuming operations (like updating the OLED screen) from interrupting the ultra-fast motor control loop, tasks are split using FreeRTOS:
* **Core 0 (PRO_CPU) - Interface & Peripherals**: Dedicated to non-time-critical background tasks. This core handles the I2C OLED display rendering, rotary encoder UI menu navigation, and general state tracking.
* **Core 1 (APP_CPU) - Flight Controller**: Dedicated exclusively to time-critical robotic operations. Runs the 14-channel MUX high-speed analog reads, PID error calculations, motor PWM output generation, and immediate intersection detection. This ensures ultra-low latency response times.
* **Concurrency & Resource Protection (Avoiding Clashes)**: To prevent hardware collisions (like I2C bus timing crashes between cores), strict peripheral isolation is enforced. **Only Core 0** is allowed to execute `Wire` (I2C) display commands. When Core 1 calculates new variables that need displaying or saving, it passes them to Core 0 safely using **FreeRTOS Queues** or variables secured by **Mutexes/Spinlocks**.
* **Implementation**: Realized by wrapping loops in `xTaskCreatePinnedToCore()`.

### Octal PSRAM (8MB) Utilization
The N16R8 variant has an absolutely massive 8MB of fast PSRAM. Since standard internal SRAM is limited (~512KB) and needed for rapid FreeRTOS stack execution, bulk data is shifted to PSRAM:
* **Maze Path Arrays**: During exploration, massive node histories (instructions like `L`, `S`, `R`, `B`) and coordinate data are dynamically stored in PSRAM using `ps_malloc()`. This guarantees the robot will never run out of memory regardless of maze complexity.
* **Frame Buffers & Telemetry**: Full-screen graphic buffers for the OLED and logged PID performance history are pushed entirely to PSRAM.
* **Hardware Confirmation**: Because all GPIOs from 26 to 38 were strictly avoided in our optimized pinout array, the Octal PSRAM interface is guaranteed continuous, unrestricted high-speed data flow.

### Over-The-Air (OTA) Updates
Since the bot is tightly packed for competition and already utilizes Wi-Fi for telemetry, `ArduinoOTA` (or ESP-IDF OTA) is implemented. This allows flashing new code wire-free while the robot is powered entirely by its battery on the track.

---

## 7. References & Links
* **HC-SR04+ Ultrasonic Sensor**: [RoboticsBD - 3.3V/5V Compatible Sonar Sensor](https://store.roboticsbd.com/sensors/2446-hc-sr04-plus-ultrasonic-sonar-sensor-2021-33v-5v-robotics-bangladesh.html)
* **Voltage Detection Module**: [RoboticsBD - 25V Voltage Sensor Module](https://store.roboticsbd.com/sensors/2675-voltage-detection-sensor-module-25v-robotics-bangladesh.html)
* **PID Auto-Tuning Research**: Åström, K. J., & Hägglund, T. (1984). "Automatic tuning of simple regulators with specifications on phase and amplitude margins." *Automatica*, 20(5), 645-651. [Wikipedia: Ziegler-Nichols tuning](https://en.wikipedia.org/wiki/Ziegler%E2%80%93Nichols_method)
