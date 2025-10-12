# ArduRover Complete Parameter Reference
## All Parameters with Descriptions & Rover Mode Relevance

**Legend:**
- ✅ **ROVER SPECIFIC** - Only used in Rover
- 🌐 **COMMON** - Shared across all vehicles (Rover, Copter, Plane)
- 📊 **LIBRARY** - From shared library (applicable if feature enabled)
- 🚫 **NOT FOR ROVER** - Not applicable to Rover

---

## 📋 TABLE OF CONTENTS

1. [Frame & Vehicle Configuration](#frame--vehicle-configuration)
2. [Motor & Servo Outputs](#motor--servo-outputs)
3. [RC Input & Control](#rc-input--control)
4. [Navigation & Waypoints](#navigation--waypoints)
5. [Speed & Throttle Control](#speed--throttle-control)
6. [Steering Control](#steering-control)
7. [Failsafes & Safety](#failsafes--safety)
8. [Sensors](#sensors)
9. [Communication](#communication)
10. [Logging & Diagnostics](#logging--diagnostics)
11. [Special Features](#special-features)

---

## 1️⃣ Frame & Vehicle Configuration

### ✅ **FRAME_CLASS** - ROVER SPECIFIC
```
Description: Vehicle type classification
Values: 0=Undefined, 1=Rover, 2=Boat, 3=BalanceBot
Default: 1 (Rover)
Usage: Determines physics model and control behavior
Tank Setup: Set to 1 (Rover)
```

### ✅ **FRAME_TYPE** - ROVER SPECIFIC
```
Description: Motor/wheel arrangement for omnidirectional vehicles
Values: 0=Default, 1=Omni3, 2=OmniX, 3=OmniPlus, 4=Omni3Mecanum
Default: 0 (Default - regular or skid-steer)
Usage: Only needed for omni-directional robots
Tank Setup: Leave at 0 (Default)
```

---

## 2️⃣ Motor & Servo Outputs

### 🌐 **SERVOn_FUNCTION** - COMMON (but Rover uses specific values)
```
Description: Assigns function to each servo/motor output
Rover-Specific Values:
  26 = GroundSteering (steering servo for Ackermann)
  33-36 = Motor1-4 (for omni vehicles)
  70 = Throttle (single motor/ESC)
  73 = ThrottleLeft (left side for skid-steer) ⭐ USE FOR TANK
  74 = ThrottleRight (right side for skid-steer) ⭐ USE FOR TANK

Tank Setup (4 motors):
  SERVO1_FUNCTION = 73  # Left-front
  SERVO2_FUNCTION = 73  # Left-rear
  SERVO3_FUNCTION = 74  # Right-front
  SERVO4_FUNCTION = 74  # Right-rear
```

### 🌐 **SERVOn_MIN** - COMMON
```
Description: Minimum PWM value sent to servo/ESC
Range: 500-1500
Default: 1000
Usage: Minimum throttle position
Tank Setup: 1000 (for most ESCs)
```

### 🌐 **SERVOn_MAX** - COMMON
```
Description: Maximum PWM value sent to servo/ESC
Range: 1500-2500
Default: 2000
Usage: Maximum throttle position
Tank Setup: 2000 (for most ESCs)
```

### 🌐 **SERVOn_TRIM** - COMMON
```
Description: Neutral/center PWM value
Range: 1000-2000
Default: 1500
Usage: Stop position for motors
Tank Setup: 1500
```

### 🌐 **SERVOn_REVERSED** - COMMON
```
Description: Reverse servo/motor direction
Values: 0=Normal, 1=Reversed
Default: 0
Usage: Fix motor direction without rewiring
Tank Setup: Use if motors spin backward
```

---

## 3️⃣ Motor Configuration Group (MOT_*)

### ✅ **MOT_PWM_TYPE** - ROVER SPECIFIC
```
Description: Motor output signal type
Values:
  0 = Normal PWM (1000-2000µs) ⭐ MOST COMMON
  1 = OneShot
  2 = OneShot125
  3 = Brushed with relay
  4 = Brushed bipolar
  5 = DShot150
  6 = DShot300
  7 = DShot600
  8 = DShot1200
Default: 0 (Normal)
Tank Setup: 0 (unless using DShot ESCs)
```

### ✅ **MOT_PWM_FREQ** - ROVER SPECIFIC
```
Description: PWM frequency for brushed motors
Units: kHz
Range: 1-20
Default: 16
Usage: Only for brushed motors
Tank Setup: Not needed for brushless ESCs
```

### ✅ **MOT_THR_MIN** - ROVER SPECIFIC
```
Description: Minimum throttle percentage
Units: %
Range: 0-20
Default: 0
Usage: Minimum output even at zero throttle (for deadzone)
Tank Setup: 0 (or small value if motors need minimum to move)
```

### ✅ **MOT_THR_MAX** - ROVER SPECIFIC
```
Description: Maximum throttle percentage
Units: %
Range: 30-100
Default: 100
Usage: Limit maximum power to prevent overheating
Tank Setup: Start at 50-70 for testing, increase as needed
```

### ✅ **MOT_SLEWRATE** - ROVER SPECIFIC
```
Description: Throttle slew rate limiter
Units: %/second
Range: 0-1000
Default: 100
Usage: Rate of change limit (0=disabled, 100=full range in 1 sec)
Tank Setup: 100 (or lower for smoother acceleration)
```

### ✅ **MOT_THST_EXPO** - ROVER SPECIFIC
```
Description: Thrust curve exponent
Range: -1.0 to 1.0
Default: 0.0 (linear)
Usage: Adjust throttle feel (-1=aggressive low, +1=aggressive high)
Tank Setup: 0.0 (linear response)
```

### ✅ **MOT_SPD_SCA_BASE** - ROVER SPECIFIC
```
Description: Speed above which steering is scaled
Units: m/s
Range: 0-10
Default: 1.0
Usage: Reduce steering at high speeds (for Ackermann only)
Tank Setup: Not used for skid-steer
```

### ✅ **MOT_STR_THR_MIX** - ROVER SPECIFIC
```
Description: Steering vs throttle prioritization
Range: 0.0-1.0
Default: 0.5
Usage: Higher=prioritize steering, Lower=prioritize throttle
Tank Setup: 0.5 (balanced)
  - Use 0.7-0.9 for tight turns
  - Use 0.3-0.5 for speed priority
```

### ✅ **MOT_VEC_ANGLEMAX** - ROVER SPECIFIC (Boats only)
```
Description: Vectored thrust maximum angle
Units: deg
Range: 0-90
Default: 0 (disabled)
Usage: For boats with vectored thrust
Tank Setup: 0 (not used)
```

---

## 4️⃣ RC Input & Control

### 🌐 **MODE_CH** - COMMON
```
Description: RC channel for mode selection
Range: 1-16
Default: 5 (channel 5)
Usage: Which RC channel controls flight mode
Tank Setup: 5 (or your preference)
```

### ✅ **MODE1** through **MODE6** - ROVER MODES
```
Description: Flight mode for each switch position
Values:
  0 = Manual       ⭐ Direct RC control
  1 = Acro         ⭐ Rate control
  3 = Steering     ⭐ Speed control
  4 = Hold         ⭐ Stop/hold position
  5 = Loiter       - Hold position (GPS)
  6 = Follow       - Follow another vehicle
  7 = Simple       - Simplified control
  8 = Dock         - Auto docking
  9 = Circle       - Circle a point
  10 = Auto        ⭐ Waypoint missions
  11 = RTL         ⭐ Return to launch
  12 = SmartRTL    - Return via path
  15 = Guided      - GCS/companion control

Tank Setup:
  MODE1 = 0  (Manual - for testing)
  MODE2 = 4  (Hold - emergency stop)
  MODE3 = 10 (Auto - missions)
  MODE4 = 11 (RTL - return home)
```

### ✅ **PILOT_STEER_TYPE** - ROVER SPECIFIC
```
Description: How pilot steering input is interpreted
Values:
  0 = Default
  1 = Two Paddles (left/right sticks control tracks directly)
  2 = Direction reversed when backing up
  3 = Direction unchanged when backing up
Default: 0
Tank Setup: 0 (standard mixing) or 1 (for dual-stick control)
```

### ✅ **INITIAL_MODE** - ROVER SPECIFIC
```
Description: Mode to start in on boot
Values: Same as MODE1-6
Default: 0 (Manual)
Usage: Auto-start in specific mode
Tank Setup: 0 (Manual is safest)
```

---

## 5️⃣ Navigation & Waypoints

### ✅ **WP_SPEED** - ROVER SPECIFIC
```
Description: Target speed for waypoint navigation
Units: m/s
Range: 0-100
Default: 2.0
Usage: Speed when navigating to waypoints in Auto mode
Tank Setup: 1.0-3.0 depending on vehicle size
```

### ✅ **WP_RADIUS** - ROVER SPECIFIC
```
Description: Waypoint acceptance radius
Units: m
Range: 0-100
Default: 2.0
Usage: Distance at which waypoint is considered "reached"
Tank Setup: 1.0-5.0 depending on precision needed
```

### ✅ **WP_OVERSHOOT** - ROVER SPECIFIC
```
Description: Maximum overshoot distance
Units: m
Range: 0-10
Default: 2.0
Usage: How far past waypoint before turning back
Tank Setup: 1.0-3.0
```

### 📊 **MIS_OPTIONS** - LIBRARY (Mission)
```
Description: Mission behavior options
Bitmask values
Default: 0
Usage: Special mission behaviors
Tank Setup: Usually 0 (default)
```

### ✅ **MIS_DONE_BEHAVE** - ROVER SPECIFIC
```
Description: Behavior after mission completion
Values:
  0 = Hold in Auto
  1 = Loiter in Auto
  2 = Switch to Acro
  3 = Switch to Manual
Default: 0
Tank Setup: 0 (Hold) or 2 (Acro for RC takeover)
```

---

## 6️⃣ Speed & Throttle Control

### ✅ **CRUISE_SPEED** - ROVER SPECIFIC ⭐ IMPORTANT
```
Description: Target cruise speed in auto modes
Units: m/s
Range: 0-100
Default: 2.0
Usage: Desired speed for Auto, Guided, RTL modes
Tank Setup: 1.0-5.0 m/s
  - Start conservative (1-2 m/s)
  - Increase after testing
```

### ✅ **CRUISE_THROTTLE** - ROVER SPECIFIC ⭐ IMPORTANT
```
Description: Throttle percentage for cruise speed
Units: %
Range: 0-100
Default: 50
Usage: Initial throttle estimate to achieve CRUISE_SPEED
Tank Setup: 40-60%
  - ArduPilot will learn and adjust automatically
  - Use cruise learning feature for optimization
```

### ✅ **SPEED_MAX** - ROVER SPECIFIC
```
Description: Maximum speed limit
Units: m/s
Range: 0-30
Default: 0 (auto-calculate from CRUISE_SPEED/THROTTLE)
Usage: Absolute speed limit
Tank Setup: 0 (let it auto-calculate) or set limit
```

---

## 7️⃣ Steering Control (ATC_* Parameters)

### ✅ **ATC_STR_RAT_P** - ROVER SPECIFIC ⭐ CRITICAL FOR TANK
```
Description: Steering rate controller P gain
Range: 0.0-2.0
Default: 0.2
Usage: Proportional gain for steering control
Tank Setup: 0.1-0.3
  - Too low: Sluggish turning
  - Too high: Oscillation/wobbling
  - Start at 0.2, tune as needed
```

### ✅ **ATC_STR_RAT_I** - ROVER SPECIFIC
```
Description: Steering rate controller I gain
Range: 0.0-2.0
Default: 0.2
Usage: Integral gain (corrects steady-state error)
Tank Setup: 0.1-0.2
```

### ✅ **ATC_STR_RAT_D** - ROVER SPECIFIC
```
Description: Steering rate controller D gain
Range: 0.0-0.4
Default: 0.0
Usage: Derivative gain (dampens oscillation)
Tank Setup: 0.0 (usually not needed for ground vehicles)
```

### ✅ **ATC_STR_RAT_IMAX** - ROVER SPECIFIC
```
Description: Steering rate I gain maximum
Units: %
Range: 0-1000
Default: 100
Usage: Limits integral windup
Tank Setup: 100 (default is fine)
```

### ✅ **ATC_STR_RAT_FILT** - ROVER SPECIFIC
```
Description: Steering rate filter frequency
Units: Hz
Range: 1-100
Default: 10
Usage: Low-pass filter to reduce noise
Tank Setup: 10 (default is fine)
```

### ✅ **ATC_STR_RAT_FF** - ROVER SPECIFIC
```
Description: Steering rate feedforward
Range: 0.0-3.0
Default: 0.2
Usage: Improves steering response
Tank Setup: 0.2 (default is fine)
```

### ✅ **ATC_STR_RAT_MAX** - ROVER SPECIFIC ⭐ IMPORTANT
```
Description: Maximum steering rate
Units: deg/s
Range: 0-360
Default: 180
Usage: Limits turning speed
Tank Setup: 
  - 90 deg/s for gentle turns
  - 180 deg/s for normal
  - 360 deg/s for aggressive/pivot turns
```

### ✅ **ATC_STR_ANG_P** - ROVER SPECIFIC
```
Description: Steering angle controller P gain
Range: 1.0-5.0
Default: 2.0
Usage: How aggressively to turn to target heading
Tank Setup: 2.0-3.0
```

### ✅ **ATC_SPEED_P** - ROVER SPECIFIC
```
Description: Speed controller P gain
Range: 0.01-2.0
Default: 0.2
Usage: How aggressively to reach target speed
Tank Setup: 0.1-0.3
```

### ✅ **ATC_SPEED_I** - ROVER SPECIFIC
```
Description: Speed controller I gain
Range: 0.0-2.0
Default: 0.2
Usage: Corrects speed error over time
Tank Setup: 0.1-0.2
```

### ✅ **ATC_SPEED_D** - ROVER SPECIFIC
```
Description: Speed controller D gain
Range: 0.0-0.4
Default: 0.0
Usage: Dampens speed oscillation
Tank Setup: 0.0 (not usually needed)
```

### ✅ **ATC_ACCEL_MAX** - ROVER SPECIFIC
```
Description: Maximum acceleration
Units: m/s/s
Range: 0-10
Default: 0 (unlimited)
Usage: Limit acceleration for smooth motion
Tank Setup: 0 (unlimited) or 1-3 for smooth acceleration
```

### ✅ **ATC_DECEL_MAX** - ROVER SPECIFIC
```
Description: Maximum deceleration
Units: m/s/s
Range: 0-10
Default: 0 (unlimited)
Usage: Limit braking force
Tank Setup: 0 (unlimited) or 2-5 for smooth braking
```

### ✅ **ATC_BRAKE** - ROVER SPECIFIC
```
Description: Enable active braking
Values: 0=Disabled, 1=Enabled
Default: 0
Usage: Use motors to actively brake
Tank Setup: 0 (not needed for most tanks)
```

---

## 8️⃣ Failsafes & Safety

### ✅ **FS_ACTION** - ROVER SPECIFIC ⭐ IMPORTANT
```
Description: Action to take on failsafe
Values:
  0 = Nothing (risky!)
  1 = RTL (return to launch)
  2 = Hold (stop in place) ⭐ SAFEST FOR TANK
  3 = SmartRTL or RTL
  4 = SmartRTL or Hold
  5 = Terminate (emergency motor stop)
  6 = Loiter or Hold
Default: 2 (Hold)
Tank Setup: 2 (Hold) - safest for indoor/confined spaces
```

### ✅ **FS_TIMEOUT** - ROVER SPECIFIC
```
Description: Time before failsafe triggers
Units: seconds
Range: 1-100
Default: 1.5
Usage: Delay before failsafe action
Tank Setup: 1.5-3.0 seconds
```

### ✅ **FS_THR_ENABLE** - ROVER SPECIFIC
```
Description: RC throttle failsafe enable
Values:
  0 = Disabled
  1 = Enabled
  2 = Enabled, continue mission in Auto
Default: 1 (Enabled)
Usage: Triggers when RC signal lost
Tank Setup: 1 (Enabled)
```

### ✅ **FS_THR_VALUE** - ROVER SPECIFIC
```
Description: PWM value below which throttle failsafe triggers
Range: 910-1100
Default: 910
Usage: Detect RC signal loss
Tank Setup: 910 (default)
```

### ✅ **FS_GCS_ENABLE** - ROVER SPECIFIC
```
Description: GCS (telemetry) failsafe enable
Values:
  0 = Disabled
  1 = Enabled
  2 = Enabled, continue mission in Auto
Default: 0 (Disabled)
Usage: Triggers when GCS connection lost
Tank Setup: 0 initially, 1 for autonomous operations
```

### ✅ **FS_CRASH_CHECK** - ROVER SPECIFIC
```
Description: Crash detection action
Values:
  0 = Disabled
  1 = Hold
  2 = Hold and Disarm
Default: 0
Usage: Detect rollover/crash via IMU
Tank Setup: 1 or 2 (recommended for safety)
```

### ✅ **CRASH_ANGLE** - ROVER SPECIFIC
```
Description: Pitch/roll angle for crash detection
Units: degrees
Range: 0-60
Default: 0 (disabled)
Usage: Angle threshold for crash
Tank Setup: 30-45 degrees
```

### ✅ **FS_EKF_ACTION** - ROVER SPECIFIC
```
Description: Action when navigation fails
Values:
  0 = Disabled
  1 = Hold
  2 = Report Only
Default: 1 (Hold)
Usage: When GPS/nav system fails
Tank Setup: 1 (Hold)
```

### ✅ **FS_EKF_THRESH** - ROVER SPECIFIC
```
Description: EKF variance threshold
Values: 0.6=Strict, 0.8=Default, 1.0=Relaxed
Default: 0.8
Usage: Sensitivity of nav failure detection
Tank Setup: 0.8 (default)
```

### ✅ **FS_OPTIONS** - ROVER SPECIFIC
```
Description: Additional failsafe options
Bitmask: 0=Failsafe enabled in Hold mode
Default: 0
Usage: Special failsafe behaviors
Tank Setup: 0 (default)
```

---

## 9️⃣ Arming & Disarming (ARMING_*)

### 🌐 **ARMING_CHECK** - COMMON
```
Description: Pre-arm safety checks
Bitmask: (various checks)
Default: 1 (all checks)
Usage: Verify systems before arming
Tank Setup: 1 (all checks enabled for safety)
```

### 🌐 **ARMING_RUDDER** - COMMON
```
Description: Arm/disarm via rudder stick
Values:
  0 = Disabled
  1 = ArmDisarm
  2 = Arm only
Default: 2 (Arm only - disarm via other method)
Usage: Stick arming
Tank Setup: 0 (use switch instead for tanks)
```

### 🌐 **ARMING_REQUIRE** - COMMON
```
Description: Require arming before motors spin
Values: 0=Disabled, 1=Enabled
Default: 1
Usage: Safety feature
Tank Setup: 1 (always require arming!)
```

---

## 🔟 Sensor Configuration

### 🌐 **COMPASS_* (Many parameters)** - COMMON
```
Description: Magnetometer/compass configuration
Usage: For GPS compass modules
Rover Relevance: ✅ YES - needed for autonomous navigation
Tank Setup:
  - Auto-detected if using GPS with compass
  - Calibrate compass before autonomous driving
```

### 🌐 **GPS_* (Many parameters)** - COMMON
```
Description: GPS configuration
Usage: Position estimation
Rover Relevance: ✅ YES - essential for Auto/RTL/Loiter modes
Tank Setup:
  - Usually auto-detected
  - GPS_TYPE = 1 (Auto)
```

### 🌐 **INS_* (Many parameters)** - COMMON
```
Description: Inertial Navigation System (IMU)
Usage: Accelerometer and gyroscope
Rover Relevance: ✅ YES - core sensor
Tank Setup: Usually default, calibrate accelerometer
```

### 🌐 **BARO_* (Many parameters)** - COMMON
```
Description: Barometer/pressure sensor
Usage: Altitude estimation
Rover Relevance: ⚠️ LIMITED - altitude not critical for ground
Tank Setup: Default is fine, not critical for flat ground
```

### 📊 **RNGFND_* (Many parameters)** - LIBRARY (RangeFinder)
```
Description: Distance sensors (LIDAR, sonar, etc.)
Usage: Obstacle detection, precision docking
Rover Relevance: ✅ OPTIONAL - useful for obstacle avoidance
Tank Setup: Configure if you have rangefinders
```

### 📊 **WENC_* (Many parameters)** - LIBRARY (Wheel Encoder)
```
Description: Wheel encoder odometry
Usage: Dead reckoning, position estimation
Rover Relevance: ✅ VERY USEFUL for rovers
Tank Setup: Configure if you have wheel encoders
  - Improves GPS-denied navigation
  - Better position estimation
```

---

## 1️⃣1️⃣ Communication (SERIAL*)

### 🌐 **SERIALn_PROTOCOL** - COMMON
```
Description: Protocol for each serial port
Values: (see 50+ protocols earlier)
  1/2 = MAVLink
  5 = GPS
  23 = RCIN
  etc.
Default: Varies by port
Tank Setup:
  SERIAL0_PROTOCOL = 2  (MAVLink2 - USB)
  SERIAL1_PROTOCOL = 2  (MAVLink2 - Telemetry)
  SERIAL6_PROTOCOL = 5  (GPS)
```

### 🌐 **SERIALn_BAUD** - COMMON
```
Description: Baud rate for serial port
Values: See baud rate table
Default: Varies by protocol
Tank Setup:
  SERIAL0_BAUD = 115  (115200)
  SERIAL1_BAUD = 57   (57600)
  SERIAL6_BAUD = 230  (230400 for GPS)
```

---

## 1️⃣2️⃣ Logging (LOG_*)

### ✅ **LOG_BITMASK** - ROVER SPECIFIC
```
Description: What data to log
Bitmask:
  0 = Fast Attitude
  1 = Medium Attitude
  2 = GPS
  3 = System Performance
  4 = Throttle ⭐
  5 = Navigation Tuning ⭐
  7 = IMU
  8 = Mission Commands ⭐
  9 = Battery
  10 = Rangefinder
  11 = Compass
  12 = Camera
  13 = Steering ⭐
  14 = RC Input-Output
  19 = Raw IMU
  20 = Video Stabilization
  21 = Optical Flow
Default: 65535 (all enabled)
Tank Setup: 65535 (log everything for analysis)
```

---

## 1️⃣3️⃣ Special Rover Features

### ✅ **TURN_RADIUS** - ROVER SPECIFIC
```
Description: Vehicle turning radius
Units: m
Range: 0-10
Default: 0.9
Usage: For path planning (Ackermann steering mainly)
Rover Relevance: ✅ For Ackermann, ⚠️ Not critical for skid-steer
Tank Setup: Can leave at default (skid-steer ignores this)
```

### ✅ **ACRO_TURN_RATE** - ROVER SPECIFIC
```
Description: Maximum turn rate in Acro mode
Units: deg/s
Range: 0-360
Default: 180
Usage: Rate control mode turning limit
Tank Setup: 180-360 for responsive control
```

### ✅ **BAL_PITCH_MAX** - ROVER SPECIFIC (BalanceBot only)
```
Description: Balance bot maximum pitch
Units: deg
Range: 0-15
Default: 10
Rover Relevance: 🚫 Only for FRAME_CLASS=3 (BalanceBot)
Tank Setup: Not used (you're not a balance bot)
```

### ✅ **BAL_PITCH_TRIM** - ROVER SPECIFIC (BalanceBot only)
```
Description: Balance bot pitch trim
Units: deg
Range: -2 to 2
Default: 0
Rover Relevance: 🚫 Only for BalanceBot
Tank Setup: Not used
```

### ✅ **LOIT_TYPE** - ROVER SPECIFIC
```
Description: Loiter behavior
Values:
  0 = Forward or reverse to target
  1 = Always face forward
  2 = Always face backward
Default: 0
Usage: How to approach loiter point
Tank Setup: 0 or 1 depending on preference
```

### ✅ **LOIT_RADIUS** - ROVER SPECIFIC
```
Description: Loiter acceptance radius
Units: m
Range: 0-20
Default: 2
Usage: How close to get to loiter point
Tank Setup: 1-3 meters
```

### ✅ **LOIT_SPEED_GAIN** - ROVER SPECIFIC
```
Description: Loiter position correction aggressiveness
Range: 0-5
Default: 0.5
Usage: How hard to correct drift
Tank Setup: 0.5 (default)
```

### ✅ **RTL_SPEED** - ROVER SPECIFIC
```
Description: Return-to-launch speed
Units: m/s
Range: 0-100
Default: 0 (use WP_SPEED or CRUISE_SPEED)
Usage: Speed when returning home
Tank Setup: 0 (auto) or set specific RTL speed
```

### ✅ **SIMPLE_TYPE** - ROVER SPECIFIC
```
Description: Simple mode behavior
Values:
  0 = Initial heading based
  1 = Cardinal directions
Default: 0
Usage: Simplified control mode
Tank Setup: Not commonly used
```

---

## 1️⃣4️⃣ Advanced Features

### 📊 **AVOID_* (Many)** - LIBRARY (Object Avoidance)
```
Description: Object avoidance system
Usage: Avoid obstacles using proximity sensors
Rover Relevance: ✅ OPTIONAL - very useful for autonomous
Tank Setup: Configure if you have LIDAR/proximity sensors
```

### 📊 **PRX_* (Many)** - LIBRARY (Proximity)
```
Description: Proximity sensor configuration
Usage: 360° obstacle detection
Rover Relevance: ✅ OPTIONAL - for advanced navigation
Tank Setup: Configure if you have 360° LIDAR
```

### 📊 **OA_* (Many)** - LIBRARY (Path Planner)
```
Description: Object avoidance path planning
Usage: BendyRuler, Dijkstra algorithms
Rover Relevance: ✅ OPTIONAL - autonomous obstacle avoidance
Tank Setup: Enable if using proximity sensors
```

### 📊 **FOLL_* (Many)** - LIBRARY (Follow Mode)
```
Description: Follow another vehicle
Usage: Follow mode configuration
Rover Relevance: ✅ OPTIONAL - follow lead vehicle
Tank Setup: Configure if using follow mode
```

### 📊 **RALLY_* (Many)** - LIBRARY (Rally Points)
```
Description: Alternative return locations
Usage: Safe landing points
Rover Relevance: ✅ OPTIONAL - alternative RTL destinations
Tank Setup: Not critical, but useful for complex missions
```

### 📊 **FENCE_* (Many)** - LIBRARY (Geofencing)
```
Description: Virtual boundaries
Usage: Keep vehicle in designated area
Rover Relevance: ✅ RECOMMENDED for safety
Tank Setup: Highly recommended for testing area
```

### 📊 **SAIL_* (Many)** - LIBRARY (Sailboat)
```
Description: Sailboat-specific parameters
Usage: Sail control, tacking, wind navigation
Rover Relevance: 🚫 Only for FRAME_CLASS=2 (Boat) with sails
Tank Setup: Not used
```

---

## 1️⃣5️⃣ Battery & Power (BATT*)

### 🌐 **BATT_MONITOR** - COMMON
```
Description: Battery voltage/current monitoring
Values:
  0 = Disabled
  3 = Analog Voltage Only
  4 = Analog Voltage and Current ⭐ RECOMMENDED
Default: 0
Usage: Monitor battery state
Tank Setup: 4 (if you have current sensor)
```

### 🌐 **BATT_VOLT_PIN** - COMMON
```
Description: ADC pin for voltage sensing
Default: Board-specific (usually auto)
Usage: Voltage measurement
Tank Setup: Board default (10 for SpeedyBee)
```

### 🌐 **BATT_CURR_PIN** - COMMON
```
Description: ADC pin for current sensing
Default: Board-specific
Usage: Current measurement
Tank Setup: Board default (11 for SpeedyBee)
```

### 🌐 **BATT_VOLT_MULT** - COMMON
```
Description: Voltage multiplier for calibration
Default: Board-specific (11.2 for SpeedyBee)
Usage: Calibrate voltage reading
Tank Setup: Calibrate with known voltage
```

### 🌐 **BATT_AMP_PERVLT** - COMMON
```
Description: Amps per volt from current sensor
Default: 52.7 (SpeedyBee default)
Usage: Calibrate current reading
Tank Setup: Depends on your current sensor
```

### 🌐 **BATT_CAPACITY** - COMMON
```
Description: Battery capacity
Units: mAh
Range: 0-100000
Default: 0
Usage: Track battery usage percentage
Tank Setup: Set to your battery capacity (e.g., 5000 for 5000mAh)
```

### 🌐 **BATT_LOW_VOLT** - COMMON
```
Description: Low battery voltage threshold
Units: V
Default: 0 (disabled)
Usage: Trigger failsafe at low voltage
Tank Setup: Set to battery cutoff (e.g., 10.5V for 3S LiPo)
```

---

## 1️⃣6️⃣ RC Channel Mapping (RCMAP*)

### 🌐 **RCMAP_ROLL** - COMMON
```
Description: RC input channel for roll
Default: 1
Rover Relevance: ✅ Used as STEERING in Rover
Tank Setup: 1 (Channel 1 = Steering)
```

### 🌐 **RCMAP_PITCH** - COMMON
```
Description: RC input channel for pitch
Default: 2
Rover Relevance: ⚠️ Not used by Rover
Tank Setup: 2 (ignored)
```

### 🌐 **RCMAP_THROTTLE** - COMMON
```
Description: RC input channel for throttle
Default: 3
Rover Relevance: ✅ YES - throttle control
Tank Setup: 3 (Channel 3 = Throttle)
```

### 🌐 **RCMAP_YAW** - COMMON
```
Description: RC input channel for yaw
Default: 4
Rover Relevance: ⚠️ Not used by most Rover modes
Tank Setup: 4 (usually not used)
```

---

## 1️⃣7️⃣ Auto/Mission Specific

### ✅ **AUTO_TRIGGER_PIN** - ROVER SPECIFIC
```
Description: Pin to enable throttle in Auto mode
Values: -1=Disabled, 0-55=Pin numbers
Default: -1
Usage: "Press button to start" functionality
Tank Setup: -1 (disabled) unless using button trigger
```

### ✅ **AUTO_KICKSTART** - ROVER SPECIFIC
```
Description: Acceleration to trigger auto mode start
Units: m/s/s
Range: 0-20
Default: 0 (start immediately)
Usage: Wait for push before starting
Tank Setup: 0 (immediate start)
```

---

## 1️⃣8️⃣ Parameter Groups (Libraries)

### 📊 **CIRC_* (Circle Mode)** - Rover has dedicated circle mode
```
Circle mode parameters
Rover Relevance: ✅ YES - for circle mode
```

### 📊 **DOCK_* (Dock Mode)** - Rover docking
```
Autonomous docking parameters
Rover Relevance: ✅ OPTIONAL - precision docking
```

### 📊 **WP_* (Waypoint Navigation)** - AR_WPNav library
```
Waypoint navigation parameters
Rover Relevance: ✅ YES - critical for Auto mode
```

### 📊 **PSC_* (Position Control)** - AR_PosControl library
```
Position controller parameters
Rover Relevance: ✅ YES - for GPS navigation
```

### 📊 **GUID_OPTIONS** - Guided mode options
```
Guided mode behavior
Rover Relevance: ✅ YES - for GCS/companion control
```

---

## 🎯 ESSENTIAL PARAMETERS FOR YOUR 4-MOTOR TANK

### **Minimum Required Setup:**

```ini
# ========== FRAME & VEHICLE ==========
FRAME_CLASS = 1                    # ✅ ROVER - Ground vehicle

# ========== MOTOR OUTPUTS (4-motor tank) ==========
SERVO1_FUNCTION = 73               # ✅ ThrottleLeft (Left-Front)
SERVO2_FUNCTION = 73               # ✅ ThrottleLeft (Left-Rear)
SERVO3_FUNCTION = 74               # ✅ ThrottleRight (Right-Front)
SERVO4_FUNCTION = 74               # ✅ ThrottleRight (Right-Rear)

SERVO1_MIN = 1000                  # 🌐 PWM min
SERVO1_MAX = 2000                  # 🌐 PWM max
SERVO1_TRIM = 1500                 # 🌐 PWM neutral
SERVO2_MIN = 1000
SERVO2_MAX = 2000
SERVO2_TRIM = 1500
SERVO3_MIN = 1000
SERVO3_MAX = 2000
SERVO3_TRIM = 1500
SERVO4_MIN = 1000
SERVO4_MAX = 2000
SERVO4_TRIM = 1500

# ========== MOTOR CONFIGURATION ==========
MOT_PWM_TYPE = 0                   # ✅ Normal PWM
MOT_THR_MIN = 0                    # ✅ Min throttle %
MOT_THR_MAX = 100                  # ✅ Max throttle %
MOT_SLEWRATE = 100                 # ✅ Throttle slew rate
MOT_STR_THR_MIX = 0.5              # ✅ Steering/throttle balance

# ========== STEERING CONTROL ==========
ATC_STR_RAT_P = 0.2                # ✅ Steering P gain ⭐ TUNE THIS
ATC_STR_RAT_I = 0.2                # ✅ Steering I gain
ATC_STR_RAT_D = 0.0                # ✅ Steering D gain
ATC_STR_RAT_MAX = 180              # ✅ Max turn rate (deg/s)
ATC_STR_ANG_P = 2.0                # ✅ Heading P gain

# ========== SPEED CONTROL ==========
CRUISE_SPEED = 2.0                 # ✅ Cruise speed (m/s) ⭐
CRUISE_THROTTLE = 50               # ✅ Cruise throttle (%) ⭐
ATC_SPEED_P = 0.2                  # ✅ Speed P gain
ATC_SPEED_I = 0.2                  # ✅ Speed I gain

# ========== RC & MODES ==========
MODE_CH = 5                        # 🌐 Mode switch channel
MODE1 = 0                          # ✅ Manual
MODE2 = 4                          # ✅ Hold
MODE3 = 10                         # ✅ Auto
MODE4 = 11                         # ✅ RTL

RCMAP_ROLL = 1                     # 🌐 Ch1 = Steering
RCMAP_THROTTLE = 3                 # 🌐 Ch3 = Throttle

# ========== FAILSAFES ==========
FS_ACTION = 2                      # ✅ Hold on failsafe ⭐
FS_TIMEOUT = 1.5                   # ✅ 1.5 sec timeout
FS_THR_ENABLE = 1                  # ✅ RC failsafe enabled
FS_THR_VALUE = 910                 # ✅ Failsafe trigger PWM
FS_GCS_ENABLE = 0                  # ✅ GCS failsafe (off initially)
FS_CRASH_CHECK = 2                 # ✅ Hold and disarm on crash
CRASH_ANGLE = 30                   # ✅ 30 deg crash threshold

# ========== ARMING ==========
ARMING_CHECK = 1                   # 🌐 All pre-arm checks
ARMING_REQUIRE = 1                 # 🌐 Require arming
ARMING_RUDDER = 0                  # 🌐 Disable stick arming

# ========== NAVIGATION ==========
WP_SPEED = 2.0                     # ✅ Waypoint speed
WP_RADIUS = 2.0                    # ✅ Waypoint radius

# ========== BATTERY (if equipped) ==========
BATT_MONITOR = 4                   # 🌐 Voltage + Current
BATT_CAPACITY = 5000               # 🌐 5000 mAh (example)
BATT_VOLT_MULT = 11.2              # 🌐 SpeedyBee default
```

---

## 📝 Parameter File Export

You can also export all current parameters from Mission Planner:

```
1. Connect to vehicle
2. CONFIG → Full Parameter List
3. Click "Save to File" button (right side)
4. Save as: my_tank_params.param
```

This creates a complete backup you can reload anytime!

---

**Document saved to:** `/home/dmitry/workspace/src/ardupilot/ROVER_PARAMETERS_REFERENCE.md`

This gives you a complete parameter reference with Rover-specific annotations! For the full list of ~800+ parameters across all libraries, check the official documentation, but these are the essential ones for your tank! 🚜

