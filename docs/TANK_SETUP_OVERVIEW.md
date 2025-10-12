# ArduRover Tank Setup - Complete Overview
## Quick Reference Guide for 4-Motor Skid-Steer Tank Configuration

**Document:** High-level overview and quick reference  
**Hardware:** SpeedyBee F405 V3 (or similar STM32F4/F7/H7)  
**Configuration:** 4 motors, 2 left + 2 right, skid-steering  
**Date:** 2025-10-10

---

## 🎯 What This Document Covers

This is a **quick reference** document that links to four detailed guides:

1. **[Complete Setup Checklist](TANK_COMPLETE_SETUP_CHECKLIST.md)** - Everything you need to configure
2. **[CRUISE_SPEED Explained](CRUISE_SPEED_EXPLAINED.md)** - Understanding cruise speed control
3. **[RC Mode Switching Guide](RC_MODE_SWITCHING_GUIDE.md)** - How to switch modes with RC
4. **[Steering Concept for Tanks](STEERING_CONCEPT_FOR_TANKS.md)** - Why tanks have steering parameters

---

## ⚡ Quick Start (30-Minute Setup)

### **Absolute Minimum to Drive Manually:**

```bash
# 1. Flash firmware (5 minutes)
./waf configure --board speedybeef4v3
./waf rover
# Flash ardurover.apj via Mission Planner

# 2. Essential parameters (5 minutes)
FRAME_CLASS = 1
SERVO1_FUNCTION = 73    # Left-Front
SERVO2_FUNCTION = 73    # Left-Rear
SERVO3_FUNCTION = 74    # Right-Front
SERVO4_FUNCTION = 74    # Right-Rear
MOT_PWM_TYPE = 0
MODE1 = 0  # Manual

# 3. Calibrations (15 minutes)
□ Calibrate RC radio
□ Calibrate accelerometer
□ Test motor directions

# 4. Test drive (5 minutes)
□ Set mode to Manual
□ ARM rover
□ Drive!
```

**Total time:** 30 minutes to first drive (manual only)

---

## 📊 Hardware Architecture

### **Your Tank Configuration:**

```
        ┌─────────────────────────┐
        │  SpeedyBee F405 V3 FC   │
        └──┬──┬──────────┬──┬─────┘
           │  │          │  │
      PWM1 │  │ PWM2     │  │ PWM3   PWM4
           │  │          │  │
        ┌──┴──┴──┐    ┌──┴──┴──┐
        │ ESC 1/2 │    │ ESC 3/4 │
        └──┬──┬───┘    └───┬──┬──┘
           │  │            │  │
       ┌───┴──┴───┐    ┌───┴──┴───┐
       │ Motor 1  │    │ Motor 3  │
       │ Motor 2  │    │ Motor 4  │
       │ (LEFT)   │    │ (RIGHT)  │
       └──────────┘    └──────────┘
           │                │
       ┌───┴────────────────┴───┐
       │    Tank Chassis        │
       └────────────────────────┘
```

---

## ⚙️ Critical Parameters (Must Set)

### **The Bare Minimum:**

```ini
# Vehicle Type
FRAME_CLASS = 1                    # Ground rover

# Motor Assignments (4-motor tank)
SERVO1_FUNCTION = 73               # ThrottleLeft (Front-Left)
SERVO2_FUNCTION = 73               # ThrottleLeft (Rear-Left)
SERVO3_FUNCTION = 74               # ThrottleRight (Front-Right)
SERVO4_FUNCTION = 74               # ThrottleRight (Rear-Right)

# Motor Control
MOT_PWM_TYPE = 0                   # Normal PWM
MOT_STR_THR_MIX = 0.5              # Balanced steering/throttle

# Basic Tuning
ATC_STR_RAT_P = 0.2                # Steering response
CRUISE_SPEED = 2.0                 # Target speed (m/s)
CRUISE_THROTTLE = 50               # Throttle guess (%)

# Modes
MODE_CH = 5                        # Channel 5 for modes
MODE1 = 0                          # Manual (safe default)
MODE2 = 4                          # Hold (emergency stop)
MODE4 = 10                         # Auto (missions)

# Safety
FS_ACTION = 2                      # Hold on failsafe
FS_THR_ENABLE = 1                  # RC failsafe on
ARMING_REQUIRE = 1                 # Require arming
```

**Parameter count:** ~20 essential parameters  
**Time to configure:** 5-10 minutes

---

## 🔧 Calibration Requirements

### **Mandatory:**

| **Calibration** | **Time** | **When** | **Why** |
|----------------|----------|----------|---------|
| **RC Radio** | 2 min | Before first drive | Maps your transmitter to ArduPilot |
| **Accelerometer** | 5 min | Before first drive | Level reference, crash detection |
| **Motor Directions** | 5 min | Before first drive | Ensure correct spin direction |

### **For Autonomous Operation:**

| **Calibration** | **Time** | **When** | **Why** |
|----------------|----------|----------|---------|
| **Compass** | 3 min | Before Auto/RTL | Heading reference |
| **GPS** | - | Every boot (outdoors) | Position estimation |

**Total calibration time:** 15-20 minutes

---

## 🎮 Flight Modes Quick Reference

### **Modes You'll Actually Use:**

| **Mode** | **Number** | **Use Case** | **Needs GPS?** |
|----------|-----------|-------------|---------------|
| **Manual** | 0 | Direct RC control | ❌ No |
| **Hold** | 4 | Emergency stop | ❌ No |
| **Steering** | 3 | Speed-controlled manual | ❌ No |
| **Auto** | 10 | Waypoint missions | ✅ Yes |
| **RTL** | 11 | Return to launch | ✅ Yes |
| **Loiter** | 5 | Hold GPS position | ✅ Yes |

**For your tank:** Focus on Manual, Hold, and Auto initially.

---

## 📡 Wiring Diagram

### **Complete Tank Wiring:**

```
┌──────────────────────────────────────────────────┐
│          SpeedyBee F405 V3                        │
│                                                   │
│  Power:                                          │
│  ├── VBAT ← Battery +                            │
│  └── GND  ← Battery -                            │
│                                                   │
│  Motors:                                         │
│  ├── M1 (PWM1) ──→ ESC1 ──→ Motor 1 (L-Front)   │
│  ├── M2 (PWM2) ──→ ESC2 ──→ Motor 2 (L-Rear)    │
│  ├── M3 (PWM3) ──→ ESC3 ──→ Motor 3 (R-Front)   │
│  └── M4 (PWM4) ──→ ESC4 ──→ Motor 4 (R-Rear)    │
│                                                   │
│  RC Input:                                       │
│  └── RC IN ←──── PPM from RC Receiver            │
│                                                   │
│  Communication:                                  │
│  ├── USB ←──────→ Computer (setup/telemetry)     │
│  ├── UART1 ←────→ Telemetry radio (optional)     │
│  └── UART6 ←────→ GPS module (for autonomous)    │
│                                                   │
└──────────────────────────────────────────────────┘

Power Distribution:
┌──────────────────────────────────────────────────┐
│ Battery ─┬─→ PDB ─┬─→ ESC1, ESC2, ESC3, ESC4     │
│          │        └─→ Flight Controller (5V BEC)  │
│          └─→ Voltage/Current sensor               │
└──────────────────────────────────────────────────┘
```

---

## 📋 Setup Sequence (Recommended Order)

```
Day 1: Basic Setup (1-2 hours)
├── 1. Flash ArduRover firmware
├── 2. Connect to Mission Planner
├── 3. Set FRAME_CLASS = 1
├── 4. Configure SERVO1-4 functions
├── 5. Set MOT_* parameters
├── 6. Configure modes (MODE1-6)
├── 7. Calibrate RC radio ⭐
├── 8. Calibrate accelerometer ⭐
└── 9. Test motor directions ⭐

Day 2: Testing & Tuning (2-3 hours)
├── 10. Fix reversed motors (if needed)
├── 11. Bench test with battery (wheels off)
├── 12. First ground test (Manual mode)
├── 13. Tune ATC_STR_RAT_P for good turning
├── 14. Set appropriate CRUISE_SPEED
└── 15. Test different modes

Day 3: Autonomous Setup (2-3 hours) [Optional]
├── 16. Connect GPS module
├── 17. Calibrate compass
├── 18. Wait for GPS lock
├── 19. Set HOME position
├── 20. Test Loiter mode
├── 21. Create simple mission
├── 22. Test Auto mode
└── 23. Test RTL mode

Total: 5-8 hours to fully operational autonomous tank
```

---

## 🎯 Key Concepts Clarified

### **1. Frame Types:**

```
FRAME_CLASS:    What KIND of vehicle (Rover, Boat, BalanceBot)
FRAME_TYPE:     What MOTOR ARRANGEMENT (Omni3, OmniX, etc.)

For your tank:
FRAME_CLASS = 1  (Rover)
FRAME_TYPE = 0   (Default/Standard - includes skid-steer)
```

### **2. Steering in Tanks:**

```
"Steering" ≠ Physical steering servo
"Steering" = Turning control via differential thrust

ATC_STR_* parameters control:
• How aggressively tracks differentiate
• Maximum turning speed
• Response to heading errors
```

### **3. Cruise Speed:**

```
CRUISE_SPEED = Target speed for autonomous modes
• Auto mode: Drives at this speed
• RTL mode: Returns at this speed (or RTL_SPEED)
• Manual mode: IGNORED (direct throttle)

Controller automatically adjusts throttle to maintain speed
```

### **4. Mode Switching:**

```
MODE_CH = RC channel for mode selection (usually Ch5)
MODE1-6 = What mode for each switch position

PWM ranges automatically map to MODE slots
Toggle switch → Change mode → Change behavior
```

---

## 🔗 Related Documentation Links

### **In This Repository:**

- **[Complete Setup Checklist](TANK_COMPLETE_SETUP_CHECKLIST.md)** - Full step-by-step guide
- **[CRUISE_SPEED Explained](CRUISE_SPEED_EXPLAINED.md)** - Speed control deep dive
- **[RC Mode Switching](RC_MODE_SWITCHING_GUIDE.md)** - Mode configuration guide  
- **[Steering Concept](STEERING_CONCEPT_FOR_TANKS.md)** - Why tanks have steering
- **[Parameter Reference](../ROVER_PARAMETERS_REFERENCE.md)** - All parameters explained

### **Official ArduPilot Documentation:**

- **First Time Setup:** https://ardupilot.org/rover/docs/apmrover-setup.html
- **Motor Configuration:** https://ardupilot.org/rover/docs/rover-motor-and-servo-configuration.html
- **Parameter List:** https://ardupilot.org/rover/docs/parameters.html
- **Flight Modes:** https://ardupilot.org/rover/docs/rover-modes.html

### **Community Resources:**

- **Forum:** https://discuss.ardupilot.org/c/ground-rovers
- **Discord:** https://ardupilot.org/discord (ask questions!)
- **GitHub:** https://github.com/ArduPilot/ardupilot

---

## 🛠️ Troubleshooting Quick Reference

### **Common Issues:**

| **Problem** | **Quick Fix** | **Detailed Guide** |
|------------|--------------|-------------------|
| Motors backward | Set `SERVOn_REVERSED = 1` | Setup Checklist §5 |
| Can't arm | Check pre-arm messages | Setup Checklist §10 |
| Won't turn | Increase `ATC_STR_RAT_P` | Steering Concept doc |
| Turns too sharp | Decrease `ATC_STR_RAT_P` | Steering Concept doc |
| Speed not regulated | Check GPS, tune `ATC_SPEED_P` | CRUISE_SPEED doc |
| Modes don't switch | Verify `MODE_CH`, calibrate RC | Mode Switching doc |
| Crashes in Auto | Lower `CRUISE_SPEED`, tune steering | Setup Checklist §13 |

---

## 📈 Tuning Progression

### **Start Here (Conservative):**

```ini
CRUISE_SPEED = 1.5         # Slow
MOT_THR_MAX = 60           # Limited power
ATC_STR_RAT_P = 0.15       # Gentle steering
ATC_STR_RAT_MAX = 120      # Slow turns
```

### **Progress To (Normal):**

```ini
CRUISE_SPEED = 2.0         # Standard
MOT_THR_MAX = 100          # Full power
ATC_STR_RAT_P = 0.2        # Responsive
ATC_STR_RAT_MAX = 180      # Normal turns
```

### **Advanced (Performance):**

```ini
CRUISE_SPEED = 3.0         # Fast
ATC_STR_RAT_P = 0.3        # Aggressive
ATC_STR_RAT_MAX = 360      # Pivot turns
MOT_STR_THR_MIX = 0.7      # Tight turns
ATC_ACCEL_MAX = 3.0        # Quick acceleration
```

**Tune incrementally!** Test after each change.

---

## 🎓 Understanding the System

### **Control Flow Diagram:**

```
[RC Transmitter]
       ↓
   Throttle → Desired forward/backward speed
   Steering → Desired turn rate
       ↓
[ArduRover Mode]
       ↓
   Manual:   Pass RC directly to motors
   Steering: Use RC as speed reference
   Auto:     Calculate from waypoints
       ↓
[Speed Controller]
   ATC_SPEED_P/I/D
   Maintains CRUISE_SPEED
       ↓
[Steering Controller]
   ATC_STR_RAT_P/I/D
   Achieves desired turn rate
       ↓
[Motor Mixer - Skid Steering]
   Left  = Throttle + Steering
   Right = Throttle - Steering
       ↓
[Physical Outputs]
   SERVO1 & SERVO2 → Left tracks
   SERVO3 & SERVO4 → Right tracks
       ↓
[Tank Movement]
   Differential thrust creates turns
```

---

## 🧩 How It All Fits Together

### **The Big Picture:**

```
You configure:
├── What vehicle type (FRAME_CLASS)
├── How motors are arranged (SERVO_FUNCTION)
├── How aggressive to turn (ATC_STR_*)
├── How fast to cruise (CRUISE_SPEED)
└── What modes to use (MODE1-6)

ArduRover handles:
├── Reading RC inputs
├── Sensor fusion (GPS, IMU, compass)
├── Speed regulation (maintains CRUISE_SPEED)
├── Heading control (points toward target)
├── Differential mixing (turns via track speed)
├── Failsafe actions (if signal lost)
└── Mission execution (follows waypoints)

You just:
├── Toggle mode switch
├── Let it drive autonomously
└── Monitor progress
```

---

## 📝 Essential Parameter Summary

### **Copy-Paste Ready Configuration:**

```ini
# ========================================
# TANK CONFIGURATION - SpeedyBee F405 V3
# 4 Motors: 2 Left + 2 Right
# ========================================

# VEHICLE
FRAME_CLASS=1

# MOTORS
SERVO1_FUNCTION=73
SERVO2_FUNCTION=73
SERVO3_FUNCTION=74
SERVO4_FUNCTION=74
SERVO1_MIN=1000
SERVO1_MAX=2000
SERVO1_TRIM=1500
SERVO2_MIN=1000
SERVO2_MAX=2000
SERVO2_TRIM=1500
SERVO3_MIN=1000
SERVO3_MAX=2000
SERVO3_TRIM=1500
SERVO4_MIN=1000
SERVO4_MAX=2000
SERVO4_TRIM=1500

# MOTOR CONTROL
MOT_PWM_TYPE=0
MOT_THR_MIN=0
MOT_THR_MAX=100
MOT_SLEWRATE=100
MOT_STR_THR_MIX=0.5

# STEERING
ATC_STR_RAT_P=0.2
ATC_STR_RAT_I=0.2
ATC_STR_RAT_D=0.0
ATC_STR_RAT_MAX=180
ATC_STR_ANG_P=2.0

# SPEED
CRUISE_SPEED=2.0
CRUISE_THROTTLE=50
ATC_SPEED_P=0.2
ATC_SPEED_I=0.2

# MODES
MODE_CH=5
MODE1=0
MODE2=4
MODE3=3
MODE4=10
MODE5=11
MODE6=15

# RC MAPPING
RCMAP_ROLL=1
RCMAP_THROTTLE=3

# FAILSAFE
FS_ACTION=2
FS_TIMEOUT=1.5
FS_THR_ENABLE=1
FS_THR_VALUE=910
FS_CRASH_CHECK=2
CRASH_ANGLE=30

# ARMING
ARMING_CHECK=1
ARMING_REQUIRE=1
```

**Save this as:** `tank_baseline.param`  
**Load via:** Mission Planner → CONFIG → Load from file

---

## 🚀 Next Steps

### **After Basic Setup:**

```
1. ✅ You can drive in Manual mode
   → What next: Practice driving, get comfortable

2. ✅ Add GPS for autonomous
   → What next: Test Loiter, create missions

3. ✅ Tune steering for your terrain
   → What next: Adjust ATC_STR_RAT_P as needed

4. ✅ Optimize cruise speed
   → What next: Test different speeds, find optimal

5. ✅ Add advanced features
   → What next: Rangefinders, wheel encoders, etc.
```

---

## 📚 Learning Resources

### **Suggested Reading Order:**

```
1. This overview (you are here!)
   ↓ Get big picture

2. Complete Setup Checklist
   ↓ Follow step-by-step

3. CRUISE_SPEED Explained
   ↓ Understand speed control

4. RC Mode Switching Guide
   ↓ Configure your transmitter

5. Steering Concept for Tanks
   ↓ Understand differential control

6. Parameter Reference
   ↓ Deep dive into all parameters

7. Official ArduPilot docs
   ↓ Advanced topics
```

---

## 🎯 Success Criteria

### **You Know Setup is Complete When:**

```
✓ Rover arms without errors
✓ Manual mode: You can drive forward/back/turn
✓ Hold mode: Rover stops immediately
✓ Mode switch: All positions work correctly
✓ GPS: Shows 3D fix, 10+ satellites (if equipped)
✓ Home: Set to current location
✓ Auto mode: Follows simple waypoint mission
✓ RTL mode: Returns to home automatically
✓ Failsafe: Stops when RC off
✓ Tuning: Drives smoothly without oscillation
```

**If all above check out: Your tank is fully operational! ✅**

---

## ⚠️ Safety Reminders

```
ALWAYS:
✓ Test in open, safe area first
✓ Start in Manual mode
✓ Have Hold mode easily accessible
✓ Keep hands on transmitter
✓ Be ready to disarm
✓ Start with low throttle limits
✓ Increase speed gradually
✓ Test failsafe before autonomous

NEVER:
✗ Drive in crowded areas initially
✗ Start in Auto mode
✗ Ignore pre-arm warnings
✗ Skip calibrations
✗ Test at full speed first
✗ Drive without checking motor directions
✗ Use autonomous without GPS testing
```

---

## 🏆 Final Checklist

```
Before Declaring Victory:

Hardware:
□ All components connected correctly
□ No loose wires
□ Battery secure
□ Motors mounted firmly

Software:
□ Latest stable firmware
□ All parameters configured
□ All calibrations complete
□ Modes assigned and tested

Testing:
□ Bench test passed
□ Motor direction verified
□ Manual mode driving smooth
□ Autonomous modes tested (if using)
□ Failsafe tested and works

Documentation:
□ Parameters backed up to file
□ Setup notes written
□ Known issues documented
□ Tuning values recorded

Ready to Operate:
□ Comfortable driving manually
□ Understand all configured modes
□ Know emergency procedures
□ Have support contact (forums/discord)

🎉 CONGRATULATIONS! Your tank is operational! 🎉
```

---

## 📞 Getting Help

**If you get stuck:**

1. **Check the detailed guides** in this docs folder
2. **Search the forum:** https://discuss.ardupilot.org/
   - Search for: "tank", "skid steering", "differential"
3. **Ask on Discord:** https://ardupilot.org/discord
4. **Post logs:** Download and share .bin logs for analysis
5. **Be specific:** Include parameter file, logs, video if possible

**Community is very helpful!** Don't hesitate to ask.

---

## 📖 Document Index

**All Tank Setup Documents:**

1. **TANK_SETUP_OVERVIEW.md** (this file)
   - Quick reference
   - Links to all other docs
   - High-level overview

2. **TANK_COMPLETE_SETUP_CHECKLIST.md**
   - Detailed step-by-step
   - All parameters
   - Testing procedures

3. **CRUISE_SPEED_EXPLAINED.md**
   - What cruise speed is
   - How it works
   - Where it's used
   - Tuning guide

4. **RC_MODE_SWITCHING_GUIDE.md**
   - How mode switching works
   - Transmitter setup
   - Mode assignment
   - Testing procedures

5. **STEERING_CONCEPT_FOR_TANKS.md**
   - Why "steering" applies to tanks
   - Differential thrust explained
   - Parameter meanings
   - Tuning for tanks

6. **ROVER_PARAMETERS_REFERENCE.md** (root folder)
   - All parameters
   - Descriptions
   - Rover-specific annotations

---

## ⭐ Quick Command Reference

```bash
# Build firmware
cd /home/dmitry/workspace/src/ardupilot
./waf configure --board speedybeef4v3
./waf rover

# Firmware location
build/speedybeef4v3/bin/ardurover.apj

# List all boards
./waf list_boards | grep -i speedybee

# Clean build
./waf distclean

# Upload directly (if FC connected)
./waf --upload rover
```

---

## 🎯 Mission Accomplished!

You now have:
- ✅ Complete understanding of tank configuration
- ✅ All parameters explained
- ✅ Detailed setup guides
- ✅ Troubleshooting resources
- ✅ Community support links

**Time to build and drive your autonomous tank! Good luck! 🚜💨**

---

**Document Version:** 1.0  
**Last Updated:** 2025-10-10  
**Maintained by:** ArduPilot Community  
**License:** GPL v3

