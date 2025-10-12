# Understanding "STEERING" in Tank/Skid-Steer Vehicles
## Why Tanks Have Steering Parameters (and What They Really Mean)

**Document:** Steering concept explained for differential/skid-steer vehicles  
**Target Audience:** Tank/skid-steer rover operators  
**Common Confusion:** "My tank has no steering servo, why steering parameters?"  
**Date:** 2025-10-10

---

## 🎯 The Core Confusion

### **What People Think:**

```
"Steering" = Physical steering servo/wheel
            = Ackermann mechanism
            = Like a car's steering wheel

Therefore: "My tank has no steering servo, so steering parameters don't apply to me"
```

### **The Reality:**

```
"Steering" in ArduRover = DIRECTIONAL CONTROL
                        = YAW RATE CONTROL
                        = "How fast/hard to turn"

Therefore: "ALL vehicles that turn use steering control, 
            regardless of the physical mechanism!"
```

---

## 🔑 Key Concept: Generic vs Implementation

### **ArduRover's Brilliant Architecture:**

```
┌─────────────────────────────────────────────────────┐
│         HIGH-LEVEL CONTROL (Generic)                 │
│                                                      │
│  "Turn right at 90 degrees per second"              │
│                                                      │
│  Uses: ATC_STR_* parameters                         │
│  • ATC_STR_RAT_P  (How aggressively)                │
│  • ATC_STR_RAT_MAX (How fast max)                   │
│  • etc.                                             │
│                                                      │
│  This layer is IDENTICAL for all vehicles!          │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │   WHAT to do (turn right)  │
        │   HOW FAST (90 deg/s)      │
        │   HOW HARD (P/I/D gains)   │
        └────────────┬───────────────┘
                     │
     ┌───────────────┼────────────────┐
     │               │                │
     ▼               ▼                ▼
┌──────────┐  ┌───────────┐  ┌──────────────┐
│Ackermann │  │   TANK    │  │     OMNI     │
│  (Car)   │  │ (Skid-St) │  │   (Holonomic)│
└────┬─────┘  └─────┬─────┘  └──────┬───────┘
     │              │                │
     ▼              ▼                ▼
┌──────────┐  ┌───────────┐  ┌──────────────┐
│ HOW to   │  │  HOW to   │  │   HOW to     │
│implement:│  │implement: │  │ implement:   │
│          │  │           │  │              │
│Turn      │  │Differential│  │Vector        │
│steering  │  │thrust     │  │math on       │
│servo     │  │on tracks  │  │4 wheels      │
└──────────┘  └───────────┘  └──────────────┘
```

**Same "steering" parameters, different implementation!**

---

## 🚗 Example 1: Ackermann Car (Physical Steering)

### **How Steering Works:**

```
High-level command: "Turn right 90 deg/s"
                    ↓
ATC_STR controller: Calculates needed servo angle
                    ↓
AR_Motors output:   SERVO1 (GroundSteering) = 30°
                    ↓
Physical servo:     Turns wheels 30° right
                    ↓
Result:             Car turns right at ~90 deg/s
```

**Steering parameters control:**
- How quickly to reach desired angle
- Maximum servo movement
- Response characteristics

---

## 🛞 Example 2: TANK (Your Vehicle - Differential Thrust)

### **How "Steering" Works:**

```
High-level command: "Turn right 90 deg/s"
                    ↓
ATC_STR controller: Calculates turn effort needed
                    ↓
AR_Motors output:   Left motors = 70%
                    Right motors = 30%
                    ↓
Physical motors:    Left tracks spin faster
                    Right tracks spin slower
                    ↓
Result:             Tank turns right at ~90 deg/s
```

**Same steering parameters control:**
- How aggressively to create differential
- Maximum turn rate
- Response characteristics

**NO STEERING SERVO, but steering control exists via differential!**

---

## 🤖 Example 3: Omni-Directional Robot

### **How "Steering" Works:**

```
High-level command: "Turn right 90 deg/s"
                    ↓
ATC_STR controller: Calculates rotation needed
                    ↓
AR_Motors output:   Motor1 = +30% (steering factor)
                    Motor2 = -30%
                    Motor3 = +30%
                    Motor4 = -30%
                    ↓
Physical wheels:    Each wheel contributes to rotation
                    Via vector math and mixing
                    ↓
Result:             Robot rotates right at ~90 deg/s
```

**Same steering parameters control:**
- How much to rotate
- Maximum rotation rate  
- Response characteristics

---

## 📊 What ATC_STR_RAT_P Really Controls

### **ATC_STR_RAT_P = Steering Rate P Gain**

**NOT:**
- ❌ Steering servo angle gain
- ❌ Steering wheel sensitivity
- ❌ Physical steering mechanism gain

**BUT:**
- ✅ **Yaw rate error correction gain**
- ✅ **Turn aggressiveness**
- ✅ **How hard to try to achieve desired turn rate**

---

### **For Your Tank Specifically:**

```
Desired turn rate: 90 deg/s (turn right)
Actual turn rate:  60 deg/s (measured by gyro)
Error: 90 - 60 = 30 deg/s

ATC_STR_RAT_P = 0.2:
  Correction = 0.2 × 30 = 6
  Apply 6% more differential
  
  Left motors:  65% → 71%  }
  Right motors: 35% → 29%  } Increased differential
  
  Result: Turn rate increases toward 90 deg/s

ATC_STR_RAT_P = 0.5:
  Correction = 0.5 × 30 = 15
  Apply 15% more differential
  
  Left motors:  65% → 80%  }
  Right motors: 35% → 20%  } Large differential
  
  Result: Very aggressive turn, might oscillate
```

**Higher P = More aggressive differential = Sharper turns**  
**Lower P = Gentler differential = Smoother turns**

---

## 🔄 The Complete Tank Steering Flow

### **From RC Stick to Track Movement:**

```
┌─────────────────────────────────────────────────────┐
│ STEP 1: RC Input                                     │
│ Pilot moves steering stick RIGHT 50%                │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ STEP 2: Mode Processing                              │
│ Manual mode reads: "Steering = +2250" (half of 4500)│
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ STEP 3: Convert to Turn Rate                        │
│ calc_steering_from_lateral_acceleration()           │
│ Desired turn rate = 90 deg/s (50% of max)          │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ STEP 4: Steering Rate Controller (PID)              │
│ Target: 90 deg/s                                    │
│ Actual: 85 deg/s (from gyro)                        │
│ Error: 5 deg/s                                      │
│                                                      │
│ P term: 0.2 × 5 = 1.0                               │
│ I term: 0.1 (accumulated)                           │
│ Output: 1.1 (steering effort)                       │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ STEP 5: Convert to Motor Differential               │
│ output_skid_steering()                              │
│                                                      │
│ Base throttle = 50% (from throttle stick)           │
│ Steering effort = 1.1                               │
│ Mix factor = MOT_STR_THR_MIX = 0.5                  │
│                                                      │
│ Left  = 50% + (1.1 × 0.5) = 50.55%                  │
│ Right = 50% - (1.1 × 0.5) = 49.45%                  │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ STEP 6: Output to Motors                            │
│ SERVO1 & SERVO2 (ThrottleLeft)  = 50.55%           │
│ SERVO3 & SERVO4 (ThrottleRight) = 49.45%           │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│ STEP 7: Physical Result                             │
│ Left tracks:  Spin at 50.55% power                  │
│ Right tracks: Spin at 49.45% power                  │
│ Differential creates turn                           │
│ Tank turns right                                    │
└─────────────────────────────────────────────────────┘
```

**Every step uses "steering" terminology, but physical implementation is DIFFERENTIAL THRUST!**

---

## 🧠 Why This Architecture Makes Sense

### **Benefit 1: Code Reuse**

```
All mode code is generic:

void ModeAuto::update() {
    navigate_to_waypoint();  // Same for all
    calc_steering_to_heading(desired_heading);  // Same for all
    calc_throttle(target_speed, avoidance_enabled);  // Same for all
}

Works for:
✓ Car (Ackermann)
✓ Tank (Skid-steer)
✓ Omni robot
✓ Boat
✓ Balance bot

No duplication needed!
```

---

### **Benefit 2: Consistent Parameters**

```
Same tuning parameters across all vehicle types:

ATC_STR_RAT_P     # Works for cars, tanks, boats, omni
ATC_STR_RAT_MAX   # Works for all
ATC_STR_ANG_P     # Works for all

Learn once, apply everywhere!
```

---

### **Benefit 3: Easy Vehicle Type Changes**

```
Want to convert from car to tank?

Old (Ackermann):
SERVO1_FUNCTION = 26  (GroundSteering)
SERVO3_FUNCTION = 70  (Throttle)

New (Tank):
SERVO1_FUNCTION = 73  (ThrottleLeft)
SERVO3_FUNCTION = 74  (ThrottleRight)

That's it! All steering parameters still work, just different output!
```

---

## 📖 Terminology Translation

### **When You See This Parameter:**

| **Parameter Name** | **Generic Meaning** | **For Ackermann Car** | **For Your Tank** | **For Omni Robot** |
|-------------------|-------------------|---------------------|------------------|-------------------|
| **ATC_STR_RAT_P** | "Turn effort gain" | Servo angle response | Differential aggressiveness | Rotation speed gain |
| **ATC_STR_RAT_MAX** | "Max turn rate" | Max servo rate | Max differential (pivot) | Max rotation rate |
| **ATC_STR_RAT_IMAX** | "Max integral" | Integral limit | Integral limit | Integral limit |
| **ATC_STR_ANG_P** | "Heading correction" | Turn to heading | Differential to heading | Rotation to heading |

**Always think: "This controls turning behavior, not physical steering mechanism"**

---

## 🔬 Deep Dive: Differential Steering Math

### **How Differential Creates Turns:**

```
Base scenario: Both sides at 50% throttle
┌─────────────────────────┐
│ Left tracks:  50%       │
│ Right tracks: 50%       │
│ Result: Straight ahead  │
└─────────────────────────┘

Add steering input: Turn right
┌─────────────────────────────────────────┐
│ Steering effort from controller: 20     │
│ MOT_STR_THR_MIX = 0.5                   │
│                                         │
│ Differential = 20 × 0.5 = 10%           │
│                                         │
│ Left tracks:  50% + 10% = 60%           │
│ Right tracks: 50% - 10% = 40%           │
│                                         │
│ Difference: 20% (creates turning force) │
│ Result: Turn right                      │
└─────────────────────────────────────────┘

Larger steering effort = Larger differential = Tighter turn
```

---

### **Extreme Case: Pivot Turn**

```
Throttle: 0% (stopped)
Steering: Full right (maximum)
┌─────────────────────────────────────────┐
│ Steering effort: 100                    │
│ MOT_STR_THR_MIX = 0.5                   │
│                                         │
│ Differential = 100 × 0.5 = 50%          │
│                                         │
│ Left tracks:  0% + 50% = +50% ⭐         │
│ Right tracks: 0% - 50% = -50% ⭐         │
│                                         │
│ Left forward, right REVERSE             │
│ Result: Pivot turn in place!            │
└─────────────────────────────────────────┘

This is why tanks can spin without moving forward!
```

---

## 📊 Parameter Effect Comparison

### **Effect of ATC_STR_RAT_P on Different Vehicles:**

#### **Ackermann Car (SERVO1_FUNCTION = 26):**

```
Low P (0.1):
  Turn command → Servo moves slowly
  Gentle steering angle changes
  Smooth but slow response

High P (0.5):
  Turn command → Servo moves quickly
  Rapid steering angle changes
  Fast but might overshoot
```

#### **Tank (SERVO1/3_FUNCTION = 73/74):**

```
Low P (0.1):
  Turn command → Small track speed differential
  Gentle turns
  Wide turning radius

High P (0.5):
  Turn command → Large track speed differential
  Sharp turns
  Tight turning radius (good for pivot)
  Might oscillate if too high
```

#### **Omni Robot (SERVO1-4_FUNCTION = 33-36):**

```
Low P (0.1):
  Turn command → Small motor mixing
  Slow rotation
  Smooth but sluggish

High P (0.5):
  Turn command → Aggressive motor mixing
  Fast rotation
  Quick but might overshoot
```

**Same parameter, different physical result!**

---

## 🎛️ MOT_STR_THR_MIX Explained (Tank-Specific!)

### **What It Does:**

**Controls the balance between steering and throttle in skid-steering vehicles.**

```
MOT_STR_THR_MIX = Steering vs Throttle Priority

Range: 0.0 to 1.0
Default: 0.5 (balanced)
```

---

### **Effect on Your Tank:**

```
Scenario: Throttle = 80%, Steering = Right (need 30% differential)

MOT_STR_THR_MIX = 0.0 (Throttle priority):
┌────────────────────────────────────┐
│ Differential reduced to fit        │
│ Left:  80% + 10% = 90%             │
│ Right: 80% - 10% = 70%             │
│                                    │
│ Result: Wide turn, maintains speed │
└────────────────────────────────────┘

MOT_STR_THR_MIX = 0.5 (Balanced):
┌────────────────────────────────────┐
│ Split the difference               │
│ Left:  80% + 15% = 95%             │
│ Right: 80% - 15% = 65%             │
│                                    │
│ Result: Medium turn, slight slow   │
└────────────────────────────────────┘

MOT_STR_THR_MIX = 1.0 (Steering priority):
┌────────────────────────────────────┐
│ Full differential, reduce throttle │
│ Left:  65% + 30% = 95%             │
│ Right: 65% - 30% = 35%             │
│                                    │
│ Result: Tight turn, slows down     │
└────────────────────────────────────┘
```

**Use case:**
- **0.5-0.7:** Balanced (most common)
- **0.7-1.0:** Tight turns, willing to slow down
- **0.0-0.4:** Speed priority, wide turns

---

## 🔧 Tuning Steering for Your Tank

### **Symptom: Turns too slowly**

**Problem:**
```
You command right turn
Tank barely turns
Takes forever to reach heading
```

**Solution:**
```
Increase steering aggressiveness:
ATC_STR_RAT_P = 0.3        # From 0.2
ATC_STR_RAT_MAX = 270      # From 180

Result: More aggressive differential, faster turns
```

---

### **Symptom: Turns too aggressively / Oscillates**

**Problem:**
```
Command straight ahead
Tank wobbles left-right
Can't maintain straight line
Overshoots heading
```

**Solution:**
```
Decrease steering aggressiveness:
ATC_STR_RAT_P = 0.1        # From 0.2
ATC_STR_RAT_D = 0.01       # Add damping
ATC_STR_RAT_FILT = 5       # From 10 (more filtering)

Result: Gentler differential, smoother motion
```

---

### **Symptom: Can't make tight turns**

**Problem:**
```
Need to turn around in confined space
Tank makes wide turns
Can't pivot in place
```

**Solution:**
```
Allow more aggressive turns:
ATC_STR_RAT_MAX = 360      # From 180
MOT_STR_THR_MIX = 0.8      # From 0.5 (prioritize steering)

Result: Can pivot turn (one track forward, one reverse)
```

---

### **Symptom: Slows down too much in turns**

**Problem:**
```
Tank cruising at 2 m/s
Starts turn
Speed drops to 1 m/s
Loses momentum
```

**Solution:**
```
Reduce steering priority:
MOT_STR_THR_MIX = 0.3      # From 0.5 (prioritize throttle)

Result: Wider turns but maintains speed better
```

---

## 📐 Mathematical Explanation

### **The Skid-Steer Mixing Formula:**

```cpp
// Simplified from AR_Motors::output_skid_steering()

float base_throttle = 50.0;      // From throttle stick
float steering_demand = 30.0;     // From steering controller
float mix = MOT_STR_THR_MIX;     // 0.5

// Calculate differential
float steering_output = steering_demand * mix;  // 30 × 0.5 = 15

// Apply to motors
float left  = base_throttle + steering_output;  // 50 + 15 = 65%
float right = base_throttle - steering_output;  // 50 - 15 = 35%

// Constrain to valid range
left  = constrain(left, -100, 100);
right = constrain(right, -100, 100);

// Send to motors
SERVO1 & SERVO2 = left;   // 65% (left tracks)
SERVO3 & SERVO4 = right;  // 35% (right tracks)

// Physical result: Turn right!
```

---

## 🎯 Real-World Examples

### **Example 1: Straight Line Navigation**

```
Auto mode: Navigate straight to waypoint

Target heading: 90° (East)
Actual heading: 92° (slightly off)
Error: -2°

Steering controller:
  ATC_STR_ANG_P = 2.0
  Desired turn rate = 2.0 × (-2°) = -4 deg/s (turn left slightly)

Rate controller:
  Target rate: -4 deg/s
  Actual rate: 0 deg/s
  Error: -4 deg/s
  
  ATC_STR_RAT_P = 0.2
  Output: 0.2 × (-4) = -0.8 (small left correction)

Motor output:
  Throttle: 50%
  Steering: -0.8
  
  Left:  50% - 0.4% = 49.6%
  Right: 50% + 0.4% = 50.4%

Result: 
  Gentle correction
  Returns to 90° heading
  Maintains straight path
```

---

### **Example 2: Sharp Corner in Mission**

```
Auto mode: Turn from North (0°) to East (90°)

Initial:
  Heading: 0° (North)
  Target: 90° (East)
  Error: 90°

Steering controller:
  ATC_STR_ANG_P = 2.0
  Desired rate = 2.0 × 90° = 180 deg/s (max allowed)

Rate controller:
  Target: 180 deg/s
  Actual: 150 deg/s (building up)
  Error: 30 deg/s
  
  ATC_STR_RAT_P = 0.2
  Output: 0.2 × 30 = 6.0 (strong right turn)

Motor output:
  Throttle: 50%
  Steering: +6.0
  MOT_STR_THR_MIX = 0.5
  
  Differential = 6.0 × 0.5 = 3.0
  
  Left:  50% + 3% = 53%
  Right: 50% - 3% = 47%

As turn progresses:
  Heading approaches 90°
  Error decreases
  Steering effort reduces
  Differential reduces
  Settles on straight path East
```

---

## 💡 Key Insights

### **Insight 1: "Steering" = Abstract Concept**

```
In ArduRover source code, you'll see:

set_steering(float steering_value);
calc_steering_to_heading(float desired_heading);
get_steering();

These are GENERIC functions that work for ALL vehicles.

The implementation details are hidden in AR_Motors:
- For car → Convert to servo angle
- For tank → Convert to differential thrust
- For omni → Convert to motor mixing
```

---

### **Insight 2: Same Controller, Different Actuators**

```
The PID controller (ATC_STR_RAT_*) doesn't know or care
about physical steering mechanism.

It just knows:
"I want X deg/s turn rate, how much effort to apply?"

AR_Motors translates that effort:
Car:  "Move servo to angle Y"
Tank: "Create differential Z between tracks"
Omni: "Apply rotation matrix M to 4 motors"
```

---

### **Insight 3: Why Parameters Transfer**

```
If you tune a car and get good steering:
ATC_STR_RAT_P = 0.25
ATC_STR_RAT_I = 0.20

Those SAME values might work well for a tank!

Because they control:
• Aggressiveness of turn response
• Integral buildup rate
• Damping characteristics

Not specific to servo vs differential
```

---

## ❓ Common Questions

### **Q: My tank has no steering servo. Can I delete ATC_STR_* parameters?**

**A:** **NO!** Those parameters control your tank's turning behavior via differential thrust. Without them, your tank can't turn properly in autonomous modes!

---

### **Q: Should ATC_STR_* values be different for tank vs car?**

**A:** Usually similar, but may need tuning:
- Tanks often need higher `ATC_STR_RAT_MAX` (360 vs 180)
- Tanks might need lower `ATC_STR_RAT_P` if very heavy
- But start with defaults and tune from there

---

### **Q: What's the difference between "steering" and "throttle" for a tank?**

**A:** 
- **Throttle:** Average power to both sides (forward/backward speed)
- **Steering:** Difference in power between sides (turning)

```
Example:
Throttle = 50%, Steering = 0
  → Left 50%, Right 50% = Straight

Throttle = 50%, Steering = Right
  → Left 70%, Right 30% = Turn right

Throttle = 0%, Steering = Right  
  → Left 50%, Right -50% = Pivot right
```

---

### **Q: Do I tune ATC_STR_* the same way as a car?**

**A:** Yes! The tuning process is identical:

```
1. Start with defaults
2. Test turns in Auto mode
3. Too sluggish? Increase P
4. Oscillates? Decrease P, add D
5. Steady-state error? Increase I
6. Repeat until good
```

---

## 🎓 Educational Examples

### **Example A: Why "Steering Rate" Makes Sense**

```
Ackermann Car:
  Steering servo rotates at X degrees/second
  This is the "steering rate"

Tank:
  Differential creates rotation at Y degrees/second
  This is ALSO a "steering rate"
  
Both measured in deg/s
Both controlled by ATC_STR_RAT_*
Same concept, different mechanism!
```

---

### **Example B: Heading Hold**

```
Task: Drive straight North (0°)
Current heading: 2° (drifted right)

For Car:
  Controller: "Turn left 4 deg/s"
  Output: Steering servo 2° left
  Result: Corrects to 0°

For Tank:
  Controller: "Turn left 4 deg/s"
  Output: Left 49%, Right 51% (slight differential)
  Result: Corrects to 0°

Same controller output, different actuation!
```

---

## 📝 Glossary

**Steering (Generic Term):**
- Directional control
- Turning behavior
- Yaw rate management
- **NOT:** Physical steering mechanism

**Differential Steering:**
- Skid-steer method
- Left/right speed difference creates turns
- Used by tanks, tracked vehicles, dual-motor rovers

**Ackermann Steering:**
- Traditional car-like steering
- Front wheels pivot
- Rear wheels driven
- Needs steering servo

**Steering Rate:**
- How fast vehicle is turning
- Measured in degrees/second
- Same meaning for all vehicle types

**Steering Effort:**
- How hard controller is trying to turn
- Translates to: servo angle (car) or differential (tank)

---

## ✅ Summary

### **Key Takeaways:**

1. **"Steering" in ArduRover = Turning control (generic)**
   - Not specific to steering servos
   - Applies to all vehicles that turn

2. **Tanks DO have steering control**
   - Via differential thrust
   - Left/right track speed difference
   - No servo needed

3. **ATC_STR_* parameters ARE for tanks**
   - Control turn aggressiveness
   - Control maximum turn rate
   - Control response characteristics

4. **Same parameters, different implementation**
   - Car: Moves servo
   - Tank: Creates differential
   - Omni: Mixes 4 motors
   - All use same tuning values

5. **MOT_STR_THR_MIX is tank-specific**
   - Only exists for skid-steering
   - Balances turning vs speed
   - 0.5 = balanced, 1.0 = tight turns

---

## 🎯 For Your Tank

**Don't think:**
```
❌ "I have no steering servo, so ignore ATC_STR_* parameters"
```

**Instead think:**
```
✅ "ATC_STR_* controls my turning behavior via differential thrust"
✅ "Higher P = More aggressive track differential = Tighter turns"
✅ "ATC_STR_RAT_MAX = How fast I can pivot"
✅ "These parameters are CRITICAL for my tank!"
```

---

**The brilliance of ArduPilot's architecture:** One set of steering parameters works for cars, tanks, boats, and omni-robots. The abstraction layer handles the physical differences! 🎯

