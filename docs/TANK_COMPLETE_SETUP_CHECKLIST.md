# ArduRover Tank Setup - Complete Checklist
## Everything You Need to Configure Your 4-Motor Tank

**Document:** Complete setup guide for tank-style rover  
**Hardware:** SpeedyBee F405 V3 (or similar)  
**Configuration:** 4 motors (2 left + 2 right), skid-steering, CRSF RC input  
**Date:** 2025-10-10

---

## 📋 TABLE OF CONTENTS

1. [Quick Overview](#quick-overview)
2. [Hardware Requirements](#hardware-requirements)
3. [Essential Configuration Steps](#essential-configuration-steps)
4. [Calibration Procedures](#calibration-procedures)
5. [Testing Sequence](#testing-sequence)
6. [Safety Configuration](#safety-configuration)
7. [Optional Enhancements](#optional-enhancements)
8. [Troubleshooting](#troubleshooting)

---

## 🎯 Quick Overview

### What You're Building:
```
4-Motor Tank Configuration:
├── 2 Left motors (synchronized)
├── 2 Right motors (synchronized)
├── Skid-steering (differential control)
├── CRSF RC input (TBS Crossfire / ExpressLRS)
└── ArduRover firmware on SpeedyBee F405 V3
```

### Time Required:
- **Basic setup:** 1-2 hours
- **Full autonomous setup:** 3-4 hours
- **Tuning:** Ongoing as needed

---

## 🔌 Hardware Requirements

### Minimum Required:

| **Component** | **Quantity** | **Notes** |
|--------------|-------------|-----------|
| Flight Controller | 1 | SpeedyBee F405 V3 or compatible |
| ESCs | 4 | One per motor, bidirectional |
| Motors | 4 | 2 left, 2 right |
| Battery | 1 | Appropriate voltage (2S-6S) |
| Power Distribution | 1 | PDB or direct wiring |
| RC Transmitter | 1 | CRSF compatible (e.g., TBS Tango 2, Radiomaster) |
| RC Receiver | 1 | CRSF receiver (TBS Nano RX, ExpressLRS) |
| USB Cable | 1 | For configuration |

### For Autonomous Operation (Additional):

| **Component** | **Quantity** | **Notes** |
|--------------|-------------|-----------|
| GPS Module | 1 | With integrated compass recommended |
| Telemetry Radio | 1 pair | 900MHz or 433MHz (optional) |

---

## ⚙️ Essential Configuration Steps

### ✅ **Step 1: Frame Configuration**

**Parameter:** `FRAME_CLASS`  
**Value:** `1` (Rover)

```
Mission Planner → CONFIG → Full Parameter List
FRAME_CLASS = 1
Write Parameters
```

---

### ✅ **Step 2: Motor Output Configuration** ⭐ CRITICAL

**Configure all 4 motor outputs:**

```
Mission Planner → CONFIG → Full Parameter List

Left Side Motors (both get function 73):
┌────────────────────────────────────┐
│ SERVO1_FUNCTION = 73               │  # Left-Front motor
│ SERVO2_FUNCTION = 73               │  # Left-Rear motor
└────────────────────────────────────┘

Right Side Motors (both get function 74):
┌────────────────────────────────────┐
│ SERVO3_FUNCTION = 74               │  # Right-Front motor
│ SERVO4_FUNCTION = 74               │  # Right-Rear motor
└────────────────────────────────────┘

Write Parameters
```

**Why this works:**
- Both outputs with function 73 receive identical signals (left side)
- Both outputs with function 74 receive identical signals (right side)
- ArduRover handles differential mixing automatically

---

### ✅ **Step 3: Servo Output Ranges**

**Set PWM ranges for all servos:**

```
For each SERVO (1-4):
┌────────────────────────────────────┐
│ SERVOn_MIN = 1000                  │
│ SERVOn_MAX = 2000                  │
│ SERVOn_TRIM = 1500                 │
│ SERVOn_REVERSED = 0                │ # Change to 1 if needed
└────────────────────────────────────┘

Write Parameters
```

---

### ✅ **Step 4: Motor/PWM Type Configuration**

**Configure motor output type:**

```
┌────────────────────────────────────────────────────┐
│ MOT_PWM_TYPE = 0          # 0=Normal PWM           │
│                           # 5-8 for DShot          │
│                                                     │
│ MOT_THR_MIN = 0           # Minimum throttle %     │
│ MOT_THR_MAX = 100         # Maximum throttle %     │
│ MOT_SLEWRATE = 100        # Throttle change rate   │
│ MOT_THST_EXPO = 0.0       # Linear throttle curve  │
│ MOT_STR_THR_MIX = 0.5     # Steering/throttle mix  │
└────────────────────────────────────────────────────┘

Write Parameters
```

**Common MOT_PWM_TYPE values:**
- `0` = Normal PWM (most ESCs) ⭐ Start here
- `3` = Brushed with relay
- `4` = Brushed bipolar
- `5-8` = DShot (if ESCs support it)

---

### ✅ **Step 5: Steering Control Configuration**

**Set steering parameters:**

```
┌────────────────────────────────────────────────────┐
│ ATC_STR_RAT_P = 0.2       # Steering P gain        │
│ ATC_STR_RAT_I = 0.2       # Steering I gain        │
│ ATC_STR_RAT_D = 0.0       # Steering D gain        │
│ ATC_STR_RAT_MAX = 180     # Max turn rate (deg/s)  │
│ ATC_STR_RAT_FILT = 10     # Filter frequency       │
│ ATC_STR_ANG_P = 2.0       # Heading P gain         │
└────────────────────────────────────────────────────┘

Write Parameters
```

**Tuning guide:**
- Start with defaults above
- If turns too slow: Increase `ATC_STR_RAT_P` to 0.3
- If oscillates: Decrease `ATC_STR_RAT_P` to 0.1
- For pivot turns: Set `ATC_STR_RAT_MAX` to 360

---

### ✅ **Step 6: Speed Control Configuration**

**Set speed parameters:**

```
┌────────────────────────────────────────────────────┐
│ CRUISE_SPEED = 2.0        # Target speed (m/s)     │
│ CRUISE_THROTTLE = 50      # Throttle for cruise %  │
│ SPEED_MAX = 0             # Max speed (0=auto)     │
│                                                     │
│ ATC_SPEED_P = 0.2         # Speed P gain           │
│ ATC_SPEED_I = 0.2         # Speed I gain           │
│ ATC_SPEED_D = 0.0         # Speed D gain           │
└────────────────────────────────────────────────────┘

Write Parameters
```

---

### ✅ **Step 7: Mode Configuration**

**Configure flight modes:**

```
┌────────────────────────────────────────────────────┐
│ MODE_CH = 5               # RC channel for modes   │
│                                                     │
│ MODE1 = 0                 # Manual                 │
│ MODE2 = 4                 # Hold (emergency stop)  │
│ MODE3 = 3                 # Steering               │
│ MODE4 = 10                # Auto (missions)        │
│ MODE5 = 11                # RTL (return home)      │
│ MODE6 = 15                # Guided (GCS control)   │
└────────────────────────────────────────────────────┘

Write Parameters
```

---

### ✅ **Step 8: RC Input Configuration (CRSF)** ⭐ IMPORTANT

**Configure CRSF protocol on UART2:**

```
┌────────────────────────────────────────────────────────┐
│ SERIAL2_PROTOCOL = 23        # 23 = RCIN (CRSF)       │
│ SERIAL2_BAUD = 115           # 115200 (auto-adjusted) │
│ SERIAL2_OPTIONS = 0          # Normal (no inversion)  │
│                                                         │
│ RSSI_TYPE = 3                # Get RSSI from CRSF     │
│ RC_OPTIONS = 256             # Enable CRSF telemetry  │
│                              # (optional but useful)   │
└────────────────────────────────────────────────────────┘

Write Parameters and REBOOT flight controller
```

**Physical wiring for CRSF (SpeedyBee F405 V3):**

```
CRSF Receiver connections:
┌───────────────────────────────────────────────────┐
│ RX (receiver) ──→ T2 (FC UART2 TX)               │
│ TX (receiver) ──→ R2 (FC UART2 RX)               │
│ 5V            ──→ 5V                              │
│ GND           ──→ GND                             │
└───────────────────────────────────────────────────┘

⭐ IMPORTANT: Connect BOTH RX and TX for bidirectional CRSF!
   (TX enables telemetry back to your transmitter)
```

**Set RC channel mapping:**

```
┌────────────────────────────────────────────────────┐
│ RCMAP_ROLL = 1            # Ch1 = Steering         │
│ RCMAP_THROTTLE = 3        # Ch3 = Throttle         │
│ RCMAP_PITCH = 2           # (not used in Rover)    │
│ RCMAP_YAW = 4             # (not used in Rover)    │
└────────────────────────────────────────────────────┘

Write Parameters
```

**Why CRSF is better than PPM:**
- ✅ Lower latency (5-10ms vs 20ms)
- ✅ Bidirectional (telemetry to transmitter)
- ✅ More channels (16+ vs 8-12)
- ✅ Built-in RSSI and link quality
- ✅ More reliable digital protocol

---

### ✅ **Step 9: Failsafe Configuration** ⭐ SAFETY

**Configure failsafe behavior:**

```
┌────────────────────────────────────────────────────┐
│ FS_ACTION = 2             # Hold on failsafe       │
│ FS_TIMEOUT = 1.5          # 1.5 second delay       │
│ FS_THR_ENABLE = 1         # RC failsafe enabled    │
│ FS_THR_VALUE = 910        # Trigger PWM            │
│ FS_GCS_ENABLE = 0         # GCS failsafe (initial) │
│                                                     │
│ FS_CRASH_CHECK = 2        # Hold and disarm        │
│ CRASH_ANGLE = 30          # 30 degree threshold    │
│                                                     │
│ FS_EKF_ACTION = 1         # Hold if nav fails      │
│ FS_EKF_THRESH = 0.8       # EKF sensitivity        │
└────────────────────────────────────────────────────┘

Write Parameters
```

---

### ✅ **Step 10: Arming Configuration**

**Configure arming requirements:**

```
┌────────────────────────────────────────────────────┐
│ ARMING_CHECK = 1          # All pre-arm checks     │
│ ARMING_REQUIRE = 1        # Require arming         │
│ ARMING_RUDDER = 0         # Disable stick arming   │
└────────────────────────────────────────────────────┘

Set up arm/disarm switch (recommended):
Find unused RC channel (e.g., Ch7):
┌────────────────────────────────────────────────────┐
│ RC7_OPTION = 41           # Arm/Disarm switch      │
└────────────────────────────────────────────────────┘

Write Parameters
```

---

## 🔧 Calibration Procedures

### **1. RC Radio Calibration (CRSF)** ⭐ MANDATORY

```
1. Verify CRSF receiver is connected and bound to transmitter
   • Both RX and TX wires connected to UART2
   • Receiver LED solid (not blinking) = bound
   • If blinking: Use bind button/procedure

2. Mission Planner → CONFIG → Mandatory Hardware → Radio Calibration

3. Turn on RC transmitter

4. You should see channels responding (if CRSF working)
   • Green bars move with sticks
   • RSSI value appears (signal strength) ⭐ CRSF feature

5. Click "Calibrate Radio"

6. Move all controls through FULL range:
   • Throttle: Full up and full down
   • Steering: Full left and full right
   • Mode switch: All positions
   • Any other switches

7. Click "Click when Done"

8. Verify:
   ✓ Green bars move with stick movement
   ✓ RC1 range: ~1000-2000
   ✓ RC3 range: ~1000-2000
   ✓ All channels responsive
   ✓ RSSI shows value (80-100 typical) ⭐ CRSF only

9. Save
```

**CRSF-Specific Notes:**
- If no RC signal: Check SERIAL2_PROTOCOL = 23 and reboot FC
- CRSF updates faster than PPM (5-10ms vs 20ms)
- Link quality and RSSI available on transmitter screen

---

### **2. Accelerometer Calibration** ⭐ MANDATORY

```
1. Mission Planner → SETUP → Mandatory Hardware → Accel Calibration

2. Place tank on FLAT, LEVEL surface

3. Click "Calibrate Accel"

4. Follow prompts for 6 positions:
   
   Position 1: Level (keep as-is)
   → Click Next
   
   Position 2: On left side
   → Tilt 90° to left
   → Click Next
   
   Position 3: On right side
   → Tilt 90° to right
   → Click Next
   
   Position 4: Nose down
   → Tilt front down 90°
   → Click Next
   
   Position 5: Nose up
   → Tilt front up 90°
   → Click Next
   
   Position 6: On back (upside down)
   → Flip completely over
   → Click Next

5. "Calibration successful"

6. Place back to normal position
```

**Why this matters:**
- Crash detection relies on level reference
- Pitch/roll calculations need accurate zero point
- Balance bots need this for stability

---

### **3. Compass Calibration** (For GPS/Autonomous)

```
1. Mission Planner → SETUP → Mandatory Hardware → Compass

2. Select compass(es) to calibrate

3. Click "Onboard Mag Calibration" → Start

4. Pick up rover and rotate slowly in ALL directions:
   • Rotate around yaw axis (360°)
   • Rotate around pitch axis (360°)
   • Rotate around roll axis (360°)
   • Continue for 60 seconds
   • Keep rotating smoothly

5. Progress bar reaches 100%

6. "Calibration successful"

7. Verify offsets:
   ✓ X, Y, Z offsets should be < 500
   ✓ If > 500, redo calibration away from metal
```

**When needed:**
- ✅ YES if using GPS for autonomous navigation
- ❌ NO if only driving in Manual mode

---

### **4. ESC Calibration** (Recommended)

**Purpose:** All 4 ESCs respond identically to throttle

**Method 1: Automatic (via Mission Planner)**

```
1. DISCONNECT BATTERY! ⚠️

2. Mission Planner → SETUP → Optional Hardware → ESC Calibration

3. Click "Calibrate ESCs"

4. Follow prompts:
   → "Disconnect battery now"
   → "Move throttle to maximum"
   → "Connect battery now"
   → ESCs beep (high tones)
   → "Move throttle to minimum"
   → ESCs beep (confirmation)

5. "Calibration complete"

6. Disconnect battery

7. Reconnect everything normally
```

**Method 2: Individual ESC Calibration**

```
For each ESC separately:
1. Disconnect ESC signal wire from FC
2. Connect ESC signal to RC receiver Ch3 (throttle)
3. Turn on transmitter, throttle to MAX
4. Connect battery to ESC
5. ESC beeps (high tone)
6. Move throttle to MIN
7. ESC beeps (low tone) - calibrated!
8. Disconnect battery
9. Reconnect ESC to flight controller
10. Repeat for all 4 ESCs
```

---

## 🧪 Testing Sequence

### **Phase 1: Bench Test (No Battery)**

**Safety:** USB power only, no motors spinning

```
1. Connect flight controller via USB

2. Open Mission Planner

3. Connect to rover

4. Test RC inputs:
   CONFIG → Radio Calibration
   ✓ Move sticks → bars move
   ✓ Toggle switches → values change

5. Test mode switching:
   DATA → Quick tab
   ✓ Toggle mode switch
   ✓ "Mode" field changes
   ✓ All 6 positions work

6. Check servo outputs:
   DATA → Status → Servo/Relay
   ✓ Move throttle → SERVO1-4 values change
   ✓ Move steering → SERVO1-2 increase, SERVO3-4 decrease
```

---

### **Phase 2: Motor Direction Test (USB Power)**

**Safety:** No battery, just USB

```
1. USB connected only

2. Mission Planner → CONFIG → Optional Hardware → Motor Test

3. Test SERVO1 (Left-Front):
   Throttle: 1600 µs (forward direction)
   Duration: 2 sec
   Click "Test motor A"
   → Motor should try to spin forward
   → Note direction

4. Test SERVO2 (Left-Rear):
   Same settings, Test motor B
   → Should spin SAME direction as SERVO1

5. Test SERVO3 (Right-Front):
   Same settings, Test motor C
   → Should spin forward

6. Test SERVO4 (Right-Rear):
   Same settings, Test motor D
   → Should spin SAME direction as SERVO3

7. Fix any reversed motors:
   If motor spins backward:
   Set SERVOn_REVERSED = 1
```

---

### **Phase 3: Powered Test (Battery, Wheels Removed)** ⚠️

**Safety:** Battery connected, but wheels/tracks removed or tank lifted

```
1. Remove wheels/tracks OR lift tank off ground

2. Connect battery

3. Turn on RC transmitter

4. Set mode to MANUAL (switch position 1)

5. ARM the rover:
   • Flip arm switch
   • OR throttle down + steering right for 2 sec

6. Slowly apply throttle:
   ✓ All 4 motors spin
   ✓ Left pair spins same direction
   ✓ Right pair spins same direction
   ✓ Both pairs spin FORWARD

7. Test steering:
   Right stick → Left motors speed up, right slow down
   Left stick → Right motors speed up, left slow down

8. Test reverse:
   Throttle down → All motors reverse

9. DISARM when done

10. Check for issues, fix if needed
```

---

### **Phase 4: Ground Test (First Drive)** ⚠️⚠️

**Safety:** Clear area, low throttle limits

```
1. Set throttle limit (safety):
   MOT_THR_MAX = 50  # Only 50% max initially

2. Place tank on ground in SAFE, OPEN area

3. Set mode to MANUAL

4. ARM rover

5. Slowly test controls:
   ✓ Forward (small throttle)
   ✓ Reverse
   ✓ Turn left
   ✓ Turn right
   ✓ Pivot turn

6. Test HOLD mode:
   Switch to MODE2
   → Tank should STOP immediately

7. Back to MANUAL, continue testing

8. DISARM when complete

9. If all good:
   MOT_THR_MAX = 100  # Remove throttle limit
```

---

### **Phase 5: Autonomous Test** (GPS required)

```
1. Go outdoors (clear sky)

2. Wait for GPS lock:
   ✓ 10+ satellites
   ✓ HDOP < 1.5
   ✓ 3D Fix

3. Set HOME:
   Actions → Set Home → Set Home Here

4. Test LOITER mode:
   • Switch to LOITER
   • Tank should hold position
   • Try pushing it → should correct

5. Create simple mission:
   • 3 waypoints in 50m radius
   • Upload to rover

6. Test AUTO mode:
   • Switch to AUTO
   • Tank drives waypoint to waypoint
   • Maintains CRUISE_SPEED

7. Test RTL:
   • Drive away 30m
   • Switch to RTL
   • Tank drives back to HOME

8. SUCCESS! ✓
```

---

## 🛡️ Safety Configuration

### **Critical Safety Features:**

```
# RC Failsafe
FS_THR_ENABLE = 1          # Enable
FS_THR_VALUE = 910         # Trigger at 910 µs
FS_ACTION = 2              # Hold (stop)
FS_TIMEOUT = 1.5           # Trigger after 1.5 sec

# Crash Detection
FS_CRASH_CHECK = 2         # Hold and disarm
CRASH_ANGLE = 30           # Trigger at 30° tilt

# Navigation Failsafe
FS_EKF_ACTION = 1          # Hold if nav fails
FS_EKF_THRESH = 0.8        # Standard sensitivity

# Battery Failsafe (if monitoring enabled)
BATT_FS_LOW_ACT = 2        # RTL on low battery
BATT_FS_CRT_ACT = 1        # Hold on critical
BATT_LOW_VOLT = 10.5       # Low voltage (3S example)
BATT_CRT_VOLT = 9.9        # Critical voltage

# Geofence (optional but recommended)
FENCE_ENABLE = 1           # Enable
FENCE_TYPE = 2             # Circle
FENCE_RADIUS = 100         # 100m radius
FENCE_ACTION = 1           # RTL or Hold
```

---

## 🚀 Optional Enhancements

### **1. Wheel Encoders**

**Benefits:**
- Better odometry
- GPS-denied navigation
- More accurate positioning

**Configuration:**
```
WENC_TYPE1 = 1             # Enable encoder 1
WENC_CPR1 = 360            # Counts per revolution
WENC_RADIUS1 = 0.05        # Wheel radius (m)
WENC_POS1_X = 0.15         # Position X
WENC_POS1_Y = -0.2         # Position Y (left side)

# Repeat for WENC2 (right encoder)
```

---

### **2. Rangefinder (Obstacle Detection)**

**Configuration:**
```
SERIAL4_PROTOCOL = 9       # Rangefinder
SERIAL4_BAUD = 115         # 115200

RNGFND1_TYPE = 1           # LIDAR type
RNGFND1_ORIENT = 0         # Forward
RNGFND1_MIN_CM = 10        # 10cm min
RNGFND1_MAX_CM = 500       # 5m max

# Enable avoidance
AVOID_ENABLE = 1           # Enable
AVOID_MARGIN = 2.0         # 2m safety margin
AVOID_BACKUP_SPD = -0.5    # Reverse at 0.5 m/s
```

---

### **3. Telemetry Options**

**Option A: CRSF Telemetry (to RC Transmitter)** ⭐ RECOMMENDED

Already configured with RC_OPTIONS = 256, gives you on transmitter:
- Battery voltage
- GPS status & satellites
- Flight mode
- Speed, altitude, heading
- Distance from home
- RSSI & link quality

**Option B: MAVLink Telemetry Radio (to Computer/GCS)**

```
SERIAL1_PROTOCOL = 2       # MAVLink2
SERIAL1_BAUD = 57          # 57600 (standard)

# Stream rates (adjust as needed)
SR1_POSITION = 5           # 5 Hz
SR1_EXTRA1 = 10            # 10 Hz
SR1_RC_CHAN = 2            # 2 Hz
```

**You can use BOTH:** CRSF telemetry to TX + MAVLink to GCS simultaneously!

---

### **4. Data Logging**

**Configuration:**
```
LOG_BITMASK = 65535        # Log everything
LOG_DISARMED = 0           # Don't log when disarmed
LOG_REPLAY = 1             # Enable replay

# Useful log types:
# 4 = Throttle
# 5 = Navigation  
# 8 = Mission Commands
# 13 = Steering
```

---

### **5. Advanced Tuning**

**After basic driving works:**

```
# Acceleration limits (smoother driving)
ATC_ACCEL_MAX = 2.0        # 2 m/s/s max accel
ATC_DECEL_MAX = 3.0        # 3 m/s/s max decel

# Turn behavior
ATC_TURN_MAX_G = 0.2       # Limit lateral G-force

# Reverse delay
MOT_REVERSE_DELAY = 2.0    # 2 sec delay direction change

# Steering vs throttle priority
MOT_STR_THR_MIX = 0.7      # Prioritize steering (0.5-1.0)
```

---

## 🔍 Troubleshooting

### **Problem: No RC signal (CRSF)**

**Check:**
```
□ SERIAL2_PROTOCOL = 23?
□ Both RX and TX wires connected?
   - FC R2 ← RX from receiver
   - FC T2 → TX from receiver
□ Receiver bound to transmitter? (LED solid, not blinking)
□ Receiver powered (5V and GND connected)?
□ Flight controller rebooted after setting SERIAL2_PROTOCOL?
□ Transmitter turned on?

Try:
1. Re-bind receiver to transmitter
2. Check wiring (swap RX/TX if needed)
3. Verify SERIAL2_PROTOCOL = 23
4. Reboot FC
```

---

### **Problem: Motors don't spin**

**Check:**
```
□ Battery connected and charged?
□ ESCs powered on (beep sounds)?
□ Rover is ARMED?
□ In MANUAL or ACRO mode?
□ MOT_PWM_TYPE correct for your ESCs?
□ RC calibrated?
□ Failsafe not triggered?
□ CRSF receiver showing bound (solid LED)?
```

---

### **Problem: One or more motors spin backward**

**Solution:**
```
Set SERVO_REVERSED:
SERVO1_REVERSED = 1  # If left-front backward
SERVO2_REVERSED = 1  # If left-rear backward
SERVO3_REVERSED = 1  # If right-front backward
SERVO4_REVERSED = 1  # If right-rear backward

OR physically swap 2 motor wires at ESC
```

---

### **Problem: Tank turns wrong direction**

**Solution:**
```
Swap left/right functions:
SERVO1_FUNCTION = 74  # Right
SERVO2_FUNCTION = 74  # Right
SERVO3_FUNCTION = 73  # Left
SERVO4_FUNCTION = 73  # Left
```

---

### **Problem: Can't arm**

**Check:**
```
Look at Mission Planner messages:
"PreArm: RC not calibrated" → Calibrate RC
"PreArm: Accel not calibrated" → Calibrate accel
"PreArm: Need 3D Fix" → Wait for GPS (or disable GPS check)
"PreArm: Compass not calibrated" → Calibrate compass

Disable specific checks (temporarily):
ARMING_CHECK = 0  # Disable all (use carefully!)
```

---

### **Problem: Motors spin when disarmed**

**Solution:**
```
MOT_SAFE_DISARM = 1        # Disable PWM when disarmed
```

---

### **Problem: Turns are jerky/oscillating**

**Solution:**
```
Reduce steering gain:
ATC_STR_RAT_P = 0.1        # Lower from 0.2
ATC_STR_RAT_I = 0.1        # Lower from 0.2

Add filtering:
ATC_STR_RAT_FILT = 5       # Lower from 10
```

---

### **Problem: Speed not regulated in Auto mode**

**Check:**
```
□ GPS has good lock?
□ CRUISE_SPEED set to reasonable value?
□ ATC_SPEED_P not zero?
□ Vehicle actually moving?

Try:
ATC_SPEED_P = 0.3          # Increase from 0.2
CRUISE_THROTTLE = 60       # Increase from 50
```

---

## 📊 Complete Parameter Summary

**Minimum viable tank configuration with CRSF:**

```ini
# ============ FRAME & VEHICLE ============
FRAME_CLASS = 1

# ============ RC INPUT - CRSF ============
SERIAL2_PROTOCOL = 23         # RCIN (CRSF protocol)
SERIAL2_BAUD = 115            # 115200 (auto-adjusts to 420000)
SERIAL2_OPTIONS = 0           # Normal
RSSI_TYPE = 3                 # Get RSSI from CRSF
RC_OPTIONS = 256              # Enable CRSF telemetry (optional)

# ============ MOTOR OUTPUTS ============
SERVO1_FUNCTION = 73       # Left-Front
SERVO2_FUNCTION = 73       # Left-Rear
SERVO3_FUNCTION = 74       # Right-Front
SERVO4_FUNCTION = 74       # Right-Rear

SERVO1_MIN = 1000
SERVO1_MAX = 2000
SERVO1_TRIM = 1500
SERVO2_MIN = 1000
SERVO2_MAX = 2000
SERVO2_TRIM = 1500
SERVO3_MIN = 1000
SERVO3_MAX = 2000
SERVO3_TRIM = 1500
SERVO4_MIN = 1000
SERVO4_MAX = 2000
SERVO4_TRIM = 1500

# ============ MOTOR CONFIG ============
MOT_PWM_TYPE = 0
MOT_THR_MIN = 0
MOT_THR_MAX = 100
MOT_SLEWRATE = 100
MOT_THST_EXPO = 0.0
MOT_STR_THR_MIX = 0.5

# ============ STEERING CONTROL ============
ATC_STR_RAT_P = 0.2
ATC_STR_RAT_I = 0.2
ATC_STR_RAT_D = 0.0
ATC_STR_RAT_MAX = 180
ATC_STR_RAT_FILT = 10
ATC_STR_ANG_P = 2.0

# ============ SPEED CONTROL ============
CRUISE_SPEED = 2.0
CRUISE_THROTTLE = 50
ATC_SPEED_P = 0.2
ATC_SPEED_I = 0.2
ATC_SPEED_D = 0.0

# ============ MODES ============
MODE_CH = 5
MODE1 = 0                  # Manual
MODE2 = 4                  # Hold
MODE3 = 3                  # Steering
MODE4 = 10                 # Auto
MODE5 = 11                 # RTL
MODE6 = 15                 # Guided

# ============ RC MAPPING ============
RCMAP_ROLL = 1             # Ch1 = Steering
RCMAP_THROTTLE = 3         # Ch3 = Throttle

# ============ FAILSAFE ============
FS_ACTION = 2              # Hold
FS_TIMEOUT = 1.5
FS_THR_ENABLE = 1
FS_THR_VALUE = 910
FS_GCS_ENABLE = 0
FS_CRASH_CHECK = 2
CRASH_ANGLE = 30
FS_EKF_ACTION = 1
FS_EKF_THRESH = 0.8

# ============ ARMING ============
ARMING_CHECK = 1
ARMING_REQUIRE = 1
ARMING_RUDDER = 0

# ============ NAVIGATION (if GPS) ============
WP_SPEED = 2.0
WP_RADIUS = 2.0
RTL_SPEED = 1.5
LOIT_RADIUS = 2.0

# ============ BATTERY (if equipped) ============
BATT_MONITOR = 4
BATT_CAPACITY = 5000
BATT_VOLT_PIN = 10
BATT_CURR_PIN = 11
BATT_VOLT_MULT = 11.2
BATT_LOW_VOLT = 10.5

# ============ GPS (if equipped) ============
SERIAL6_PROTOCOL = 5
SERIAL6_BAUD = 230
GPS_TYPE = 1
```

---

## ✅ Final Checklist Before First Drive

```
Hardware:
□ All 4 motors connected to correct outputs
□ ESCs connected to power
□ Battery charged
□ CRSF receiver connected to UART2 (RX + TX wires) ⭐
□ CRSF receiver bound to transmitter
□ GPS connected (if using autonomous)

Software:
□ Firmware flashed
□ Parameters configured
□ RC calibrated ⭐
□ Accelerometer calibrated ⭐
□ Compass calibrated (if GPS)
□ Motor directions verified ⭐
□ Modes configured
□ Arming works
□ Failsafe tested

Testing:
□ Bench test passed
□ Motor direction test passed
□ Powered test passed (wheels off)
□ Ground test in safe area

Safety:
□ Emergency stop mode accessible (Hold)
□ Failsafe configured
□ Geofence set (optional)
□ Someone nearby to help if issues

Ready to Drive:
□ All above items checked ✓
□ Start in MANUAL mode
□ Low throttle initially
□ Open, clear area
□ LET'S GO! 🚀
```

---

## 📝 Notes & Tips

1. **Start conservative:**
   - Low CRUISE_SPEED (1-2 m/s)
   - Low MOT_THR_MAX initially (50%)
   - Test in MANUAL first

2. **Tune incrementally:**
   - Change one parameter at a time
   - Test after each change
   - Document what works

3. **Keep parameters backed up:**
   - Save parameters to file regularly
   - Before tuning: Save current settings
   - Can restore if needed

4. **Join the community:**
   - https://discuss.ardupilot.org/
   - Search for "tank" or "skid steering"
   - Ask questions if stuck

5. **Watch the logs:**
   - Download logs after flights
   - Analyze in Mission Planner
   - Helps diagnose issues

6. **CRSF telemetry benefits:**
   - Monitor battery on transmitter screen
   - See flight mode without GCS
   - Check GPS satellites count
   - View link quality and RSSI
   - No need to look at computer while driving!

---

## 📡 CRSF Telemetry Setup (Optional Enhancement)

### **What CRSF Telemetry Gives You:**

With `RC_OPTIONS = 256` enabled, your transmitter displays:

```
Transmitter Screen Shows:
├── Battery Voltage (real-time)
├── Current Draw (amps)
├── Flight Mode (Manual, Auto, RTL, etc.)
├── GPS Satellites (count)
├── GPS Coordinates (lat/lon)
├── Altitude (from barometer)
├── Speed (ground speed)
├── Heading (degrees)
├── Distance from Home
├── RSSI (signal strength)
└── Link Quality (0-100%)
```

**No laptop needed while driving!**

### **Transmitter Configuration:**

**For OpenTX/EdgeTX (Radiomaster, etc.):**

```
1. TELEMETRY screen on transmitter
2. Discover new sensors (should auto-detect)
3. Available sensors:
   - RxBt (Battery voltage)
   - Curr (Current)
   - GPS (Position)
   - GSpd (Ground speed)
   - Hdg (Heading)
   - FM (Flight mode)
   - etc.

4. Add to main screen:
   DISPLAY → Add widgets
   - Battery voltage widget
   - GPS widget
   - Flight mode text
   - etc.
```

**For Yaapu Telemetry Script:**

```
Enhanced telemetry display with:
- Artificial horizon
- Speed indicators
- Battery graphs
- Mission progress
- Warnings and alerts

Download: https://github.com/yaapu/FrskyTelemetryScript
Install on SD card in transmitter
```

---

**Document complete!** This is your complete tank setup reference with CRSF protocol. Follow step-by-step and you'll have a working autonomous tank with telemetry! 🎯

