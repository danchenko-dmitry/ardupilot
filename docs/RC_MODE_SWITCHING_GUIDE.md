# ArduRover RC Mode Switching Guide
## How to Switch Between Flight Modes Using Your RC Transmitter

**Document:** Complete guide to RC mode switching  
**Applies to:** All ArduRover vehicles  
**Date:** 2025-10-10

---

## 📋 TABLE OF CONTENTS

1. [What Are Flight Modes?](#what-are-flight-modes)
2. [How Mode Switching Works](#how-mode-switching-works)
3. [Configuration Steps](#configuration-steps)
4. [Transmitter Setup Examples](#transmitter-setup-examples)
5. [Verification & Testing](#verification--testing)
6. [Recommended Mode Assignments](#recommended-mode-assignments)
7. [Troubleshooting](#troubleshooting)

---

## 🎯 What Are Flight Modes?

### **ArduRover Flight Modes Overview:**

| **Mode** | **Number** | **Type** | **Description** |
|----------|-----------|----------|-----------------|
| **Manual** | 0 | Manual | Direct RC control, no stabilization |
| **Acro** | 1 | Manual | Rate-controlled steering |
| **Steering** | 3 | Manual | RC sets speed, autopilot maintains it |
| **Hold** | 4 | Auto | Stop and hold position |
| **Loiter** | 5 | Auto | Navigate to and hold GPS position |
| **Follow** | 6 | Auto | Follow another vehicle |
| **Simple** | 7 | Manual | Simplified directional control |
| **Dock** | 8 | Auto | Autonomous docking |
| **Circle** | 9 | Auto | Circle around a point |
| **Auto** | 10 | Auto | Follow waypoint mission |
| **RTL** | 11 | Auto | Return to launch point |
| **SmartRTL** | 12 | Auto | Return via recorded path |
| **Guided** | 15 | Auto | Accept commands from GCS |

---

## ⚙️ How Mode Switching Works

### **The Mechanism:**

```
RC Transmitter Switch
         ↓
Outputs PWM value on selected channel (e.g., Ch5)
         ↓
ArduRover reads PWM value
         ↓
Maps PWM range to mode number
         ↓
Switches to corresponding MODE1-6
         ↓
Vehicle behavior changes
```

### **PWM Range Mapping:**

```
PWM Range         → Mode Slot    → Assigned Mode
─────────────────────────────────────────────────
910  - 1230     →   MODE1      → (You assign: e.g., Manual)
1231 - 1360     →   MODE2      → (You assign: e.g., Hold)
1361 - 1490     →   MODE3      → (You assign: e.g., Steering)
1491 - 1620     →   MODE4      → (You assign: e.g., Auto)
1621 - 1749     →   MODE5      → (You assign: e.g., RTL)
1750 - 2049     →   MODE6      → (You assign: e.g., Guided)
2050+           →   MODE1      → (Wraps back to MODE1)
```

**Example:**
```
Your switch outputs 1425 µs
→ Falls in range 1361-1490
→ Activates MODE3
→ MODE3 = 3 (Steering)
→ Rover switches to Steering mode
```

---

## 🔧 Configuration Steps

### **Step 1: Choose Mode Channel**

**Select which RC channel controls modes:**

```
Mission Planner → CONFIG → Full Parameter List

┌────────────────────────────────────┐
│ MODE_CH = 5      # Channel 5       │
│                  # (can be 5-16)   │
└────────────────────────────────────┘

Write Parameters
```

**Common choices:**
- **Channel 5:** Most popular (3-pos aux switch)
- **Channel 6:** Alternative
- **Channel 8:** If Ch5/6 used for other functions

---

### **Step 2: Configure Your Transmitter**

**You need to create distinct PWM values for each mode position.**

#### **Option A: Single 6-Position Switch**

```
If you have a 6-position rotary switch:

1. Assign switch to Channel 5
2. Each position outputs different PWM
3. No mixing needed

Typical output:
Position 1: ~1100 µs → MODE1
Position 2: ~1300 µs → MODE2
Position 3: ~1425 µs → MODE3
Position 4: ~1550 µs → MODE4
Position 5: ~1700 µs → MODE5
Position 6: ~1900 µs → MODE6
```

#### **Option B: Two Switches (3-pos × 2-pos = 6 modes)**

```
Switch A: 3-position
Switch B: 2-position

Use transmitter mixing to create 6 combinations:

Mix formula (example for OpenTX):
CH5 = 50% × SwA + 50% × SwB

Results:
A↓ B↓ → PWM 1100 → MODE1
A↓ B↑ → PWM 1300 → MODE2
A- B↓ → PWM 1450 → MODE3
A- B↑ → PWM 1550 → MODE4
A↑ B↓ → PWM 1700 → MODE5
A↑ B↑ → PWM 1900 → MODE6
```

#### **Option C: Simple 3-Position Switch**

```
If you only need 3 modes:

Switch outputs:
Position 1: ~1100 µs → MODE1
Position 2: ~1500 µs → MODE3 or MODE4
Position 3: ~1900 µs → MODE6

Set unused MODE slots to duplicates:
MODE1 = 0   (Manual)
MODE2 = 0   (Manual - duplicate)
MODE3 = 10  (Auto)
MODE4 = 10  (Auto - duplicate)
MODE5 = 11  (RTL)
MODE6 = 11  (RTL - duplicate)

Effectively gives 3 modes with 3-pos switch
```

---

### **Step 3: Assign Modes in Mission Planner**

#### **Method A: Visual Mode Setup (Easier)**

```
1. Mission Planner → CONFIG → Mandatory Hardware → Flight Modes

2. You'll see a screen with 6 mode boxes:
   ┌────────────────────────────────────────────┐
   │ Flight Mode 1: [Manual      ▼]             │
   │ Flight Mode 2: [Hold        ▼]             │
   │ Flight Mode 3: [Steering    ▼]             │
   │ Flight Mode 4: [Auto        ▼]             │
   │ Flight Mode 5: [RTL         ▼]             │
   │ Flight Mode 6: [Guided      ▼]             │
   │                                            │
   │ [Save Modes]                               │
   └────────────────────────────────────────────┘

3. Toggle your mode switch on transmitter

4. Active mode box highlights in GREEN

5. Click dropdown for that mode

6. Select desired mode from list

7. Repeat for all 6 positions

8. Click "Save Modes"
```

#### **Method B: Direct Parameter Entry**

```
Mission Planner → CONFIG → Full Parameter List

Set directly:
┌────────────────────────────────────┐
│ MODE1 = 0      # Manual            │
│ MODE2 = 4      # Hold              │
│ MODE3 = 3      # Steering          │
│ MODE4 = 10     # Auto              │
│ MODE5 = 11     # RTL               │
│ MODE6 = 15     # Guided            │
└────────────────────────────────────┘

Write Parameters
```

---

## 📻 Transmitter Setup Examples

### **Example 1: FrSky Taranis/X9D (OpenTX)**

```
1. MIXER screen:
   CH5
   └── Source: SA (3-pos switch)
       Weight: 100%
       Offset: 0

2. INPUTS screen (optional, for 6 modes):
   Create mix for 6 positions using 2 switches

3. Check output (SERVOS screen):
   SA↓ → CH5 = -100% (PWM ~1000)
   SA- → CH5 = 0%    (PWM ~1500)
   SA↑ → CH5 = +100% (PWM ~2000)

4. Test in Mission Planner
```

---

### **Example 2: Radiomaster TX16S (EdgeTX)**

```
1. MIXER screen:
   CH5: SA Weight(100)

2. Adjust switch endpoints if needed:
   INPUTS → SA
   Min: -100%
   Max: +100%

3. For 6 modes, add second switch:
   CH5: SA Weight(50) + SB Weight(50)

4. Verify output values match mode ranges
```

---

### **Example 3: Spektrum DX6/DX8**

```
1. Function List:
   Find GEAR or AUX1 (becomes Ch5)

2. Assign 3-position switch

3. Adjust Travel/Endpoints:
   Switch Down: 100%
   Switch Center: 50%
   Switch Up: 0%

4. Test in Mission Planner
```

---

### **Example 4: FlySky FS-i6 (Budget)**

```
1. Functions menu:
   Aux channels setup
   
2. Assign SwC (3-pos) to Ch5

3. Endpoints:
   Position 1: -100%
   Position 2: 0%
   Position 3: +100%

4. Save and test
```

---

## ✅ Verification & Testing

### **Test 1: PWM Value Check**

```
1. Mission Planner → CONFIG → Radio Calibration

2. Toggle your mode switch through all positions

3. Watch RC5 (or your MODE_CH):
   ┌────────────────────────────────────┐
   │ Radio 5: [████████░░░░] 1165      │
   └────────────────────────────────────┘

4. Verify distinct PWM values:
   Position 1: 1000-1230 ✓
   Position 2: 1231-1360 ✓
   Position 3: 1361-1490 ✓
   Position 4: 1491-1620 ✓
   Position 5: 1621-1749 ✓
   Position 6: 1750-2049 ✓

5. Each position should be in different range
```

---

### **Test 2: Mode Change Verification**

```
1. Mission Planner → DATA → Quick tab

2. Look at "Mode" field:
   ┌────────────────────────────────────┐
   │ Mode: Manual                       │
   └────────────────────────────────────┘

3. Toggle mode switch

4. Mode field should change:
   Switch Pos 1 → "Manual"
   Switch Pos 2 → "Hold"
   Switch Pos 3 → "Steering"
   Switch Pos 4 → "Auto"
   Switch Pos 5 → "RTL"
   Switch Pos 6 → "Guided"

5. All modes respond correctly ✓
```

---

### **Test 3: Flight Modes Screen**

```
1. CONFIG → Mandatory Hardware → Flight Modes

2. Current mode shows in large text

3. Toggle switch - active mode highlights:
   ┌────────────────────────────────────────────┐
   │ Current Flight Mode: Manual                │
   │                                            │
   │ Mode 1: [1165] Manual    ← GREEN highlight │
   │ Mode 2: [1295] Hold                        │
   │ Mode 3: [1425] Steering                    │
   │ Mode 4: [1555] Auto                        │
   │ Mode 5: [1685] RTL                         │
   │ Mode 6: [1815] Guided                      │
   └────────────────────────────────────────────┘

4. Toggle through all - each highlights ✓
```

---

## 🎯 Recommended Mode Assignments

### **For Tank Testing & Operation:**

```
MODE1 = 0  (Manual)        ⭐ ALWAYS Position 1
  • Direct control
  • Safest mode
  • Easy to access
  • Default failback

MODE2 = 4  (Hold)          ⭐ EMERGENCY STOP
  • Immediate stop
  • Quick access
  • Safety critical

MODE3 = 3  (Steering)      # Optional
  • Speed-controlled manual
  • Good for learning
  • Smoother than Manual

MODE4 = 10 (Auto)          ⭐ AUTONOMOUS MISSIONS
  • Waypoint following
  • Main autonomous mode
  • Test thoroughly first

MODE5 = 11 (RTL)           ⭐ RETURN HOME
  • Automatic return
  • Battery low? Switch here
  • Lost orientation? RTL

MODE6 = 15 (Guided)        # Advanced
  • GCS/companion control
  • Precision positioning
  • Research/development
```

---

### **Alternative: Simplified 3-Mode Setup**

**For beginners or limited switch positions:**

```
MODE1 = 0  (Manual)        # Position 1: Manual control
MODE2 = 0  (Manual)        # Not used
MODE3 = 4  (Hold)          # Position 2: Emergency stop
MODE4 = 10 (Auto)          # Position 3: Autonomous mission
MODE5 = 11 (RTL)           # Not used (or Position 4 if 4-pos)
MODE6 = 11 (RTL)           # Not used

3-position switch gives:
Position 1 → Manual  (drive yourself)
Position 2 → Hold    (stop!)
Position 3 → Auto    (autonomous)
```

---

## 🎮 During Operation

### **Typical Mission Scenario:**

```
[Startup]
Switch Position 1 (Manual)
→ Manual mode
→ ARM rover
→ Drive to mission start area

[Start Mission]
Switch Position 4 (Auto)
→ Auto mode
→ Tank starts following waypoints
→ Maintains CRUISE_SPEED
→ You monitor, hands off sticks

[Problem Detected]
Switch Position 2 (Hold)
→ Hold mode
→ IMMEDIATE STOP
→ Assess situation

[Resume or Take Control]
Switch Position 1 (Manual)
→ Manual mode
→ Drive manually
→ Fix issue

[Battery Low]
Switch Position 5 (RTL)
→ RTL mode
→ Automatically drives home
→ Lands/stops at launch point

[Mission Complete]
Switch Position 1 (Manual)
→ Manual mode
→ DISARM
→ Done!
```

---

## 🔧 Advanced Configuration

### **PWM Value Customization:**

**If your switch doesn't output standard PWM values:**

```
Example: Your switch outputs:
Position 1: 1050
Position 2: 1200
Position 3: 1350
Position 4: 1500
Position 5: 1650
Position 6: 1800

These work fine! ArduRover automatically maps to closest MODE slot.

No need to adjust - just assign modes to each slot.
```

---

### **Using More Than 6 Modes:**

**ArduRover only supports 6 mode slots via MODE_CH, but you can:**

**Option 1: Use RCx_OPTION for additional mode switches**

```
RC7_OPTION = 58            # Acro mode on Ch7
RC8_OPTION = 9             # RTL on Ch8

Now:
• Ch5 switch: Primary mode selection (6 modes)
• Ch7 switch: Quick Acro toggle
• Ch8 switch: Quick RTL (panic button)

Last activated takes priority
```

**Option 2: Conditional mixing on transmitter**

```
Create logical switches on transmitter:
• If SwA↑ AND SwB↑ → Output special PWM
• More complex setups possible
• 9+ modes achievable
```

---

## 🧪 Testing Procedure

### **Safe Testing Steps:**

```
1. Bench Test (No Battery):
   □ Connect USB only
   □ Mission Planner connected
   □ Toggle mode switch
   □ Watch mode change in HUD
   □ Verify all positions work
   □ No motors spin (disarmed)

2. Ground Test (Battery, Not Armed):
   □ Connect battery
   □ Don't ARM yet
   □ Toggle mode switch
   □ Mode changes visible
   □ Still safe (not armed)

3. Armed Test (Wheels Off):
   □ Lift tank or remove wheels
   □ ARM rover
   □ Test mode in Manual
   □ Motors should be controllable
   □ Switch to Hold → motors stop
   □ DISARM

4. First Drive Test:
   □ Place on ground in safe area
   □ Start in Manual mode
   □ ARM and drive
   □ Test switching to Hold
   □ Test switching back to Manual
   □ Test other modes as comfortable

5. Autonomous Test:
   □ GPS lock acquired
   □ Test Loiter
   □ Test Auto with simple mission
   □ Test RTL
   □ Always have Manual easily accessible
```

---

## ⚠️ Safety Best Practices

### **Critical Safety Rules:**

```
1. ✅ ALWAYS have Manual mode on Position 1
   • Easiest position to reach
   • Most intuitive
   • Instant manual control

2. ✅ Have Hold or Manual on Position 2
   • Emergency stop
   • Quick access
   • Safety critical

3. ❌ NEVER put Auto on Position 1
   • Dangerous on startup
   • Might start moving unexpectedly
   • Hard to take control

4. ✅ Test mode switching on bench FIRST
   • Before battery connected
   • Before motors can spin
   • Verify all modes work

5. ✅ Know your switch positions by feel
   • Should be able to switch blindly
   • Muscle memory important
   • Practice mode switching
```

---

### **Emergency Procedures:**

```
If tank is doing something unexpected:

Option 1: Switch to Hold mode (Position 2)
→ Immediate stop
→ Assess situation

Option 2: Switch to Manual mode (Position 1)
→ Take direct control
→ Drive to safety

Option 3: DISARM (kill switch)
→ Motors stop instantly
→ May roll/coast
→ Last resort

Option 4: Turn off transmitter
→ Triggers RC failsafe
→ Executes FS_ACTION (usually Hold)
→ Tank stops
```

---

## 🎨 Mode Assignment Strategies

### **Strategy 1: Mission-Focused**

```
MODE1 = 0  (Manual)        # Safe default
MODE2 = 4  (Hold)          # Emergency
MODE3 = 10 (Auto)          # Main mission mode
MODE4 = 11 (RTL)           # Return
MODE5 = 3  (Steering)      # Manual with speed control
MODE6 = 15 (Guided)        # GCS override
```

**Use case:** Autonomous missions, minimal manual driving

---

### **Strategy 2: Mixed Operation**

```
MODE1 = 0  (Manual)        # Manual driving
MODE2 = 3  (Steering)      # Speed-controlled manual
MODE3 = 4  (Hold)          # Stop
MODE4 = 10 (Auto)          # Auto mission
MODE5 = 11 (RTL)           # Return
MODE6 = 5  (Loiter)        # Hold position
```

**Use case:** Lots of manual + some autonomous

---

### **Strategy 3: Testing/Development**

```
MODE1 = 0  (Manual)        # Basic control
MODE2 = 1  (Acro)          # Rate testing
MODE3 = 3  (Steering)      # Speed testing
MODE4 = 5  (Loiter)        # Position hold test
MODE5 = 9  (Circle)        # Circle test
MODE6 = 10 (Auto)          # Mission test
```

**Use case:** Testing different control modes

---

## 📝 Mode Switching Flow Diagram

```
         [RC Transmitter]
                │
                │ Mode Switch toggled
                ▼
         ┌─────────────┐
         │ Switch Pos  │
         │ Changed     │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │ Outputs PWM │
         │ on MODE_CH  │
         └──────┬──────┘
                │
                ▼
    ┌───────────────────────┐
    │ ArduRover Receives    │
    │ PWM on Channel 5      │
    └──────┬────────────────┘
           │
           ▼
    ┌───────────────────────┐
    │ Maps PWM to MODE slot │
    │ 1425µs → MODE3        │
    └──────┬────────────────┘
           │
           ▼
    ┌───────────────────────┐
    │ Reads MODE3 value     │
    │ MODE3 = 10 (Auto)     │
    └──────┬────────────────┘
           │
           ▼
    ┌───────────────────────┐
    │ Switches to Auto mode │
    │ Starts mission        │
    └───────────────────────┘
```

---

## 🔍 Troubleshooting

### **Problem: Mode doesn't change when I toggle switch**

**Check:**
```
□ MODE_CH set correctly? (e.g., MODE_CH = 5)
□ Switch actually assigned to that channel?
□ RC calibrated?
□ PWM values in correct ranges?
□ Mission Planner connected and showing telemetry?

Test:
• CONFIG → Radio Calibration
• Watch RC channel corresponding to MODE_CH
• Value should change when switch toggled
```

---

### **Problem: Wrong mode activates**

**Solution:**
```
1. Check PWM value when switch in that position
2. Determine which MODE slot it maps to
3. Either:
   a) Adjust transmitter to output different PWM, OR
   b) Assign correct mode to that MODE slot

Example:
Switch Pos 2 outputs 1400µs
→ Maps to MODE3 (range 1361-1490)
→ Currently MODE3 = 10 (Auto)
→ Want Hold instead
→ Change MODE3 = 4 (Hold)
```

---

### **Problem: Can't access certain mode**

**Check:**
```
□ Mode assigned to any MODE1-6 slot?
□ Switch can output PWM in that range?
□ Not conflicting with other settings?

Solution:
Rearrange MODE assignments to match your switch
```

---

### **Problem: Mode changes randomly**

**Causes:**
```
1. Noisy RC signal
   → Check RC receiver antenna
   → Check for interference
   → Move away from metal/motors

2. Bad switch connection
   → Clean switch contacts
   → Check for loose wires

3. PWM values on boundary
   → Adjust endpoints to be clearly in range
   → Add deadband in transmitter
```

---

## 💡 Pro Tips

### **Tip 1: Mode Switch Position Labels**

```
Add labels to your transmitter:
Position 1: "MAN" (Manual)
Position 2: "STOP" (Hold)
Position 3: "AUTO" (Auto)
Position 4: "HOME" (RTL)

Use tape/stickers/label maker
Helps in field when you can't look at screen
```

---

### **Tip 2: Audio Callouts**

```
Many transmitters support audio announcements:

Configure to announce:
"Manual mode"
"Auto mode"
"R T L mode"
"Hold mode"

Helps confirm mode change without looking
```

---

### **Tip 3: Mode Memory**

```
Some prefer modes to "stick":
• Switch to Auto → starts mission
• Switch back to Manual → takes control
• Switch to Auto again → resumes mission

Others prefer:
• Switch away from Auto → mission pauses
• Must reset or use different mode

Configure via mission parameters if needed
```

---

### **Tip 4: Quick Mode Access**

```
Besides MODE_CH, you can use RCx_OPTION:

RC7_OPTION = 9             # RTL on Ch7
  → Instant RTL when switch flipped
  → Overrides MODE_CH mode
  → "Panic button"

RC8_OPTION = 41            # Arm/Disarm on Ch8
  → Quick arm/disarm
  → Separate from mode selection
```

---

## 📊 Mode Switching Cheat Sheet

```
═══════════════════════════════════════════════════════════
                  MODE SWITCHING QUICK REF
═══════════════════════════════════════════════════════════

CONFIGURATION:
  MODE_CH = 5              # Which RC channel
  MODE1-6 = Mode numbers   # What mode for each position

PWM RANGES:
  MODE1:  910-1230   (typically ~1100)
  MODE2: 1231-1360   (typically ~1300)
  MODE3: 1361-1490   (typically ~1425)
  MODE4: 1491-1620   (typically ~1555)
  MODE5: 1621-1749   (typically ~1685)
  MODE6: 1750-2049   (typically ~1815)

RECOMMENDED FOR TANK:
  MODE1 = 0   Manual    ← Always accessible
  MODE2 = 4   Hold      ← Emergency stop
  MODE3 = 3   Steering  ← Speed control
  MODE4 = 10  Auto      ← Missions
  MODE5 = 11  RTL       ← Return home
  MODE6 = 15  Guided    ← GCS control

VERIFICATION:
  1. CONFIG → Radio Calibration → Check PWM values
  2. CONFIG → Flight Modes → Test mode switching
  3. DATA → Quick → Watch Mode field change

═══════════════════════════════════════════════════════════
```

---

## ✅ Summary

**Mode switching in ArduRover:**
1. Uses ONE RC channel (typically Ch5)
2. PWM value determines which of 6 mode slots activates
3. You assign modes (0-15) to slots (MODE1-MODE6)
4. Configure in Mission Planner
5. Test before flying/driving
6. Always keep Manual easily accessible

**It's simple once configured - just toggle a switch and your tank changes behavior!** 🎯

