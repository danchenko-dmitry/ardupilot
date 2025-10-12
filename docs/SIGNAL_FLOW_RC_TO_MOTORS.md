# ArduRover Signal Flow: RC Input to Motor Output
## Complete Technical Deep-Dive with Code References

**Document:** Complete signal path tracing through ArduPilot source code  
**Target:** Understanding how RC inputs become motor PWM outputs  
**Level:** Advanced / Developer-oriented  
**Date:** 2025-10-10

---

## 📋 TABLE OF CONTENTS

1. [Overview](#overview)
2. [The Complete Signal Path](#the-complete-signal-path)
3. [Layer-by-Layer Breakdown](#layer-by-layer-breakdown)
4. [Code References](#code-references)
5. [Timing & Execution](#timing--execution)
6. [Tank-Specific Example](#tank-specific-example)
7. [Data Structures](#data-structures)

---

## 🎯 Overview

### **The 10,000-Foot View:**

```
┌─────────────────────────────────────────────────────────────┐
│                    COMMAND PATH (Forward)                    │
│              "What the pilot WANTS to do"                    │
└─────────────────────────────────────────────────────────────┘

[RC Transmitter]
      ↓ Radio waves
[RC Receiver]
      ↓ CRSF/SBUS/PPM
[HAL - Hardware Abstraction Layer]
      ↓ Decoded RC values
[RC_Channels - RC Input Management]
      ↓ Normalized inputs
[Mode - Current Flight Mode]
      ↓ Control commands (steering, throttle)
[Controllers - Speed/Steering PID] (optional)
      ↓ Adjusted outputs
[AR_Motors - Motor Mixing]  ←──────────────┐
      ↓ Individual motor commands            │
[SRV_Channels - Servo Management]           │
      ↓ PWM values                           │
[HAL - PWM Output]                           │
      ↓ Hardware timers                      │
[ESCs]                                       │
      ↓ Motor power                          │
[Physical Motors] → Vehicle moves            │
                                             │
┌────────────────────────────────────────────┴─────────────────┐
│                  FEEDBACK PATH (Sensors)                      │
│          "What the vehicle is ACTUALLY doing"                 │
└───────────────────────────────────────────────────────────────┘

[IMU - Accelerometers & Gyros]
[GPS - Position & Velocity]      ┐
[Compass - Heading]              ├→ Sensor fusion
[Barometer - Altitude]           │
[Wheel Encoders] (optional)      ┘
      ↓
[AHRS/EKF - Extended Kalman Filter]
      ↓ Fused state estimate
[AR_AttitudeControl::get_forward_speed()]
      ↓ Current speed (m/s)
[AR_Motors::output()] ──────────────┘
      Uses current speed for advanced features
      (e.g., speed scaling, balance control)
```

⭐ **KEY INSIGHT:**
- **Command path:** RC → Mode → Motors → PWM (what you command)
- **Feedback path:** Sensors → AHRS → Speed estimate (what actually happens)
- **They meet at:** `output()` function uses BOTH command and feedback

---

## 🔐 Protocol Abstraction: How CRSF Works

### **⚠️ CRITICAL CONCEPT: No set_pwm() During RC Input!**

**Actual Code Path:**
```cpp
// ✅ CORRECT - What actually happens:
radio_in = hal.rcin->read(ch_in);      // Read from HAL
control_in = pwm_to_angle();           // Convert to normalized
```

**The Actual Flow:**

```
┌─────────────────────────────────────────────────────┐
│ CRSF Receiver (Connected to UART2)                  │
│                                                      │
│ Sends serial packet:                                │
│ [0xC8][len][type][CH1:992][CH3:1105][...][CRC]     │
│ ↓ 420000 baud serial data                          │
└─────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ HAL Layer: libraries/AP_HAL_ChibiOS/RCInput.cpp    │
│                                                      │
│ void UART_IRQHandler() {                            │
│     // Parse CRSF packet                            │
│     // Extract 16 channels (10-bit values)          │
│     // Convert to µs equivalent:                    │
│     _rc_values[0] = 1450;  // Ch1: 992 → 1450µs    │
│     _rc_values[2] = 1650;  // Ch3: 1105 → 1650µs   │
│ }                                                    │
└─────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ RC_Channel Layer: libraries/RC_Channel/RC_Channel.cpp
│                                                      │
│ bool RC_Channel::update() {                         │
│     radio_in = hal.rcin->read(ch_in);  ← KEY LINE! │
│     control_in = pwm_to_angle();                    │
│     return true;                                     │
│ }                                                    │
│                                                      │
│ ⚠️ NO set_pwm() call!                               │
└─────────────────────────────────────────────────────┘
```

### **Protocol Transparency:**

The beauty of this design is that `RC_Channel` doesn't care about the underlying protocol:

| Protocol | Physical Connection | HAL Processing | Result to RC_Channel |
|----------|---------------------|----------------|----------------------|
| **CRSF** | UART (TX+RX wires) | Parse serial packets | µs values (1000-2000) |
| **PPM** | Single signal wire | Measure pulse widths | µs values (1000-2000) |
| **SBUS** | Single signal wire (inverted) | Parse serial frames | µs values (1000-2000) |

**All protocols → HAL converts → µs values → `hal.rcin->read()` returns same format**

### **Complete CRSF Processing Flow:**

```
┌──────────────────────────────────────────────────────────────┐
│ LAYER 1: CRSF Receiver (Hardware)                           │
│ ─────────────────────────────────────────────────────────    │
│ Input: Pilot moves stick                                     │
│ Output: Serial packet [0xC8][...][CH1:992][CH3:1105][CRC]   │
│ Frequency: 150-200 Hz                                        │
└──────────────────────────────────────────────────────────────┘
                          ↓ UART @ 420000 baud
┌──────────────────────────────────────────────────────────────┐
│ LAYER 2: HAL - Protocol Abstraction                         │
│ ─────────────────────────────────────────────────────────    │
│ File: libraries/AP_HAL_ChibiOS/RCInput.cpp                  │
│ Input: CRSF serial packets                                   │
│ Processing:                                                   │
│   1. UART interrupt captures bytes                           │
│   2. Validates CRC                                            │
│   3. Extracts 10-bit channel values (0-1984)                 │
│   4. Converts to µs equivalent:                              │
│      992 → 1575µs, 1105 → 1650µs                            │
│ Output: _rc_values[] array in µs                             │
│ Function: hal.rcin->read(ch_in) returns µs                   │
└──────────────────────────────────────────────────────────────┘
                          ↓ µs values
┌──────────────────────────────────────────────────────────────┐
│ LAYER 3: RC_Channel - Normalization (PROTOCOL AGNOSTIC!)    │
│ ─────────────────────────────────────────────────────────    │
│ File: libraries/RC_Channel/RC_Channel.cpp:301-318           │
│ Input: radio_in = 1575µs (from HAL)                          │
│ Processing:                                                   │
│   Line 306: radio_in = hal.rcin->read(ch_in);               │
│   ✅ Line 315: control_in = pwm_to_angle();                 │
│   ✅ Line 312: control_in = pwm_to_range();                 │
│                                                               │
│   These functions:                                            │
│   - Apply RC1_MIN, RC1_MAX, RC1_TRIM calibration             │
│   - Apply deadzone                                            │
│   - Scale to -4500/+4500 (steering) or -100/+100 (throttle) │
│ Output: control_in = normalized values                       │
│ ⚠️  SAME CODE RUNS FOR CRSF, PPM, AND SBUS!                 │
└──────────────────────────────────────────────────────────────┘
                          ↓ Normalized values
┌──────────────────────────────────────────────────────────────┐
│ LAYER 4+: Mode Processing → Motors → PWM Output             │
│ (Rest of the signal flow is identical for all protocols)    │
└──────────────────────────────────────────────────────────────┘
```

### **To Answer Your Question Directly:**

**Q: Do we dive into `pwm_to_angle()` and `pwm_to_range()` when we have CRSF?**

**A: YES! 100% YES!** ✅

These functions **ALWAYS execute** regardless of protocol. Here's why:

1. **HAL abstracts the protocol** - CRSF packets are converted to µs values
2. **RC_Channel only sees µs values** - doesn't know if they came from CRSF/PPM/SBUS
3. **pwm_to_angle/pwm_to_range always run** - they convert µs → normalized values
4. **The names are misleading** - "pwm" is historical, they work with µs from any source

**Code proof:**
```cpp
// This ALWAYS executes (line 311-316) - no if-statement checking protocol!
if (type_in == ControlType::RANGE) {
    control_in = pwm_to_range();      // Throttle
} else {
    control_in = pwm_to_angle();      // Steering  
}
```

### **When IS set_pwm() Used?**

```cpp
// Only for GCS overrides:
if (has_override()) {
    radio_in = override_value;  // From Mission Planner/QGC
} else {
    radio_in = hal.rcin->read(ch_in);  // From RC receiver
}
```

---

## 🔄 The Complete Signal Path

### **End-to-End Journey (Tank Example):**

```
STEP 1: RC Transmitter
├── Pilot moves throttle stick 50% forward
├── Pilot moves steering stick 30% right
├── Transmitter encodes: Ch1=1450, Ch3=1650
└── Transmits via 2.4GHz radio

STEP 2: RC Receiver
├── Receives radio signal
├── Decodes channels
├── Outputs CRSF serial data (or PPM/SBUS for other receivers)
│   CRSF: Digital packet stream at 420000 baud
│   Contains: Ch1=1450, Ch3=1650, RSSI, link quality, etc.
└── Sends to flight controller UART2 (TX/RX pins)

STEP 3: HAL (Hardware Layer)
├── File: libraries/AP_HAL_ChibiOS/RCInput.cpp
├── For CRSF: UART interrupt reads serial packets
├── For PPM/SBUS: Timer interrupt captures signal edges
├── Decodes channel values from protocol
└── Stores: rc_values[0]=1450, rc_values[2]=1650

STEP 4: RC_Channels (Input Processing)
├── File: libraries/RC_Channel/RC_Channels.cpp
├── Function: RC_Channels::read_input()
├── Called: 50 Hz (from scheduler)
├── For each channel, calls: RC_Channel::update()
│   ├── Code: radio_in = hal.rcin->read(ch_in);
│   │   └── HAL returns µs value (extracted from CRSF packet)
│   ├── Code: control_in = pwm_to_range(); // or pwm_to_angle()
│   │   └── Converts µs to normalized range
│   ├── Applies calibration (RC1_MIN, RC1_MAX, RC1_TRIM)
│   └── Applies deadzone
├── Maps via RCMAP (RCMAP_ROLL→Ch1, RCMAP_THROTTLE→Ch3)
├── Results:
│   ├── Steering Ch1: 1450µs → control_in = -1500 (range: -4500 to +4500)
│   └── Throttle Ch3: 1650µs → control_in = +50 (range: -100 to +100)


STEP 5: Scheduler
├── File: Rover/Rover.cpp
├── Main loop runs at 400 Hz
├── Scheduler calls tasks based on priority
├── Key tasks:
│   ├── read_radio() - 50 Hz - Reads RC
│   ├── update_current_mode() - 400 Hz - Calls mode
│   └── set_servos() - 400 Hz - Outputs to motors

STEP 6: Mode Processing
├── File: Rover/mode_manual.cpp (if in Manual mode)
├── Function: ModeManual::update()
├── Called: 400 Hz (via update_current_mode)
├── Gets pilot input:
│   ├── get_pilot_desired_steering_and_throttle(steer, thr)
│   ├── Returns: steer=-1500, thr=50
├── Sends to motors:
│   ├── g2.motors.set_steering(-1500)    # Stores in _steering
│   └── g2.motors.set_throttle(50)       # Stores in _throttle

⭐ IMPORTANT: Two-phase motor control:
   Phase 1: set_steering() and set_throttle() STORE values
   Phase 2: output() USES stored values to generate PWM

STEP 6.5: Set Servos (Calls Output)
├── File: Rover/Steering.cpp
├── Function: Rover::set_servos()
├── Called: 400 Hz from main loop
├── Code:
│   float speed = 0.0f;
│   g2.attitude_control.get_forward_speed(speed);  // Gets current speed
│   g2.motors.output(arming.is_armed(), speed, G_Dt);
├── Passes to output():
│   ├── armed = true (from arming.is_armed())
│   ├── ground_speed = 2.5 m/s (from attitude_control → SENSOR DATA!)
│   └── dt = 0.0025 (time delta, G_Dt)

⭐ IMPORTANT: Where does ground_speed come from?
   get_forward_speed() READS from sensors, not from control commands!
   
   Source chain:
   1. AHRS/EKF (Extended Kalman Filter) - fuses IMU, GPS, compass
   2. Calculates velocity estimate (North, East, Down)
   3. Projects into vehicle body frame
   4. If AHRS unavailable → fallback to GPS ground_speed
   
   This is ACTUAL vehicle speed, not commanded speed!

STEP 7: AR_Motors (Motor Library)
├── File: libraries/AR_Motors/AP_MotorsUGV.cpp
├── Function: AP_MotorsUGV::output(bool armed, float ground_speed, float dt)
├── Called: 400 Hz (from set_servos)
├── Receives: armed=true, ground_speed=2.5, dt=0.0025
├── Uses member variables set earlier (in Step 6):
│   ├── _steering = -1500 (set by set_steering())
│   └── _throttle = 50 (set by set_throttle())
├── Calls: output_skid_steering(armed, _steering, _throttle, dt)

STEP 8: Skid-Steer Mixing
├── Function: output_skid_steering()
├── Code location: AR_Motors/AP_MotorsUGV.cpp:804
├── Mixing calculation:
│   ├── steering_scaled = -1500 / 4500 = -0.33
│   ├── throttle_scaled = 50 * 0.01 = 0.50
│   ├── motor_left  = 0.50 + (-0.33) = 0.17 (17%)
│   ├── motor_right = 0.50 - (-0.33) = 0.83 (83%)
├── Calls: output_throttle(k_throttleLeft, 17.0)
└── Calls: output_throttle(k_throttleRight, 83.0)

STEP 9: SRV_Channels (Servo Management)
├── File: libraries/SRV_Channel/SRV_Channels.cpp
├── Function: SRV_Channels::set_output_scaled()
├── Receives: function=k_throttleLeft, value=17.0
├── Finds ALL channels with function 73 (ThrottleLeft)
│   ├── SERVO1_FUNCTION = 73 ✓
│   └── SERVO2_FUNCTION = 73 ✓
├── Converts 17.0% to PWM:
│   ├── Range: SERVO1_MIN=1000 to SERVO1_MAX=2000
│   ├── 17% = 1000 + (0.17 × 1000) = 1170 µs
└── Sets: servo1_output_pwm = 1170, servo2_output_pwm = 1170

STEP 10: PWM Calculation
├── Function: SRV_Channels::calc_pwm()
├── For each servo channel:
│   ├── Applies REVERSED if needed
│   ├── Applies MIN/MAX limits
│   ├── Applies TRIM offset
│   └── Final PWM value ready

STEP 11: PWM Output
├── Function: SRV_Channels::output_ch_all()
├── Calls HAL: hal.rcout->write(channel, pwm)
├── Hardware layer writes to timer registers
├── PWM waveforms generated:
│   ├── SERVO1: 1170 µs pulse every 20ms
│   ├── SERVO2: 1170 µs pulse every 20ms
│   ├── SERVO3: 1830 µs pulse every 20ms
│   └── SERVO4: 1830 µs pulse every 20ms

STEP 12: ESCs
├── ESCs read PWM pulses
├── Decode duty cycle:
│   ├── 1170µs → 17% forward (left motors)
│   └── 1830µs → 83% forward (right motors)
├── Generate motor drive signals
└── Apply power to motors

STEP 13: Physical Motors
├── Left motors (17%): Slow forward
├── Right motors (83%): Fast forward
└── RESULT: Tank turns left!
```

**Total latency:** ~7-10ms from RC stick movement to motor response (CRSF)
                  ~20-30ms with PPM protocol

---

## 📡 CRSF Protocol Configuration

### **What is CRSF?**

CRSF (Crossfire) is a modern, digital RC protocol developed by TBS (Team BlackSheep) and also used by ExpressLRS and other systems. It's significantly superior to legacy PPM/SBUS protocols.

### **Hardware Connection:**

```
┌─────────────────────────────────────────────────────┐
│  CRSF Receiver          SpeedyBee F405 V3          │
│  ┌──────────┐           ┌──────────┐               │
│  │          │           │          │               │
│  │  TX ─────┼──────────►│ R2 (RX2) │               │
│  │  RX ◄────┼───────────┤ T2 (TX2) │               │
│  │  5V ◄────┼───────────┤ 5V       │               │
│  │  GND ────┼──────────►│ GND      │               │
│  │          │           │          │               │
│  └──────────┘           └──────────┘               │
└─────────────────────────────────────────────────────┘

⭐ KEY: Both TX and RX must be connected for bidirectional communication!
```

### **Software Configuration:**

```ini
# UART Configuration for CRSF
SERIAL2_PROTOCOL = 23        # 23 = RCIN (RC Input via serial)
SERIAL2_BAUD = 115           # 115200, auto-negotiates to 420000
SERIAL2_OPTIONS = 0          # Normal (no inversion needed)

# RSSI Configuration
RSSI_TYPE = 3                # Get RSSI from RC protocol (CRSF)

# Optional: Enable CRSF telemetry to transmitter
RC_OPTIONS = 256             # Bit 8: CRSF telemetry passthrough
```

### **CRSF Protocol Features:**

| **Feature** | **Details** |
|-------------|-------------|
| **Channels** | 16 standard (up to 32 possible) |
| **Update Rate** | 150-200 Hz (vs. 50 Hz for PPM) |
| **Latency** | 5-10ms end-to-end |
| **Resolution** | 10-12 bits per channel |
| **Bidirectional** | Yes - telemetry flows back to TX |
| **Telemetry Data** | Battery, GPS, attitude, mode, etc. |
| **RSSI/LQ** | Built-in signal strength & link quality |
| **Failsafe** | Protocol-level failsafe detection |
| **Connection** | Full UART (TX + RX wires) |

### **Data Flow (CRSF Specific):**

```
Transmitter → 2.4GHz Radio → Receiver
      ↓
   Formats CRSF packet:
   [Sync][Len][Type][16xChannels][RSSI][LQ][CRC]
      ↓
   Sends via UART at 420000 baud
      ↓
Flight Controller UART2 RX pin
      ↓
   UART interrupt handler
      ↓
   Parse packet, extract channels
      ↓
   Store in rc_values[] array
      ↓
   (Same as PPM/SBUS from here on)
```

### **Why CRSF is Better:**

1. **Lower Latency:** 5-10ms vs. 20ms (PPM) or 14-18ms (SBUS)
2. **More Channels:** 16+ vs. 8-12 (PPM) or 16 (SBUS)
3. **Telemetry:** View battery, GPS, mode on transmitter screen
4. **Better Failsafe:** Protocol-level detection vs. timeout-based
5. **Digital:** Immune to electrical noise
6. **RSSI Built-in:** Always know signal strength

---

## 📂 Layer-by-Layer Breakdown

---

## **Layer 1: RC Hardware Input** 🔌

### **File:** `libraries/AP_HAL_ChibiOS/RCInput.cpp`

**What happens (for CRSF protocol):**

**CRSF (Crossfire) - Modern Digital Protocol:**
```cpp
// UART interrupt reads serial packets from CRSF receiver
// SERIAL2_PROTOCOL = 23 (RCIN)
// Baud: 420000 (auto-negotiated from 115200)

void UART_IRQHandler(void) {
    // Read CRSF packet (digital data)
    // Format: [sync byte][length][type][payload][CRC]
    // Payload contains: 16 channels + RSSI + LQ + telemetry
    
    // Extract channel data
    _rc_values[0] = extract_channel_1();  // 1450 µs
    _rc_values[1] = extract_channel_2();  // 1500 µs
    _rc_values[2] = extract_channel_3();  // 1650 µs
    // ... etc for 16 channels
    
    // Also extracts:
    _rssi = extract_rssi();         // Signal strength
    _link_quality = extract_lq();   // Link quality %
}
```

**Alternative: PPM/SBUS (Legacy Protocols):**
```cpp
// For PPM/SBUS: Hardware timer interrupt captures signal edges
void RCInput::_timer_tick(void) {
    // Interrupt handler
    // Measures pulse widths
    // Stores in buffer
    _rc_values[channel] = pulse_width_us;
}
```

**Key differences:**

| Feature | CRSF | PPM | SBUS |
|---------|------|-----|------|
| Connection | UART (TX+RX) | Single wire | Single wire |
| Channels | 16+ | 8-12 | 16 |
| Latency | 5-10ms | 20ms | 14-18ms |
| Bidirectional | ✅ Yes (telemetry) | ❌ No | ❌ No |
| RSSI | ✅ Built-in | ❌ No | ⚠️ Optional |
| Update rate | 150-200 Hz | 50 Hz | 70-100 Hz |

**Key functions:**
- **CRSF:** UART RX interrupt handler, packet parser
- **PPM/SBUS:** `_timer_tick()` - ISR (Interrupt Service Routine)
- Hardware layer abstracts protocol differences
- Stores normalized µs values per channel

**Output:**
```
Raw RC values (microseconds):
rc_values[0] = 1450  // Channel 1 (Steering)
rc_values[1] = 1500  // Channel 2  
rc_values[2] = 1650  // Channel 3 (Throttle)
rc_values[3] = 1500  // Channel 4
rc_values[4] = 1425  // Channel 5 (Mode switch)
...
```

---

## **Layer 2: RC Input Reading** 📡

### **File:** `Rover/radio.cpp`

**Function:** `Rover::read_radio()`  
**Called:** 50 Hz (from scheduler)  
**Lines:** 64-75

```cpp
void Rover::read_radio()
{
    // Read RC input from HAL
    if (!rc().read_input()) {
        // check if we lost RC link
        radio_failsafe_check(channel_throttle->get_radio_in());
        return;
    }

    failsafe.last_valid_rc_ms = AP_HAL::millis();
    // check that RC values are valid
    radio_failsafe_check(channel_throttle->get_radio_in());
}
```

**What it does:**
1. Calls `RC_Channels::read_input()`
2. Fetches latest values from HAL
3. Checks for failsafe condition
4. Updates timestamp

---

## **Layer 3: RC Channel Processing** 🎮

### **File:** `libraries/RC_Channel/RC_Channels.cpp`

**Function:** `RC_Channels::read_input()`  
**Lines:** 76-99

```cpp
bool RC_Channels::read_input(void)
{
    if (hal.rcin->new_input()) {
        _has_had_rc_receiver = true;
    } else if (!has_new_overrides) {
        return false;
    }

    last_update_ms = AP_HAL::millis();

    bool success = false;
    for (uint8_t i=0; i<NUM_RC_CHANNELS; i++) {
        success |= channel(i)->update();  // ← Calls RC_Channel::update()
    }

    return success;
}
```

**Then channel normalization:**

### **File:** `libraries/RC_Channel/RC_Channel.cpp`  
**Lines:** 301-319

```cpp
// ACTUAL CODE - RC_Channel::update()
bool RC_Channel::update(void)
{
    if (has_override() && !rc().option_is_enabled(RC_Channels::Option::IGNORE_OVERRIDES)) {
        radio_in = override_value;
    } else if (rc().has_had_rc_receiver() && !rc().option_is_enabled(RC_Channels::Option::IGNORE_RECEIVER)) {
        // THIS IS THE KEY LINE - reads from HAL
        radio_in = hal.rcin->read(ch_in);  // ← HAL returns µs value
    } else {
        return false;
    }

    // Convert µs to normalized value
    if (type_in == ControlType::RANGE) {
        control_in = pwm_to_range();  // For throttle: -100 to +100
    } else {
        // ControlType::ANGLE
        control_in = pwm_to_angle();  // For steering: -4500 to +4500
    }

    return true;
}
```

**How HAL abstracts protocols:**
```
CRSF Protocol:
  ├── Receiver sends serial packet at 420000 baud
  ├── HAL UART interrupt reads packet
  ├── HAL parses CRSF format: [sync][len][type][channels][CRC]
  ├── HAL extracts 16 channels (10-bit values)
  ├── HAL converts to µs equivalent (e.g., 992 → 1500µs)
  └── hal.rcin->read(ch_in) returns this µs value

Legacy PPM/SBUS:
  ├── Timer interrupt captures pulse edges
  ├── HAL measures pulse widths directly in µs
  └── hal.rcin->read(ch_in) returns measured µs value

Result: RC_Channel doesn't care which protocol - always gets µs!
```

**RCMAP (Channel Mapping):**

```cpp
// File: Rover/radio.cpp
void Rover::set_control_channels() {
    // Map physical channels to functions
    // RCMAP_ROLL = 1 → Channel 1 is "roll" (steering for rover)
    // RCMAP_THROTTLE = 3 → Channel 3 is throttle
    
    channel_steer = &rc().get_roll_channel();      // Ch1
    channel_throttle = &rc().get_throttle_channel(); // Ch3
    
    // Set value ranges
    channel_steer->set_angle(SERVO_MAX);     // -4500 to +4500
    channel_throttle->set_angle(100);         // -100 to +100
}
```

**Output:**
```
channel_steer->get_control_in() = -1500  // Normalized steering
channel_throttle->get_control_in() = 50   // Normalized throttle
```

---

## **Layer 4: Main Loop Scheduler** ⏱️

### **File:** `Rover/Rover.cpp`  
**Lines:** 69-100

```cpp
const AP_Scheduler::Task Rover::scheduler_tasks[] = {
    //         Function name,       Hz,    µs, Priority
    SCHED_TASK(read_radio,          50,   200,   3),  // Read RC
    SCHED_TASK(ahrs_update,        400,   400,   6),  // Update AHRS
    SCHED_TASK(update_current_mode,400,   200,  12),  // Call mode update
    SCHED_TASK(set_servos,         400,   200,  15),  // Output to motors
    // ... more tasks
};
```

**Execution flow (every 2.5ms - 400 Hz):**
```
Loop iteration N:
├── read_radio() [if N % 8 == 0] (50 Hz)
├── ahrs_update() [every iteration] (400 Hz)
├── update_current_mode() [every iteration] (400 Hz)
└── set_servos() [every iteration] (400 Hz)
```

---

## **Layer 5: Mode Update** 🎮

### **File:** `Rover/Rover.cpp`  
**Lines:** 516-525

```cpp
void Rover::update_current_mode(void)
{
    // check for emergency stop
    if (SRV_Channels::get_emergency_stop()) {
        g2.attitude_control.relax_I();
    }

    // Call current mode's update function
    control_mode->update();  // ← Polymorphic call
}
```

**Mode-specific update examples:**

---

### **Manual Mode:** `Rover/mode_manual.cpp`

```cpp
void ModeManual::update()
{
    float desired_steering, desired_throttle, desired_lateral;
    
    // Get pilot input from RC
    get_pilot_desired_steering_and_throttle(desired_steering, desired_throttle);
    get_pilot_desired_lateral(desired_lateral);
    // Returns: steering=-1500, throttle=50, lateral=0
    
    // Apply manual expo (curve)
    desired_steering = 4500.0 * input_expo(desired_steering / 4500, 
                                            g2.manual_steering_expo);
    
    // Balance bot adjustment (if applicable)
    if (rover.is_balancebot()) {
        rover.balancebot_pitch_control(desired_throttle);
    }
    
    // Walking robot inputs
    float desired_roll, desired_pitch, desired_walking_height;
    get_pilot_desired_roll_and_pitch(desired_roll, desired_pitch);
    get_pilot_desired_walking_height(desired_walking_height);
    g2.motors.set_roll(desired_roll);
    g2.motors.set_pitch(desired_pitch);
    g2.motors.set_walking_height(desired_walking_height);
    
    // Sailboat
    g2.sailboat.set_pilot_desired_mainsail();
    
    // SEND TO MOTORS ⭐
    g2.motors.set_throttle(desired_throttle);  // throttle = 50
    g2.motors.set_steering(desired_steering,   // steering = -1500
                          (g2.manual_options & ManualOptions::SPEED_SCALING));
    g2.motors.set_lateral(desired_lateral);    // lateral = 0
}
```

**Key point:** Manual mode passes RC inputs **directly** to motors (with minimal processing)

---

### **Auto Mode:** `Rover/mode_auto.cpp`

```cpp
void ModeAuto::update()
{
    // Navigate to waypoint
    navigate_to_waypoint();  
    // ↑ This calculates desired speed and lateral acceleration
}

void Mode::navigate_to_waypoint() {
    // Use waypoint navigation library
    g2.wp_nav.update(rover.G_Dt);
    
    // Get desired speed and lateral accel
    float desired_speed = g2.wp_nav.get_speed();
    float desired_lat_accel = g2.wp_nav.get_lat_accel();
    
    // Convert to throttle
    calc_throttle(desired_speed, avoidance_enabled);
    
    // Convert to steering
    calc_steering_from_lateral_acceleration(desired_lat_accel);
}
```

**Key point:** Auto mode **calculates** desired values from waypoint geometry, then uses controllers

---

## **Layer 6: Control Algorithms** 🧮

### **A) Speed Controller**

**File:** `libraries/APM_Control/AR_AttitudeControl.cpp`  
**Called from:** `Mode::calc_throttle()`

```cpp
void Mode::calc_throttle(float target_speed, bool avoidance_enabled)
{
    float throttle_out = 0.0f;

    if (g2.sailboat.sail_enabled()) {
        // Sailboat-specific
        g2.sailboat.get_throttle_and_set_mainsail(target_speed, throttle_out);
    } else {
        // Call speed controller
        if (is_zero(target_speed) && !rover.is_balancebot()) {
            // Stopping
            bool stopped;
            throttle_out = 100.0f * attitude_control.get_throttle_out_stop(
                g2.motors.limit.throttle_lower, 
                g2.motors.limit.throttle_upper, 
                g.speed_cruise, 
                g.throttle_cruise * 0.01f, 
                rover.G_Dt, 
                stopped);
        } else {
            // Speed control ⭐ THIS IS THE PID CONTROLLER
            bool motor_lim_low = g2.motors.limit.throttle_lower || 
                                 attitude_control.pitch_limited();
            bool motor_lim_high = g2.motors.limit.throttle_upper || 
                                  attitude_control.pitch_limited();
                                  
            throttle_out = 100.0f * attitude_control.get_throttle_out_speed(
                target_speed,          // Target (from CRUISE_SPEED or calc)
                motor_lim_low,
                motor_lim_high,
                g.speed_cruise,        // CRUISE_SPEED parameter
                g.throttle_cruise * 0.01f,  // CRUISE_THROTTLE
                rover.G_Dt);
            // Returns throttle percentage (-100 to +100)
        }

        // Balance bot pitch adjustment
        if (rover.is_balancebot()) {
            rover.balancebot_pitch_control(throttle_out);
        }
    }

    // SEND TO MOTORS
    g2.motors.set_throttle(throttle_out);
}
```

**Speed PID controller (AR_AttitudeControl):**
```cpp
float AR_AttitudeControl::get_throttle_out_speed(float desired_speed, ...) {
    // Get current speed from GPS/wheel encoders
    float speed = 0;
    get_forward_speed(speed);  // e.g., 1.85 m/s
    
    // Calculate error
    float speed_error = desired_speed - speed;  // 2.0 - 1.85 = 0.15
    
    // PID calculation
    float p_out = _throttle_speed_pid.get_p(speed_error);     // P term
    float i_out = _throttle_speed_pid.get_i(speed_error, dt); // I term
    float d_out = _throttle_speed_pid.get_d(speed_error, dt); // D term
    float ff_out = _throttle_speed_pid.get_ff();              // FF term
    
    float output = p_out + i_out + d_out + ff_out;
    
    // Constrain and return
    return constrain_float(output, -1.0, 1.0);  // -100% to +100%
}
```

---

### **B) Steering Controller**

**File:** `libraries/APM_Control/AR_AttitudeControl.cpp`  
**Called from:** `Mode::calc_steering_to_heading()` or `calc_steering_from_lateral_acceleration()`

```cpp
void Mode::calc_steering_to_heading(float desired_heading_cd, 
                                     float rate_max_degs)
{
    // Calculate desired turn rate from heading error
    // Uses angle controller
    float desired_turn_rate_rads = attitude_control.get_desired_turn_rate(
        desired_heading_cd * 0.01f,  // Desired heading
        rate_max_degs);               // Max rate limit
    
    // Convert turn rate to steering output
    calc_steering_from_turn_rate(desired_turn_rate_rads);
}

void Mode::calc_steering_from_turn_rate(float turn_rate)
{
    // Call steering rate controller (PID)
    const float steering_out = attitude_control.get_steering_out_rate(
        turn_rate,                    // Desired turn rate (rad/s)
        g2.motors.limit.steer_left,
        g2.motors.limit.steer_right,
        rover.G_Dt);
    
    // SEND TO MOTORS
    set_steering(steering_out * 4500.0f);  // Convert to -4500:+4500 range
}

void Mode::set_steering(float steering_value)
{
    // Store and send to motor library
    g2.motors.set_steering(steering_value);
}
```

**Steering Rate PID (AR_AttitudeControl):**
```cpp
float AR_AttitudeControl::get_steering_out_rate(float desired_rate_rads, ...) {
    // Get actual turn rate from gyro
    float actual_rate = ahrs.get_yaw_rate_earth();  // e.g., 0.8 rad/s
    
    // Calculate error
    float rate_error = desired_rate_rads - actual_rate;
    
    // PID calculation
    float p_out = _steer_rate_pid.get_p(rate_error);      // P term
    float i_out = _steer_rate_pid.get_i(rate_error, dt);  // I term  
    float d_out = _steer_rate_pid.get_d(rate_error, dt);  // D term
    float ff_out = _steer_rate_pid.get_ff();              // FF term
    
    float output = p_out + i_out + d_out + ff_out;
    
    // Constrain and return
    return constrain_float(output, -1.0, 1.0);  // -1 to +1 (normalized)
}
```

---

## **Layer 7: Motor Library** 🔧

### **File:** `libraries/AR_Motors/AP_MotorsUGV.cpp`

**Function:** `AP_MotorsUGV::output()`  
**Called from:** `Rover::set_servos()`  
**Frequency:** 400 Hz  
**Lines:** 322-358

**Function signature:**
```cpp
void AP_MotorsUGV::output(bool armed, float ground_speed, float dt)
```

**Parameters:**
- `armed` - Whether vehicle is armed (from `arming.is_armed()`)
- `ground_speed` - Current vehicle speed in m/s (from `attitude_control`)
- `dt` - Time delta since last call (typically 0.0025s at 400Hz)

**Uses member variables:**
- `_steering` - Set earlier by `set_steering()` calls from mode
- `_throttle` - Set earlier by `set_throttle()` calls from mode

**Code:**
```cpp
void AP_MotorsUGV::output(bool armed, float ground_speed, float dt)
{
    // Soft-arm override
    if (!hal.util->get_soft_armed()) {
        armed = false;
        _throttle = 0.0f;
    }

    // Clear limit flags
    limit.steer_left = limit.steer_right = false;
    limit.throttle_lower = limit.throttle_upper = false;

    // Sanity check parameters
    sanity_check_parameters();

    // Apply throttle slew rate limiting
    slew_limit_throttle(dt);

    // ⭐ DISPATCH TO APPROPRIATE OUTPUT METHOD
    
    // Regular steering/throttle (Ackermann)
    output_regular(armed, ground_speed, _steering, _throttle);

    // Skid steering (TANK) ⭐⭐⭐
    output_skid_steering(armed, _steering, _throttle, dt);

    // Omni frames
    output_omni(armed, _steering, _throttle, _lateral);

    // Sailboat
    output_sail();

    // ⭐ FINAL PWM CALCULATION AND OUTPUT
    auto &srv = AP::srv();
    SRV_Channels::calc_pwm();    // Convert scaled to PWM
    srv.cork();                   // Start batch
    SRV_Channels::output_ch_all(); // Write to hardware
    srv.push();                   // Execute batch
}
```

**Key points:**
- Calls ALL output methods (regular, skid, omni, sail)
- Each method checks if applicable (returns early if not)
- Only skid method executes for your tank

---

## **Layer 8: Skid-Steer Mixing** ⚙️

### **File:** `libraries/AR_Motors/AP_MotorsUGV.cpp`  
**Function:** `output_skid_steering()`  
**Lines:** 804-926

**This is THE CRITICAL function for your tank!**

```cpp
void AP_MotorsUGV::output_skid_steering(bool armed, float steering, 
                                         float throttle, float dt)
{
    // Exit if not skid-steering
    if (!have_skid_steering()) {
        return;  // ← Returns immediately if not tank
    }

    // Check armed status
    if (!armed) {
        // Set to TRIM or ZERO_PWM
        SRV_Channels::set_output_limit(SRV_Channel::k_throttleLeft, 
                                       SRV_Channel::Limit::TRIM);
        SRV_Channels::set_output_limit(SRV_Channel::k_throttleRight, 
                                       SRV_Channel::Limit::TRIM);
        return;
    }

    // ⭐ MIXING CALCULATION STARTS HERE

    // Normalize inputs
    float steering_scaled = steering / 4500.0f;  // -1 to +1
    float throttle_scaled = throttle * 0.01f;     // -1 to +1
    
    // Example values:
    // steering_scaled = -1500 / 4500 = -0.33
    // throttle_scaled = 50 * 0.01 = 0.50

    // Handle thrust asymmetry (forward vs reverse power)
    const float thrust_asymmetry = MAX(_thrust_asymmetry, 1.0);
    const float lower_throttle_limit = -1.0 / thrust_asymmetry;

    // Calculate available steering range (complex logic for edge cases)
    const float best_steering_throttle = (1.0 + lower_throttle_limit) * 0.5;
    float steering_range;
    if (throttle_scaled < best_steering_throttle) {
        steering_range = MAX(throttle_scaled,0.0) - lower_throttle_limit;
    } else {
        steering_range = 1 - best_steering_throttle;
    }

    // Constrain steering to available range
    // (prevents one motor going beyond 100% while other can't match)
    if (fabsf(steering_scaled) > steering_range) {
        steering_scaled = constrain_float(steering_scaled, 
                                          -steering_range, 
                                          steering_range);
        // Set limit flags
        limit.steer_left |= is_negative(steering_scaled);
        limit.steer_right |= is_positive(steering_scaled);
    }

    // Apply steering/throttle mix parameter
    // ⭐ THIS IS WHERE MOT_STR_THR_MIX IS USED
    steering_scaled *= _steering_throttle_mix;
    
    // Example: steering_scaled = -0.33 × 0.5 = -0.165

    // ⭐⭐⭐ THE CORE MIXING CALCULATION ⭐⭐⭐
    float motor_left  = throttle_scaled + steering_scaled;
    float motor_right = throttle_scaled - steering_scaled;
    
    // Example calculation:
    // motor_left  = 0.50 + (-0.165) = 0.335  (33.5%)
    // motor_right = 0.50 - (-0.165) = 0.665  (66.5%)

    // Apply asymmetry correction for reverse
    if (is_negative(motor_right)) {
        motor_right *= thrust_asymmetry;
    }
    if (is_negative(motor_left)) {
        motor_left *= thrust_asymmetry;
    }

    // ⭐ SEND TO SRV_CHANNELS
    // Convert back to percentage (-100 to +100)
    output_throttle(SRV_Channel::k_throttleLeft,  100.0f * motor_left,  dt);
    output_throttle(SRV_Channel::k_throttleRight, 100.0f * motor_right, dt);
    // Sends: ThrottleLeft=33.5%, ThrottleRight=66.5%
}
```

**Critical mixing formula:**
```
motor_left  = throttle + (steering × MOT_STR_THR_MIX)
motor_right = throttle - (steering × MOT_STR_THR_MIX)
```

---

## **Layer 9: SRV_Channels (Servo Output Management)** 📤

### **File:** `libraries/AR_Motors/AP_MotorsUGV.cpp`  
**Function:** `output_throttle()`  
**Lines:** ~500-600 (varies)

```cpp
void AP_MotorsUGV::output_throttle(SRV_Channel::Function function, 
                                    float throttle, float dt)
{
    // throttle = 33.5 (for left), 66.5 (for right)
    // function = k_throttleLeft or k_throttleRight

    // Apply throttle curve expo
    throttle = get_scaled_throttle(throttle);
    
    // Apply rate control if enabled (advanced feature)
    if (rate_control_enabled) {
        throttle = get_rate_controlled_throttle(function, throttle, dt);
    }
    
    // Apply reverse delay (prevents shock when changing direction)
    // Uses struct rev_delay_throttle/throttleLeft/throttleRight
    
    // ⭐ SET OUTPUT VALUE IN SRV_CHANNELS
    SRV_Channels::set_output_scaled(function, throttle);
    // This finds ALL servos with this function and sets their output
}
```

**SRV_Channels processing:**

```cpp
// File: libraries/SRV_Channel/SRV_Channels.cpp
void SRV_Channels::set_output_scaled(SRV_Channel::Function function, 
                                      float value)
{
    // Find all channels with this function
    for (uint8_t i=0; i<NUM_SERVO_CHANNELS; i++) {
        SRV_Channel *ch = &channels[i];
        
        if (ch->get_function() == function) {
            // Found a match!
            // function = k_throttleLeft (73)
            // Matches: SERVO1 and SERVO2 (both have function 73)
            
            ch->set_output_scaled(value);
            // value = 33.5 for both
        }
    }
}

// In individual SRV_Channel:
void SRV_Channel::set_output_scaled(float value)
{
    // value = 33.5 (percentage)
    // range = -100 to +100 (from set_angle(100))
    
    // Convert to normalized
    float normalized = value / 100.0f;  // 0.335
    
    // Store for later PWM calculation
    output_norm = normalized;
}
```

---

## **Layer 10: PWM Calculation** 📊

### **File:** `libraries/SRV_Channel/SRV_Channels.cpp`  
**Function:** `SRV_Channels::calc_pwm()`

```cpp
void SRV_Channels::calc_pwm()
{
    for (uint8_t i=0; i<NUM_SERVO_CHANNELS; i++) {
        channels[i].calc_pwm();
    }
}

// For each channel:
void SRV_Channel::calc_pwm()
{
    // Get normalized output (-1 to +1 or percentage)
    float value = output_norm;  // e.g., 0.335 for left motors
    
    // Get parameters
    uint16_t pwm_min = servo_min;   // 1000
    uint16_t pwm_max = servo_max;   // 2000
    uint16_t pwm_trim = servo_trim; // 1500
    
    // Convert normalized to PWM
    uint16_t pwm;
    if (value >= 0) {
        // Positive (forward)
        pwm = pwm_trim + (value * (pwm_max - pwm_trim));
        // pwm = 1500 + (0.335 × 500) = 1500 + 167.5 = 1668
    } else {
        // Negative (reverse)
        pwm = pwm_trim + (value * (pwm_trim - pwm_min));
    }
    
    // Apply REVERSED if needed
    if (reversed) {
        pwm = pwm_min + pwm_max - pwm;
    }
    
    // Store final PWM
    output_pwm = pwm;  // 1668 µs for left motors
}
```

**For your 4-motor tank:**
```
SERVO1 (Left-Front):  output_pwm = 1668 µs
SERVO2 (Left-Rear):   output_pwm = 1668 µs
SERVO3 (Right-Front): output_pwm = 1833 µs
SERVO4 (Right-Rear):  output_pwm = 1833 µs
```

---

## **Layer 11: Hardware PWM Output** ⚡

### **File:** `libraries/SRV_Channel/SRV_Channels.cpp`

```cpp
void SRV_Channels::output_ch_all()
{
    for (uint8_t i=0; i<NUM_SERVO_CHANNELS; i++) {
        SRV_Channel *ch = &channels[i];
        
        if (ch->is_enabled()) {
            // Get final PWM value
            uint16_t pwm = ch->get_output_pwm();
            
            // Send to hardware
            hal.rcout->write(i, pwm);
            // Writes to hardware timer registers
        }
    }
}
```

**HAL layer (hardware-specific):**

### **File:** `libraries/AP_HAL_ChibiOS/RCOutput.cpp`

```cpp
void RCOutput::write(uint8_t chan, uint16_t period_us)
{
    // chan = 0 (SERVO1), period_us = 1668
    
    // Convert to timer ticks
    uint32_t ticks = (period_us * _pwm_timer_freq) / 1000000;
    
    // Write to STM32 hardware timer compare register
    // This generates PWM waveform in hardware
    pwmEnableChannel(&PWMD3, 0, ticks);
    
    // Hardware now outputs:
    // 1668µs HIGH pulse, followed by LOW
    // Repeats every 20ms (50 Hz PWM frequency)
}
```

**Hardware generates:**
```
SERVO1 output (physical pin):
  ___1668µs___          ___1668µs___
__|          |________|          |________
  0         1668     20000      21668    (microseconds)
  
Repeats at 50 Hz
```

---

## **Layer 12: ESC & Motor** 🔌

### **ESC Processing:**

```
1. ESC measures pulse width
   ├── Detects rising edge
   ├── Starts timer
   ├── Detects falling edge
   ├── Stops timer
   └── Pulse width = 1668 µs

2. ESC decodes command
   ├── PWM range: 1000-2000 µs (standard)
   ├── 1668 µs in this range
   ├── Position = (1668 - 1000) / (2000 - 1000) = 0.668
   └── Command = 66.8% forward

3. ESC generates motor drive
   ├── PWM to motor: Duty cycle = 66.8%
   ├── Or 3-phase commutation at corresponding speed
   └── Motor spins at 66.8% power

4. Motor rotates
   └── Drives left track at 33.5% speed (calculated earlier)
```

---

## ⏱️ Timing & Execution

### **Main Loop Frequency:**

```
ArduRover main loop: 400 Hz (every 2.5 milliseconds)

Timing breakdown:
├── 0.0 ms:  Loop starts
├── 0.5 ms:  update_current_mode() → mode.update()
├── 1.0 ms:  Controllers run (speed/steering PID)
├── 1.5 ms:  set_servos() → motors.output()
├── 2.0 ms:  SRV_Channels PWM calculation
├── 2.3 ms:  HAL writes to hardware timers
└── 2.5 ms:  Loop ends, next iteration starts

Total latency: ~2.5-5ms from RC input to motor change
```

### **Task Frequencies:**

| **Task** | **Frequency** | **Function** |
|----------|--------------|-------------|
| `read_radio()` | 50 Hz | Read RC values from HAL |
| `ahrs_update()` | 400 Hz | Update attitude/heading |
| `update_current_mode()` | 400 Hz | Call current mode update |
| `set_servos()` | 400 Hz | Output to motors |
| `update_compass()` | 10 Hz | Read compass |
| `update_gps()` | 50 Hz | Read GPS |
| `read_rangefinders()` | 50 Hz | Read distance sensors |

---

## 🎯 Tank-Specific Example (Complete Trace)

### **Scenario: Manual Mode, Turn Right**

**INPUT: RC Stick Positions**
```
Throttle stick: 50% forward  → Ch3 = 1650 µs
Steering stick: 30% right    → Ch1 = 1575 µs
Mode switch:    Position 1   → Ch5 = 1165 µs (Manual)
```

---

**STEP 1: HAL RC Input**
```
File: AP_HAL_ChibiOS/RCInput.cpp
Function: Hardware interrupt

Captures:
├── rc_values[0] = 1575  (Ch1 - Steering)
├── rc_values[2] = 1650  (Ch3 - Throttle)
└── rc_values[4] = 1165  (Ch5 - Mode)
```

---

**STEP 2: RC Channel Reading**
```
File: Rover/radio.cpp:64
Function: Rover::read_radio()
Called: 50 Hz

Calls RC_Channels::read_input()
├── For each channel: calls RC_Channel::update()
├── ACTUAL CODE (RC_Channel.cpp:306):
│   radio_in = hal.rcin->read(ch_in);
│   ├── HAL has already parsed CRSF packet
│   ├── Returns µs equivalent value
│   └── ch_in = 0 for steering, ch_in = 2 for throttle
├── Results stored in radio_in:
│   ├── channel_steer->radio_in = 1575 µs
│   └── channel_throttle->radio_in = 1650 µs
```

---

**STEP 3: RC Normalization**
```
File: libraries/RC_Channel/RC_Channel.cpp:311-316
Function: RC_Channel::update() 
         (calls pwm_to_range() or pwm_to_angle())

✅ YES - THESE LINES EXECUTE WITH CRSF!

ACTUAL CODE (from RC_Channel.cpp:301-318):
    bool RC_Channel::update(void) {
        // Line 306: Read from HAL (CRSF already parsed into µs)
        radio_in = hal.rcin->read(ch_in);
        
        // Lines 311-316: ALWAYS execute (all protocols)
        if (type_in == ControlType::RANGE) {
            control_in = pwm_to_range();      // Throttle
        } else {
            control_in = pwm_to_angle();      // Steering
        }
        return true;
    }

🔑 KEY: Protocol abstraction happens BEFORE this point!

CRSF Data Flow:
┌────────────────────────────────────────────────┐
│ CRSF Receiver: Sends digital packet           │
│ [0xC8][len][type][CH1:992][CH3:1105][CRC]     │
│ ↓ (Serial @ 420000 baud)                      │
├────────────────────────────────────────────────┤
│ HAL: Parses CRSF packet                        │
│ Extracts: CH1 = 992 (10-bit digital value)    │
│ Converts to µs: 992 → 1575µs                  │
│ ↓                                              │
├────────────────────────────────────────────────┤
│ RC_Channel::update() Line 306                  │
│ radio_in = hal.rcin->read(ch_in); → 1575µs   │
│ ↓                                              │
├────────────────────────────────────────────────┤
│ RC_Channel::update() Lines 311-316             │
│ ✅ pwm_to_angle() EXECUTES for steering       │
│ ✅ pwm_to_range() EXECUTES for throttle       │
└────────────────────────────────────────────────┘

Steering (Ch1 - uses pwm_to_angle):
├── radio_in = 1575 µs (from HAL, converted from CRSF digital)
├── radio_min = 1000, radio_max = 2000, radio_trim = 1500
├── Above trim: (1575 - 1500) / (2000 - 1500) = 0.15
├── Scaled to angle: 0.15 × 4500 = +675
└── control_in = +675 (15% right of center)

Throttle (Ch3 - uses pwm_to_range):
├── radio_in = 1650 µs (from HAL, converted from CRSF digital)
├── radio_min = 1000, radio_max = 2000, radio_trim = 1500
├── Above trim: (1650 - 1500) / (2000 - 1500) = 0.30
├── Scaled to percentage: 0.30 × 100 = +30
└── control_in = +30 (30% throttle)

⚠️ CRITICAL INSIGHT:
   pwm_to_angle() and pwm_to_range() are NOT "PWM-specific"
   Despite the names, they work with µs values from ANY protocol!
   The "PWM" in the name is historical - they convert µs to normalized values.
```

---

**STEP 4: Mode Determination**
```
File: libraries/RC_Channel/RC_Channels.cpp
Function: read_mode_switch()
Called: 7 Hz

Mode channel (Ch5):
├── PWM = 1165 µs
├── Falls in range 910-1230
├── Maps to MODE1
├── MODE1 = 0 (Manual)
└── rover.set_mode(mode_manual)
```

---

**STEP 5: Mode Update**
```
File: Rover/Rover.cpp:516
Function: Rover::update_current_mode()
Called: 400 Hz

Calls: control_mode->update()
├── control_mode = &mode_manual
└── Calls: ModeManual::update()
```

---

**STEP 6: Manual Mode Processing**
```
File: Rover/mode_manual.cpp:9
Function: ModeManual::update()

get_pilot_desired_steering_and_throttle(desired_steering, desired_throttle);
├── Reads from RC channels
├── desired_steering = channel_steer->get_control_in() = +675
├── desired_throttle = channel_throttle->get_control_in() = +50
└── Returns: steering=+675, throttle=+50

Apply expo (optional curve):
├── desired_steering = 4500 × input_expo(675/4500, expo)
└── Result: steering ≈ +675 (if expo=0)

Send to motors:
├── g2.motors.set_throttle(50.0)
└── g2.motors.set_steering(675.0)
```

---

**STEP 7: Motors Library Storage**
```
File: libraries/AR_Motors/AP_MotorsUGV.cpp:235-242
Functions: set_steering() and set_throttle()

AP_MotorsUGV::set_steering(675.0, true):
├── _steering = 675.0
└── _scale_steering = true

AP_MotorsUGV::set_throttle(50.0):
├── Check armed
├── Constrain to MOT_THR_MAX
└── _throttle = 50.0

Stored member variables:
├── _steering = 675.0
└── _throttle = 50.0
```

---

**STEP 8: Motor Output Called**
```
File: Rover/Steering.cpp:6
Function: Rover::set_servos()
Called: 400 Hz

float speed = 0.0f;
g2.attitude_control.get_forward_speed(speed);  // Get current speed

g2.motors.output(arming.is_armed(), speed, G_Dt);
├── armed = true
├── speed = 1.85 m/s (from GPS/encoders)
└── dt = 0.0025 s (400 Hz)
```

---

**STEP 9: Motor Output Dispatch**
```
File: libraries/AR_Motors/AP_MotorsUGV.cpp:322
Function: AP_MotorsUGV::output()

Called with:
├── armed = true
├── ground_speed = 1.85
├── dt = 0.0025
├── _steering = 675.0 (stored)
└── _throttle = 50.0 (stored)

Executes:
├── output_regular() → Returns (not Ackermann)
├── output_skid_steering(true, 675, 50, 0.0025) → ⭐ EXECUTES
├── output_omni() → Returns (not omni)
└── output_sail() → Returns (not sailboat)
```

---

**STEP 10: Skid-Steer Mixing**
```
File: libraries/AR_Motors/AP_MotorsUGV.cpp:804
Function: output_skid_steering(true, 675.0, 50.0, 0.0025)

Normalize:
├── steering_scaled = 675 / 4500 = 0.15
└── throttle_scaled = 50 × 0.01 = 0.50

Apply MOT_STR_THR_MIX:
├── _steering_throttle_mix = 0.5
└── steering_scaled = 0.15 × 0.5 = 0.075

Mix calculation:
├── motor_left  = 0.50 + 0.075 = 0.575 (57.5%)
└── motor_right = 0.50 - 0.075 = 0.425 (42.5%)

Send to outputs:
├── output_throttle(k_throttleLeft, 57.5, dt)
└── output_throttle(k_throttleRight, 42.5, dt)
```

---

**STEP 11: SRV_Channels Update**
```
File: libraries/SRV_Channel/SRV_Channels.cpp
Function: set_output_scaled(k_throttleLeft, 57.5)

Finds channels with function 73:
├── SERVO1_FUNCTION = 73 ✓
└── SERVO2_FUNCTION = 73 ✓

For each:
└── set_output_scaled(57.5)
    └── output_norm = 0.575

Same for k_throttleRight (finds SERVO3 & SERVO4)
└── output_norm = 0.425
```

---

**STEP 12: PWM Conversion**
```
File: libraries/SRV_Channel/SRV_Channel.cpp
Function: SRV_Channel::calc_pwm()
Called: From SRV_Channels::calc_pwm()

For SERVO1 & SERVO2 (Left motors):
├── output_norm = 0.575
├── servo_min = 1000, servo_max = 2000, servo_trim = 1500
├── value positive, so:
├── pwm = 1500 + (0.575 × (2000 - 1500))
├── pwm = 1500 + (0.575 × 500)
├── pwm = 1500 + 287.5
└── output_pwm = 1788 µs ⭐

For SERVO3 & SERVO4 (Right motors):
├── output_norm = 0.425
├── pwm = 1500 + (0.425 × 500)
├── pwm = 1500 + 212.5
└── output_pwm = 1713 µs ⭐
```

---

**STEP 13: Hardware PWM Write**
```
File: libraries/SRV_Channel/SRV_Channels.cpp
Function: output_ch_all()

For each servo:
hal.rcout->write(channel_num, output_pwm);

Writes:
├── Channel 0 (SERVO1): 1788 µs
├── Channel 1 (SERVO2): 1788 µs
├── Channel 2 (SERVO3): 1713 µs
└── Channel 3 (SERVO4): 1713 µs
```

---

**STEP 14: Hardware Timer Configuration**
```
File: libraries/AP_HAL_ChibiOS/RCOutput.cpp
Function: RCOutput::write(channel, period_us)

For each channel:
├── Calculate timer ticks from microseconds
├── ticks = (period_us × timer_frequency) / 1000000
├── Configure STM32 timer compare register
└── Hardware generates PWM waveform

Physical output on pin:
├── HIGH for period_us microseconds
├── LOW for remaining time in 20ms frame
└── Repeats at 50 Hz
```

---

**STEP 15: ESC Response**
```
ESC 1 & 2 (Left motors):
├── Measure pulse: 1788 µs
├── Range: 1000-2000 µs
├── Position: (1788-1000)/(2000-1000) = 0.788
├── Command: 78.8% forward ⭐
└── Motors spin at 78.8% power

ESC 3 & 4 (Right motors):
├── Measure pulse: 1713 µs
├── Position: (1713-1000)/1000 = 0.713
├── Command: 71.3% forward ⭐
└── Motors spin at 71.3% power

Physical Result:
├── Left tracks: 78.8% power (faster)
├── Right tracks: 71.3% power (slower)
└── Tank turns RIGHT (as commanded!)
```

---

## 📊 Complete Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    SIGNAL FLOW SUMMARY                       │
└─────────────────────────────────────────────────────────────┘

Layer  File/Location             Function                    Data Type          Value Example
═════  ═════════════════════════ ═════════════════════════  ═══════════════    ══════════════
  1    RC Transmitter            Pilot input                 Stick position     30% right
       ↓
  2    RC Receiver               Encode PWM                  Microseconds       1575 µs
       ↓
  3    AP_HAL_ChibiOS/           Hardware ISR                Raw µs             1575 µs
       RCInput.cpp                
       ↓
  4    RC_Channel/               Normalize                   Scaled value       +675
       RC_Channel.cpp             
       ↓
  5    Rover/radio.cpp           Read RC                     Stored in channel  +675
       read_radio()
       ↓
  6    Rover/mode_manual.cpp     Get pilot input             Float              +675
       ModeManual::update()
       ↓
  7    AR_Motors/                Store values                Member variable    _steering=675
       AP_MotorsUGV.cpp                                                         _throttle=50
       set_steering()/set_throttle()
       ↓
  8    AR_Motors/                Calculate mix               Float              motor_left=0.575
       AP_MotorsUGV.cpp           steering + throttle                           motor_right=0.425
       output_skid_steering()     
       ↓
  9    SRV_Channel/              Set scaled output           Normalized         0.575, 0.425
       SRV_Channels.cpp           Find matching functions                       (for L/R)
       set_output_scaled()
       ↓
 10    SRV_Channel/              Convert to PWM              Microseconds       1788, 1713
       SRV_Channel.cpp
       calc_pwm()
       ↓
 11    SRV_Channel/              Output all channels         PWM µs             1788, 1713
       SRV_Channels.cpp
       output_ch_all()
       ↓
 12    AP_HAL_ChibiOS/           Write hardware timer        Timer ticks        Hardware specific
       RCOutput.cpp
       write()
       ↓
 13    STM32 Hardware            Generate PWM waveform       Electrical signal  5V pulses
       Timer peripheral
       ↓
 14    ESC                       Decode PWM                  Motor command      78.8%, 71.3%
       ↓
 15    Motor                     Spin!                       Physical rotation  Different speeds
       ↓
       Tank turns right! ✓
```

---

## 💻 Key Code Locations

### **Core Files (In Order of Execution):**

```
1. RC Input Hardware:
   └── libraries/AP_HAL_ChibiOS/RCInput.cpp
       └── _timer_tick() - ISR

2. RC Reading:
   └── Rover/radio.cpp
       └── read_radio() - Called 50 Hz

3. RC Channel Processing:
   └── libraries/RC_Channel/RC_Channel.cpp
       ├── set_pwm() - Store raw
       └── get_control_in() - Get normalized

4. Scheduler:
   └── Rover/Rover.cpp:69
       └── scheduler_tasks[] - Task table

5. Mode Update:
   └── Rover/Rover.cpp:516
       └── update_current_mode()

6. Manual Mode:
   └── Rover/mode_manual.cpp:9
       └── ModeManual::update()

7. Auto Mode (for comparison):
   └── Rover/mode_auto.cpp
       └── ModeAuto::update()

8. Controllers:
   └── libraries/APM_Control/AR_AttitudeControl.cpp
       ├── get_throttle_out_speed() - Speed PID
       └── get_steering_out_rate() - Steering PID

9. Motor Library:
   └── libraries/AR_Motors/AP_MotorsUGV.cpp
       ├── output() - Main output function
       ├── output_skid_steering() - Tank mixing ⭐
       ├── output_regular() - Ackermann
       └── output_omni() - Omni vehicles

10. Servo Management:
    └── libraries/SRV_Channel/SRV_Channels.cpp
        ├── set_output_scaled() - Set value
        ├── calc_pwm() - Convert to PWM
        └── output_ch_all() - Write to hardware

11. Hardware Output:
    └── libraries/AP_HAL_ChibiOS/RCOutput.cpp
        └── write() - Hardware timer control
```

---

## 🔬 Critical Functions Explained

### **1. Mode::get_pilot_desired_steering_and_throttle()**

**File:** `Rover/mode.cpp:60-90`

```cpp
void Mode::get_pilot_desired_steering_and_throttle(float &steering_out, 
                                                     float &throttle_out) const
{
    // Get RC input directly
    get_pilot_input(steering_out, throttle_out);
    // steering_out = channel_steer->get_control_in()    // +675
    // throttle_out = channel_throttle->get_control_in() // +50
}

void Mode::get_pilot_input(float &steering_out, float &throttle_out) const
{
    // Get from channel object (already normalized)
    steering_out = channel_steer->get_control_in();
    throttle_out = channel_throttle->get_control_in();
    
    // Apply deadzone
    if (fabsf(steering_out) < channel_steer->get_dead_zone()) {
        steering_out = 0;
    }
    if (fabsf(throttle_out) < channel_throttle->get_dead_zone()) {
        throttle_out = 0;
    }
}
```

---

### **2. AP_MotorsUGV::have_skid_steering()**

**File:** `libraries/AR_Motors/AP_MotorsUGV.cpp`

```cpp
bool AP_MotorsUGV::have_skid_steering() const
{
    // Returns true if any ThrottleLeft/Right configured
    return SRV_Channels::function_assigned(SRV_Channel::k_throttleLeft) ||
           SRV_Channels::function_assigned(SRV_Channel::k_throttleRight);
}
```

**This function determines which output method to use!**

---

### **3. AR_AttitudeControl::get_forward_speed() - SENSOR DATA**

**File:** `libraries/APM_Control/AR_AttitudeControl.cpp:990-1010`

⭐ **IMPORTANT:** This is NOT part of the control command flow!  
This function READS the actual vehicle speed from sensors.

```cpp
bool AR_AttitudeControl::get_forward_speed(float &speed) const
{
    Vector3f velocity;
    const AP_AHRS &_ahrs = AP::ahrs();
    
    // Try to get velocity from AHRS (EKF)
    if (!_ahrs.get_velocity_NED(velocity)) {
        // Fallback to GPS if AHRS unavailable
        if (AP::gps().status() >= AP_GPS::GPS_OK_FIX_3D) {
            // Check if moving forward or backward
            if (abs(wrap_180_cd(_ahrs.yaw_sensor - AP::gps().ground_course_cd())) <= 9000) {
                speed = AP::gps().ground_speed();    // Forward
            } else {
                speed = -AP::gps().ground_speed();   // Backward
            }
            return true;
        } else {
            return false;  // No GPS fix
        }
    }
    
    // Calculate forward speed by projecting into body frame
    speed = velocity.x * _ahrs.cos_yaw() + velocity.y * _ahrs.sin_yaw();
    return true;
}
```

**Data sources (in priority order):**

1. **AHRS/EKF (Primary):**
   - Extended Kalman Filter fuses multiple sensors:
     - IMU (Inertial Measurement Unit) - accelerometers & gyros
     - GPS position & velocity
     - Compass (magnetometer)
     - Barometer (for altitude)
     - Optional: wheel encoders, optical flow
   - Produces best estimate of vehicle velocity in NED (North-East-Down)
   - Projects into vehicle body frame to get forward speed

2. **GPS Ground Speed (Fallback):**
   - Direct GPS velocity measurement
   - Compares vehicle heading vs. GPS course to determine forward/reverse
   - Less accurate during turns
   - Requires 3D GPS fix

**Why this is needed in output():**
- Some vehicles use speed scaling (reduce steering at high speed)
- Balance bots need speed for pitch control
- Advanced mixing algorithms consider current velocity
- But it's FEEDBACK, not a command!

**Control vs. Feedback:**
```
CONTROL PATH (Command):
RC → Mode → set_steering/throttle → output → Motors
     ↓
  Command what you WANT

FEEDBACK PATH (Measurement):
GPS/IMU → AHRS/EKF → get_forward_speed() → output
     ↓
  Measure what you HAVE
```

---

### **4. output_skid_steering() - THE KEY MIXING**

**File:** `libraries/AR_Motors/AP_MotorsUGV.cpp:804-926`

**Detailed code with comments:**

```cpp
void AP_MotorsUGV::output_skid_steering(bool armed, float steering, 
                                         float throttle, float dt)
{
    // Exit if not skid-steering
    if (!have_skid_steering()) {
        return;
    }

    // Handle disarmed case
    if (!armed) {
        SRV_Channels::set_output_limit(SRV_Channel::k_throttleLeft, 
                                       SRV_Channel::Limit::TRIM);
        SRV_Channels::set_output_limit(SRV_Channel::k_throttleRight, 
                                       SRV_Channel::Limit::TRIM);
        return;
    }

    // ═══════════════════════════════════════════
    //        CORE MIXING ALGORITHM
    // ═══════════════════════════════════════════

    // Step 1: Normalize inputs to -1.0 to +1.0
    float steering_scaled = steering / 4500.0f;
    float throttle_scaled = throttle * 0.01f;
    // steering_scaled = 675 / 4500 = 0.15
    // throttle_scaled = 50 / 100 = 0.50

    // Step 2: Handle thrust asymmetry (forward vs reverse power)
    const float thrust_asymmetry = MAX(_thrust_asymmetry, 1.0);
    const float lower_throttle_limit = -1.0 / thrust_asymmetry;

    // Step 3: Calculate available steering range
    // (Complex logic to prevent saturation)
    const float best_steering_throttle = (1.0 + lower_throttle_limit) * 0.5;
    float steering_range;
    if (throttle_scaled < best_steering_throttle) {
        steering_range = MAX(throttle_scaled,0.0) - lower_throttle_limit;
    } else {
        steering_range = 1 - best_steering_throttle;
    }

    // Step 4: Constrain steering to available range
    float steering_scaled_orig = steering_scaled;
    if (fabsf(steering_scaled) > steering_range) {
        steering_scaled = constrain_float(steering_scaled, 
                                          -steering_range, 
                                          steering_range);
    }

    // Step 5: Apply steering/throttle mix
    // ⭐ MOT_STR_THR_MIX parameter used here
    steering_scaled *= _steering_throttle_mix;
    // steering_scaled = 0.15 × 0.5 = 0.075

    // Step 6: Check if we had to reduce commands (set limit flags)
    if (fabsf(steering_scaled) < fabsf(steering_scaled_orig)) {
        limit.steer_left |= is_negative(steering_scaled_orig);
        limit.steer_right |= is_positive(steering_scaled_orig);
    }

    // ═══════════════════════════════════════════
    //   CRITICAL MIXING FORMULA (2 LINES!)
    // ═══════════════════════════════════════════
    
    float motor_left  = throttle_scaled + steering_scaled;
    float motor_right = throttle_scaled - steering_scaled;
    
    // motor_left  = 0.50 + 0.075 = 0.575
    // motor_right = 0.50 - 0.075 = 0.425

    // Step 7: Apply asymmetry for reverse thrust
    if (is_negative(motor_right)) {
        motor_right *= thrust_asymmetry;
    }
    if (is_negative(motor_left)) {
        motor_left *= thrust_asymmetry;
    }

    // Step 8: Send to output functions
    // Convert back to percentage (-100 to +100)
    output_throttle(SRV_Channel::k_throttleLeft,  100.0f * motor_left,  dt);
    output_throttle(SRV_Channel::k_throttleRight, 100.0f * motor_right, dt);
    // Sends: left=57.5, right=42.5
}
```

**This is the HEART of tank control!**

---

### **4. SRV_Channels::set_output_scaled()**

**File:** `libraries/SRV_Channel/SRV_Channels.cpp`

```cpp
void SRV_Channels::set_output_scaled(SRV_Channel::Function function, 
                                      int16_t value)
{
    // Loop through all 16 servo channels
    for (uint8_t i=0; i<NUM_SERVO_CHANNELS; i++) {
        SRV_Channel *ch = &channels[i];
        
        // Check if this channel has the requested function
        if ((SRV_Channel::Function)ch->function.get() == function) {
            // MATCH! Set output
            ch->set_output_scaled(value);
        }
    }
}
```

**For your tank:**
```
Called: set_output_scaled(k_throttleLeft, 57.5)

Searches:
├── ch[0]: function = 73 ✓ MATCH → set_output_scaled(57.5)
├── ch[1]: function = 73 ✓ MATCH → set_output_scaled(57.5)
├── ch[2]: function = 74 ✗ Skip
└── ch[3]: function = 74 ✗ Skip

Result: SERVO1 and SERVO2 both get 57.5%
```

---

### **5. SRV_Channel::calc_pwm()**

**File:** `libraries/SRV_Channel/SRV_Channel.cpp`

```cpp
void SRV_Channel::calc_pwm()
{
    // Get current scaled output
    float value = get_output_norm();  // Already normalized by set_output_scaled
    
    // Get min/max/trim parameters
    const uint16_t pwm_min = servo_min;
    const uint16_t pwm_max = servo_max;
    const uint16_t pwm_trim = servo_trim;
    
    uint16_t pwm;
    
    // Convert normalized (-1 to +1) to PWM (1000-2000)
    if (value >= 0.0f) {
        // Positive direction (forward for motors)
        pwm = pwm_trim + value * (pwm_max - pwm_trim);
    } else {
        // Negative direction (reverse for motors)
        pwm = pwm_trim + value * (pwm_trim - pwm_min);
    }
    
    // Apply REVERSED parameter
    if (reversed) {
        pwm = pwm_min + pwm_max - pwm;
    }
    
    // Apply MIN/MAX constraints
    pwm = constrain_uint16(pwm, pwm_min, pwm_max);
    
    // Store final PWM
    output_pwm = pwm;
}
```

---

## 🎯 Complete Example: Tank Turning

### **Inputs:**

```
RC Input:
├── Throttle: 1650 µs (50% forward)
└── Steering: 1575 µs (15% right)

Configuration:
├── SERVO1_FUNCTION = 73 (ThrottleLeft)
├── SERVO2_FUNCTION = 73 (ThrottleLeft)
├── SERVO3_FUNCTION = 74 (ThrottleRight)
├── SERVO4_FUNCTION = 74 (ThrottleRight)
├── MOT_STR_THR_MIX = 0.5
└── All SERVOn_MIN=1000, MAX=2000, TRIM=1500
```

### **Processing:**

```
1. RC → Normalized:
   throttle_scaled = 0.50
   steering_scaled = 0.15

2. Apply mix:
   steering_scaled = 0.15 × 0.5 = 0.075

3. Mixing:
   motor_left  = 0.50 + 0.075 = 0.575
   motor_right = 0.50 - 0.075 = 0.425

4. To percentage:
   left  = 57.5%
   right = 42.5%

5. To PWM:
   SERVO1 & 2 = 1500 + (0.575 × 500) = 1788 µs
   SERVO3 & 4 = 1500 + (0.425 × 500) = 1713 µs

6. Hardware output:
   M1 (Left-Front):  1788 µs → ESC → 78.8% power
   M2 (Left-Rear):   1788 µs → ESC → 78.8% power
   M3 (Right-Front): 1713 µs → ESC → 71.3% power
   M4 (Right-Rear):  1713 µs → ESC → 71.3% power

7. Physical result:
   Left faster than right → Turn right ✓
```

---

## 📝 Data Structures

### **Key Classes:**

```cpp
// Main vehicle class
class Rover : public AP_Vehicle {
    Mode *control_mode;               // Current mode
    RC_Channel *channel_steer;        // Steering RC channel
    RC_Channel *channel_throttle;     // Throttle RC channel
    AP_MotorsUGV motors;              // Motor control
    // ...
};

// Motor control class
class AP_MotorsUGV {
    float _steering;     // -4500 to +4500
    float _throttle;     // -100 to +100
    float _lateral;      // -100 to +100
    float _steering_throttle_mix;  // MOT_STR_THR_MIX
    // ...
};

// Servo channel class
class SRV_Channel {
    AP_Int8 function;    // SERVO_FUNCTION (e.g., 73)
    AP_Int16 servo_min;  // SERVO_MIN (e.g., 1000)
    AP_Int16 servo_max;  // SERVO_MAX (e.g., 2000)
    AP_Int16 servo_trim; // SERVO_TRIM (e.g., 1500)
    AP_Int8 reversed;    // SERVO_REVERSED
    uint16_t output_pwm; // Final PWM value
    // ...
};
```

---

## ⚡ Performance & Timing

### **Loop Rates:**

```
Main loop: 400 Hz (2.5 ms period)
├── Sensor reads: 10-400 Hz
├── Mode update: 400 Hz
├── Motor output: 400 Hz
└── PWM output: 400 Hz (but hardware at 50 Hz)

RC Input: 50 Hz (20 ms period)
├── Enough for human reaction times
└── Smoothed by running at lower rate

PWM Output: 50 Hz (standard for servos/ESCs)
├── 20 ms frame
└── Industry standard
```

### **Latency Breakdown:**

```
RC stick moved:                          T=0 ms
├── RC receiver processes:               +2-5 ms (receiver lag)
├── HAL captures (next 50Hz cycle):      +0-20 ms (max)
├── Scheduler processes:                  +2.5 ms (one loop)
├── Mode calculates:                      +0.5 ms
├── Motors mix:                           +0.5 ms
├── PWM output:                           +1 ms
└── ESC responds:                         +1-3 ms

Total latency: 7-32 ms (average ~15ms)
```

**Human perception:** >50ms  
**System response:** <20ms typical  
**Result:** Feels instant! ✓

---

## 🎓 Understanding the Abstraction

### **Why This Architecture?**

```
Benefits of layered architecture:

1. Hardware Independence:
   ├── Same code runs on F4, F7, H7, Linux
   ├── HAL hides hardware differences
   └── Easy to port to new platforms

2. Vehicle Type Independence:
   ├── Same mode code for car, tank, omni, boat
   ├── AR_Motors handles vehicle-specific mixing
   └── Easy to add new vehicle types

3. Testability:
   ├── Each layer can be tested independently
   ├── SITL simulation uses same layers
   └── No hardware needed for development

4. Maintainability:
   ├── Clear separation of concerns
   ├── Easy to find and fix bugs
   └── Well-documented interfaces
```

---

## 🔍 Tracing a Signal (Debugging)

### **How to Debug Signal Flow:**

```cpp
// Add debug prints at each layer:

// Layer 3: RC input
void Rover::read_radio() {
    GCS_SEND_TEXT(MAV_SEVERITY_INFO, "RC: Ch1=%u Ch3=%u", 
                  channel_steer->get_radio_in(),
                  channel_throttle->get_radio_in());
}

// Layer 6: Mode processing
void ModeManual::update() {
    GCS_SEND_TEXT(MAV_SEVERITY_INFO, "Manual: steer=%.0f thr=%.0f", 
                  desired_steering, desired_throttle);
}

// Layer 8: Mixing
void AP_MotorsUGV::output_skid_steering(...) {
    GCS_SEND_TEXT(MAV_SEVERITY_INFO, "Mix: L=%.1f%% R=%.1f%%", 
                  motor_left*100, motor_right*100);
}

// Layer 10: PWM output
void SRV_Channel::calc_pwm() {
    GCS_SEND_TEXT(MAV_SEVERITY_INFO, "PWM: Ch%u = %u us", 
                  ch_num, output_pwm);
}
```

**Then watch in Mission Planner Messages tab!**

---

## 📊 Summary Flow Chart

```
╔═══════════════════════════════════════════════════════════╗
║              COMPLETE SIGNAL FLOW                          ║
╚═══════════════════════════════════════════════════════════╝

RC Stick Movement
    │ (Physical)
    ↓
RC Transmitter Encoding
    │ (Radio signal)
    ↓
RC Receiver Decoding
    │ (CRSF packet or PPM/SBUS: 1575 µs)
    ↓
[HAL Layer] RCInput.cpp
    │ (CRSF: UART ISR, PPM/SBUS: Timer ISR)
    │ (Normalized to µs values)
    ↓
[RC_Channels] RC_Channel.cpp
    │ (Calibration: normalized +675)
    ↓
[Scheduler] Rover.cpp
    │ (Task dispatch: 400 Hz)
    ↓
[Mode] mode_manual.cpp
    │ (Get pilot input: steer=675, thr=50)
    ↓
[Motors] AP_MotorsUGV.cpp
    │ (Store: _steering=675, _throttle=50)
    ↓
[Mixing] output_skid_steering()
    │ (Calculate: L=57.5%, R=42.5%)
    ↓
[SRV_Channels] SRV_Channels.cpp
    │ (Find function 73 & 74)
    ↓
[PWM Calc] SRV_Channel.cpp
    │ (Convert: 1788µs, 1713µs)
    ↓
[HAL Output] RCOutput.cpp
    │ (Hardware timers)
    ↓
ESC Processing
    │ (Decode PWM)
    ↓
Motor Power
    │ (Electrical current)
    ↓
Physical Movement
    │ (Rotation)
    ↓
Tank Turns Right! ✓
```

---

## ✅ Key Takeaways

1. **Layered architecture** - Each layer has specific responsibility
2. **400 Hz main loop** - Fast enough for responsive control
3. **Mode abstraction** - Same interface for all modes
4. **Motor abstraction** - AR_Motors handles vehicle-specific mixing
5. **SRV_Channels** - Function-based output (multiple channels, same function)
6. **HAL isolation** - Hardware differences hidden from application code

**The complete path:** RC stick → 15 layers → Motor spinning in ~10-15ms! 🚀

---

**Document complete!** This shows the exact code path with file names, line numbers, and function calls. Use this for debugging or understanding the system deeply! 🎯

