# Understanding CRUISE_SPEED in ArduRover
## What It Is, How It Works, and Where It's Used

**Document:** CRUISE_SPEED parameter comprehensive guide  
**Applies to:** All ArduRover vehicles  
**Date:** 2025-10-10

---

## 🎯 What is CRUISE_SPEED?

### **Simple Definition:**

**CRUISE_SPEED is the target speed (in meters per second) that ArduRover tries to maintain during autonomous operation.**

Think of it like **cruise control in a car** - you set a desired speed, and the system automatically adjusts throttle to maintain it.

---

## 📊 Parameter Details

```
Parameter: CRUISE_SPEED
Units: m/s (meters per second)
Range: 0-100
Default: 2.0
Type: Float
Group: Navigation/Speed Control
```

**Related Parameters:**
```
CRUISE_THROTTLE  # Initial throttle guess to achieve CRUISE_SPEED
WP_SPEED         # Overrides CRUISE_SPEED for waypoints (if set)
RTL_SPEED        # Overrides CRUISE_SPEED for RTL mode (if set)
SPEED_MAX        # Absolute maximum speed limit
```

---

## 🔄 How CRUISE_SPEED Works

### **The Speed Control Loop:**

```
┌─────────────────────────────────────────────────────────┐
│ 1. You set: CRUISE_SPEED = 2.0 m/s                     │
│    "I want my tank to cruise at 2 m/s"                  │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Initial throttle guess: CRUISE_THROTTLE = 50%       │
│    "Start by applying 50% throttle"                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Rover drives and measures actual speed              │
│    GPS/Wheel encoders report: "Currently 1.8 m/s"      │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Speed controller calculates error                    │
│    Error = CRUISE_SPEED - Actual = 2.0 - 1.8 = 0.2 m/s │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 5. PID controller adjusts throttle                      │
│    P term: ATC_SPEED_P × error = 0.2 × 0.2 = 0.04      │
│    I term: Accumulated error over time                  │
│    New throttle = 50% + 4% = 54%                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Speed increases to 2.0 m/s                          │
│    Controller maintains this by adjusting throttle      │
│    Uphill? Increases throttle                          │
│    Downhill? Decreases throttle                        │
└─────────────────────────────────────────────────────────┘
```

---

## 📍 Where CRUISE_SPEED is Used

### **Mode-by-Mode Breakdown:**

| **Mode** | **Uses CRUISE_SPEED?** | **How It's Used** |
|----------|----------------------|-------------------|
| **Manual** | ❌ **NO** | Direct throttle control - stick position = motor power |
| **Acro** | ✅ **YES** | Maintains speed while you control turn rate |
| **Steering** | ✅ **YES** | Throttle stick sets speed as % of CRUISE_SPEED |
| **Hold** | ❌ **NO** | Speed = 0 (stopped) |
| **Loiter** | ✅ **YES** | Drives to loiter point at CRUISE_SPEED, then holds |
| **Follow** | ✅ **YES** | Follows target vehicle at CRUISE_SPEED |
| **Simple** | ✅ **YES** | Similar to Steering mode |
| **Dock** | ✅ **YES** | Approaches dock, slowing as it gets closer |
| **Circle** | ✅ **YES** | Circles at CRUISE_SPEED (or CIRC_SPEED if set) |
| **Auto** | ✅ **YES** | Follows waypoints at CRUISE_SPEED (or WP_SPEED) |
| **Guided** | ✅ **YES** | Goes to commanded location at CRUISE_SPEED |
| **RTL** | ✅ **YES*** | Uses RTL_SPEED if set, otherwise CRUISE_SPEED |
| **SmartRTL** | ✅ **YES*** | Uses RTL_SPEED if set, otherwise CRUISE_SPEED |

---

## 🔬 Detailed Mode Examples

### **Example 1: Manual Mode (NO cruise speed)**

```
Mode: Manual
CRUISE_SPEED = 2.0  ← IGNORED in Manual mode

Your RC Input → Direct to Motors:
Throttle stick at 50% → Motors get 50% power
Throttle stick at 75% → Motors get 75% power
Throttle stick at 100% → Motors get 100% power

Actual speed depends on:
• Terrain (flat, uphill, downhill)
• Surface (pavement, grass, sand)
• Battery voltage
• Load

You control power, not speed
```

---

### **Example 2: Steering Mode (Cruise speed as reference)**

```
Mode: Steering
CRUISE_SPEED = 2.0 m/s

Your RC Input → Target Speed:
Throttle stick at 0% → Target = 0 m/s (stopped)
Throttle stick at 50% → Target = 1.0 m/s (50% of cruise)
Throttle stick at 100% → Target = 2.0 m/s (100% of cruise)

Controller adjusts throttle automatically:
Going uphill → Throttle increases to 70%
Going downhill → Throttle decreases to 30%
On flat → Throttle settles around CRUISE_THROTTLE

You control speed setpoint, controller manages power
```

---

### **Example 3: Auto Mode (Full cruise speed control)**

```
Mode: Auto (waypoint mission)
CRUISE_SPEED = 2.0 m/s
WP_SPEED = 0 (not set, use CRUISE_SPEED)

Autonomous Operation:
1. Tank starts mission
2. Drives to waypoint 1 at 2.0 m/s
   • Terrain changes don't affect speed
   • Controller constantly adjusts throttle
   • Speed stays at 2.0 m/s ± 0.1 m/s

3. Reaches waypoint 1
4. Turns toward waypoint 2
5. Continues at 2.0 m/s
6. Repeats for all waypoints

You set the speed once, rover maintains it automatically
```

---

### **Example 4: RTL Mode (Special speed handling)**

```
Mode: RTL (Return to Launch)
CRUISE_SPEED = 2.0 m/s
RTL_SPEED = 1.5 m/s  ← If set, takes priority

Speed used: 1.5 m/s (RTL_SPEED)

If RTL_SPEED = 0 (not set):
Speed used: 2.0 m/s (falls back to CRUISE_SPEED)

Why have separate RTL_SPEED?
• You might want slower return for safety
• Battery saving
• Easier to stop if needed
```

---

## 🎚️ Speed Hierarchy (Priority Order)

### **Which speed parameter takes precedence?**

```
1. Mission DO_CHANGE_SPEED command
   ↓ (if not in mission)
   
2. Mode-specific speed (WP_SPEED, RTL_SPEED, CIRC_SPEED)
   ↓ (if not set or = 0)
   
3. CRUISE_SPEED ⭐ DEFAULT FALLBACK
   ↓
   
4. SPEED_MAX (hard limit, never exceed)
```

### **Example:**

```ini
CRUISE_SPEED = 2.0      # Default: 2 m/s
WP_SPEED = 3.0          # Waypoints: 3 m/s (overrides cruise)
RTL_SPEED = 1.5         # RTL: 1.5 m/s (overrides cruise)
SPEED_MAX = 5.0         # Never exceed 5 m/s

Result:
• Auto mode → 3.0 m/s (uses WP_SPEED)
• RTL mode → 1.5 m/s (uses RTL_SPEED)
• Loiter mode → 2.0 m/s (uses CRUISE_SPEED)
• All modes limited to 5.0 m/s maximum
```

---

## 🧮 Calculating Appropriate CRUISE_SPEED

### **Conversion Table:**

| **m/s** | **km/h** | **mph** | **Use Case** |
|---------|----------|---------|--------------|
| 0.5 | 1.8 | 1.1 | Very slow, precision work |
| 1.0 | 3.6 | 2.2 | Slow, careful navigation |
| 1.5 | 5.4 | 3.4 | Conservative autonomous |
| 2.0 | 7.2 | 4.5 | **Standard cruise ⭐** |
| 3.0 | 10.8 | 6.7 | Fast transit |
| 5.0 | 18.0 | 11.2 | High-speed (careful!) |

### **How to Choose:**

```
Small RC car (1:10 scale):
CRUISE_SPEED = 1.0-2.0 m/s

Medium rover (1:5 scale):
CRUISE_SPEED = 2.0-3.0 m/s

Large rover (full size):
CRUISE_SPEED = 1.5-2.5 m/s  # Safety first!

Tank (your setup):
CRUISE_SPEED = 1.5-2.5 m/s  # Start at 1.5, increase to 2.5
```

---

## 🔧 Tuning CRUISE_SPEED & CRUISE_THROTTLE

### **Initial Setup Process:**

```
Step 1: Set conservative values
┌────────────────────────────────────┐
│ CRUISE_SPEED = 1.5                 │
│ CRUISE_THROTTLE = 40               │
└────────────────────────────────────┘

Step 2: Drive in Auto or Steering mode

Step 3: Observe in Mission Planner HUD:
┌────────────────────────────────────┐
│ Ground Speed: 1.48 m/s             │  ← Actual speed
│ Target Speed: 1.50 m/s             │  ← From CRUISE_SPEED
│ Throttle Out: 47%                  │  ← Controller output
└────────────────────────────────────┘

Step 4: If speed is stable and close to target:
• Note the throttle percentage (e.g., 47%)
• Update CRUISE_THROTTLE = 47
• This becomes new starting point

Step 5: If you want faster:
• Increase CRUISE_SPEED = 2.0
• Controller will adjust

Step 6: Monitor and fine-tune
• Watch actual vs target speed
• Adjust ATC_SPEED_P if response too slow/fast
```

---

## 📈 Real-World Scenarios

### **Scenario 1: Flat Ground**

```
Setup:
CRUISE_SPEED = 2.0 m/s
CRUISE_THROTTLE = 50%

Result:
Time 0s: Apply 50% throttle
Time 2s: Speed reaches 2.0 m/s
Time 10s: Throttle settles at 48% (slightly less needed)
Time 20s: Maintains 2.0 m/s at 48% throttle

Constant speed, throttle adjusts slightly
```

---

### **Scenario 2: Variable Terrain**

```
Setup:
CRUISE_SPEED = 2.0 m/s
CRUISE_THROTTLE = 50%

Driving sequence:
┌────────────────────────────────────────────┐
│ Flat ground:                               │
│   Speed: 2.0 m/s, Throttle: 50%            │
│                                            │
│ Start climbing hill:                       │
│   Speed drops to 1.7 m/s                   │
│   Controller sees error                    │
│   Throttle increases to 65%                │
│   Speed recovers to 2.0 m/s                │
│                                            │
│ Reach hilltop, start descending:           │
│   Speed increases to 2.3 m/s               │
│   Controller sees positive error           │
│   Throttle decreases to 35%                │
│   Speed returns to 2.0 m/s                 │
│                                            │
│ Back to flat:                              │
│   Throttle returns to ~50%                 │
│   Speed stable at 2.0 m/s                  │
└────────────────────────────────────────────┘

Speed remains constant, throttle varies
```

---

### **Scenario 3: Mission with Speed Changes**

```
Mission commands:
1. NAV_WAYPOINT (Waypoint 1)
2. DO_CHANGE_SPEED 1.0    ← Slow down
3. NAV_WAYPOINT (Waypoint 2 - precision area)
4. DO_CHANGE_SPEED 3.0    ← Speed up
5. NAV_WAYPOINT (Waypoint 3 - fast transit)
6. RTL                     ← Return at RTL_SPEED

Execution:
• WP1: Travels at CRUISE_SPEED (2.0 m/s)
• After DO_CHANGE_SPEED: Travels at 1.0 m/s
• WP2: Slow precision navigation
• After DO_CHANGE_SPEED: Travels at 3.0 m/s
• WP3: Fast transit
• RTL: Returns at RTL_SPEED (1.5 m/s)
```

---

## 🎮 Mode-Specific Behavior

### **Manual Mode:**
```
CRUISE_SPEED: IGNORED ❌

Throttle control:
• RC stick position directly controls motor power
• 0% stick = 0% motors
• 50% stick = 50% motors
• 100% stick = 100% motors

Actual speed varies with:
• Terrain (uphill/downhill)
• Surface (pavement/grass/mud)
• Battery voltage (decreases as battery drains)
• Load/weight

You control POWER, not SPEED
```

---

### **Steering Mode:**
```
CRUISE_SPEED: REFERENCE VALUE ✅

Throttle stick behavior:
• 0% stick = 0 m/s (stopped)
• 25% stick = 0.5 m/s (25% of CRUISE_SPEED)
• 50% stick = 1.0 m/s (50% of CRUISE_SPEED)
• 75% stick = 1.5 m/s (75% of CRUISE_SPEED)
• 100% stick = 2.0 m/s (100% of CRUISE_SPEED)

Controller maintains actual speed at stick percentage
Terrain changes compensated automatically

You control SPEED SETPOINT, controller manages power
```

---

### **Auto Mode:**
```
CRUISE_SPEED: FULLY ACTIVE ✅

Autonomous speed control:
• Target speed = CRUISE_SPEED (or WP_SPEED if set)
• RC throttle stick ignored (unless STICK_MIXING enabled)
• Controller maintains constant speed
• Throttle varies 20-80% to compensate terrain
• Very consistent speed regardless of conditions

Autopilot controls both SPEED and POWER
```

---

### **Loiter Mode:**
```
CRUISE_SPEED: APPROACH SPEED ✅

Behavior:
1. Calculate distance to loiter point
2. Drive toward point at CRUISE_SPEED
3. Slow down as approaching (based on LOIT_RADIUS)
4. Within LOIT_RADIUS (e.g., 2m):
   • Speed decreases
   • Gentle corrections to stay near point
5. At point: Speed = 0 (hold position)

CRUISE_SPEED used for approach, then slows to stop
```

---

### **RTL Mode:**
```
CRUISE_SPEED: FALLBACK VALUE ✅

Speed selection priority:
1. If RTL_SPEED > 0: Use RTL_SPEED
2. Else if WP_SPEED > 0: Use WP_SPEED
3. Else: Use CRUISE_SPEED ⭐

Example:
RTL_SPEED = 1.5 m/s  → Returns at 1.5 m/s
RTL_SPEED = 0        → Returns at CRUISE_SPEED

Why separate RTL_SPEED?
• Return home might need different speed
• Often slower for safety/battery conservation
```

---

## 🧪 Testing & Tuning Guide

### **Step 1: Initial Conservative Setup**

```
CRUISE_SPEED = 1.0         # Start slow
CRUISE_THROTTLE = 30       # Low throttle guess
ATC_SPEED_P = 0.2         # Default
```

---

### **Step 2: Test in Steering Mode**

```
1. Set mode to STEERING
2. ARM rover
3. Slowly push throttle to 50%
4. Watch Mission Planner HUD:
   ┌────────────────────────────────────┐
   │ Ground Speed: 0.95 m/s             │
   │ Target Speed: 1.00 m/s             │  ← 50% of CRUISE_SPEED
   │ Throttle: 35%                      │
   └────────────────────────────────────┘
5. Note actual throttle needed
```

---

### **Step 3: Adjust CRUISE_THROTTLE**

```
Observed: Need 35% throttle for 1.0 m/s

Update:
CRUISE_THROTTLE = 35       # Match observed value

This improves initial response speed
```

---

### **Step 4: Increase Speed (If Desired)**

```
Want to go faster?

CRUISE_SPEED = 2.0         # Double the speed
CRUISE_THROTTLE = 50       # Scale throttle proportionally

Test again:
• Should reach 2.0 m/s
• Throttle settles around 50% (may vary)
```

---

### **Step 5: Fine-Tune Speed Controller**

**If speed response is slow:**
```
ATC_SPEED_P = 0.3          # Increase from 0.2
ATC_SPEED_I = 0.3          # Increase from 0.2
```

**If speed oscillates:**
```
ATC_SPEED_P = 0.1          # Decrease from 0.2
ATC_SPEED_I = 0.1          # Decrease from 0.2
ATC_SPEED_FILT = 5         # Add filtering
```

---

## 📊 Typical Values for Different Vehicles

### **Small RC Car (1:10 scale):**
```
CRUISE_SPEED = 1.0-2.0
CRUISE_THROTTLE = 30-50
ATC_SPEED_P = 0.3
```

### **Medium Rover (1:5 scale):**
```
CRUISE_SPEED = 2.0-3.0
CRUISE_THROTTLE = 40-60
ATC_SPEED_P = 0.2
```

### **Large Rover/Tank (Full size, heavy):**
```
CRUISE_SPEED = 1.5-2.5
CRUISE_THROTTLE = 50-70
ATC_SPEED_P = 0.15
ATC_ACCEL_MAX = 1.0        # Limit acceleration
ATC_DECEL_MAX = 2.0        # Limit braking
```

### **Racing Rover (High speed):**
```
CRUISE_SPEED = 5.0-10.0
CRUISE_THROTTLE = 60-80
ATC_SPEED_P = 0.4          # Aggressive
ATC_ACCEL_MAX = 5.0        # Fast acceleration
```

---

## 💡 Advanced Tips

### **Tip 1: Cruise Learning**

**ArduRover can automatically learn optimal throttle:**

```
1. Drive in Auto or Steering mode
2. ArduRover observes:
   • What throttle achieves CRUISE_SPEED
   • Over many runs, learns optimal value
3. Automatically adjusts internal estimate
4. Gets better over time

You don't need to manually tune CRUISE_THROTTLE
Just set CRUISE_SPEED and drive!
```

---

### **Tip 2: Different Speeds for Different Situations**

```
# Conservative autonomous operation:
CRUISE_SPEED = 1.5         # Slow and safe
WP_SPEED = 0               # Use cruise speed
RTL_SPEED = 1.0            # Even slower return

# Performance operation:
CRUISE_SPEED = 3.0         # Fast cruise
WP_SPEED = 2.0             # Slower for waypoint precision
RTL_SPEED = 2.5            # Moderate return speed
```

---

### **Tip 3: Speed-Dependent Steering**

```
MOT_SPD_SCA_BASE = 1.0     # Speed above which steering scales

For Ackermann steering:
• At low speed: Full steering angle
• Above 1.0 m/s: Steering reduces proportionally
• Prevents flipping at high speed

For skid-steer (tank):
• Not used (skid-steer doesn't need this)
• Leave at default
```

---

### **Tip 4: Acceleration Limits**

**Smooth out speed changes:**

```
ATC_ACCEL_MAX = 2.0        # Max 2 m/s² acceleration
ATC_DECEL_MAX = 3.0        # Max 3 m/s² deceleration

Effect:
• Instead of instant speed change
• Ramps smoothly from current to target
• Gentler on mechanical components
• More comfortable operation
• Battery friendly

Set to 0 for no limits (instant response)
```

---

## 🎯 Quick Reference

### **Speed Control Parameters Summary:**

```
┌─────────────────────────────────────────────────────────┐
│ CRUISE_SPEED        What speed to maintain (m/s)       │
│ CRUISE_THROTTLE     Initial throttle guess (%)         │
│ WP_SPEED            Waypoint speed override (m/s)      │
│ RTL_SPEED           RTL speed override (m/s)           │
│ SPEED_MAX           Absolute maximum (m/s)             │
│                                                         │
│ ATC_SPEED_P         Speed controller P gain            │
│ ATC_SPEED_I         Speed controller I gain            │
│ ATC_SPEED_D         Speed controller D gain            │
│                                                         │
│ ATC_ACCEL_MAX       Max acceleration (m/s²)            │
│ ATC_DECEL_MAX       Max deceleration (m/s²)           │
└─────────────────────────────────────────────────────────┘
```

---

## ❓ Common Questions

### **Q: Do I need to set CRUISE_THROTTLE exactly?**

**A:** No! ArduRover learns the correct value automatically. CRUISE_THROTTLE is just the starting guess. Set it to something reasonable (40-60%) and let the controller learn.

---

### **Q: Can I change CRUISE_SPEED during operation?**

**A:** Yes! 
- Via Mission Planner: Change parameter and write
- Via mission command: DO_CHANGE_SPEED
- Via GCS command in Guided mode

---

### **Q: What if my tank can't reach CRUISE_SPEED?**

**A:** 
- Controller will apply maximum throttle
- Tank goes as fast as it can
- Lower CRUISE_SPEED to achievable value
- Check MOT_THR_MAX isn't limiting

---

### **Q: CRUISE_SPEED vs WP_SPEED - which to use?**

**A:**
- **CRUISE_SPEED:** General default for all autonomous modes
- **WP_SPEED:** Specific to waypoint navigation
- **Recommendation:** Set CRUISE_SPEED, leave WP_SPEED = 0

---

## 📝 Summary

**CRUISE_SPEED is:**
✅ Target speed for autonomous modes  
✅ Reference speed for Steering mode  
✅ Fallback if mode-specific speed not set  
✅ Maintained automatically by speed controller  
✅ Adjustable during operation  

**CRUISE_SPEED is NOT:**
❌ Used in Manual mode  
❌ Direct throttle percentage  
❌ Maximum speed (that's SPEED_MAX)  
❌ Fixed - controller adjusts throttle dynamically  

**For your tank:**
```
Start with: CRUISE_SPEED = 1.5 m/s
Test and tune
Increase to: 2.0-2.5 m/s as comfortable
```

**The beauty of CRUISE_SPEED:** Set it once, and your tank maintains that speed automatically regardless of terrain! 🎯

