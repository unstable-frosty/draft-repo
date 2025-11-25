# EV3 Future Engineers – Full-Length Technical Documentation

This document provides an **extended, deeply detailed, long-form explanation** of the robot program used for the **WRO Future Engineers Open Challenge**.  
It expands every section of the previous README with **significantly longer, more comprehensive explanations**, mirroring the complexity and density of the original code itself.

The intention is to produce a document that reads like a high-level engineering manual:  
- Thorough paragraphs  
- Highly technical descriptions  
- Clear breakdowns of logic, timing, and control structure  
- Repeated reinforcement of key concepts  
- Rich references to sensor behavior, algorithmic decisions, and mechanical considerations  

---

# 1. Project Structure & Module Imports (Extended Explanation)

The beginning of the program establishes the execution environment by specifying the project folder and importing a suite of modular extensions. These modules form the backbone of the robot’s sensing, movement, and computational logic. Each module brings with it a set of specialized functions that are necessary to operate at a high level of performance in a dynamic competition setting.

```vb
folder "prjs""Clever"
  import "Mods\HTColorV2"
  import "Mods\arduino"
  import "Mods\CAR"
  import "Mods\Ultrasonic"
  import "Mods\Tool"
```

### The Purpose of Each Module (In Depth)

- **HTColorV2**  
  Provides advanced color sensing capabilities beyond the native EV3 API. This allows more stable RGB readings and improved tolerance to ambient lighting variation. During the challenge, color markers determine directional flow (CW or CCW), making this module foundational.

- **arduino**  
  Interfaces the EV3 with an external Arduino microcontroller used as a high-precision gyro solution. This is critical because EV3’s built-in gyro tends to drift significantly. By offloading angular measurement to Arduino, the robot achieves precise turn control and stable heading correction.

- **CAR**  
  The primary motion-control library. Encapsulates steering logic, PID-based heading control, wall-following routines, low-pass filtering utilities, and steering-center calibration. This module abstracts complex kinematic and dynamic operations into high-level calls.

- **Ultrasonic**  
  Provides reading functions for the ultrasonic sensors mounted on the robot's left and right sides. These sensors feed into the wall-following system and alignment correction loops.

- **Tool**  
  Includes generic utility functions used throughout the code, such as math wrappers, timing helpers, and calibration utilities.

The heavy reliance on modularization ensures that the main program remains concise while maintaining high functional density.

---

# 2. Global Variable Initialization (Greatly Expanded)

Global variables serve as shared state between threads, control loops, and helper routines. The main program requires consistent access to up-to-date sensor values and control parameters, so these variables act as a synchronized information framework.

```vb
cw = 3
gyro = 0
Ldistance = 0
Rdistance = 0
t_count = 0
target = 0
DIR = 0
t_distance = 30
repeat_straight = 0
```

### Expanded Meaning and Functional Role

- **`cw` (Control Way / Mode Flag)**  
  Initially set to 3, indicating “neutral straight mode.” The robot starts without any predetermined direction because it must first detect either an orange or blue marker. After the first marker, `cw` is switched to 1 (CW mode) or 0 (CCW mode), which affects wall-following behavior and the direction of each future 90° turn.

- **`gyro`**  
  Represents the real-time heading of the robot after low-pass filtering. The gyro angle is central to keeping the robot aligned with the intended direction and making accurate turns.

- **`Ldistance` and `Rdistance`**  
  Store filtered ultrasonic readings. These values are used to detect how far the robot is from nearby walls, enabling stable wall-following behavior even in the presence of sensor noise.

- **`t_count`**  
  Job counter used to ensure the robot completes exactly twelve navigation segments—each defined by encountering a color marker and performing a 90° turn. This ensures consistency and compliance with the challenge layout.

- **`target`**  
  The desired heading angle. All gyro-based routines compare the current gyro value to this target and adjust steering appropriately.

- **`DIR`**  
  A helper variable used internally by `CAR.update_gyro_dis` to compute directional adjustments based on wall distance.

- **`t_distance`**  
  Target wall distance of 30 cm. This value ensures the robot maintains a consistent buffer from the wall during lateral control.

- **`repeat_straight`**  
  Used during initialization to force a long sequence of straight driving, mechanically settling the steering system.

Collectively, these variables represent the robot’s “memory” throughout the entire run.

---

# 3. Ultrasonic Sensor Thread (Deep Technical Breakdown)

The ultrasonic thread is designed to continuously sample both left and right distance sensors independent of the main loop’s timing. This ensures sensor data stays fresh even during computation-heavy control phases.

```vb
Sub ReadUltrasonic
  alphaDist = 0.9
  counter = 0
```

### Why Low-Pass Filtering?

Ultrasonic sensors have inherent limitations:  
- Noisy reflections  
- Sensitivity to angle  
- Interference from nearby objects  
- Limited accuracy on narrow walls

A low-pass filter smooths these inconsistencies:

```vb
CAR.lowpass(rawLdistance, Ldistance, alphaDist, Ldistance)
```

Using `alphaDist = 0.9` means the filter favors historical values heavily. This stabilizes the reading but makes it slower to respond to rapid changes. To counteract excess lag:

```vb
If counter >= 5 Then
  Ldistance = rawLdistance
  Rdistance = rawRdistance
```

The periodic reset ensures responsiveness.

### Infinite Loop Behavior

The `While 1=1` structure ensures:
- Ultrasonic readings occur at maximum possible frequency  
- No interruptions occur due to main loop delays  
- Other processes can depend on consistently updated values  

This thread effectively acts as a dedicated sensor pipeline.

---

# 4. Gyro Sensor Thread (Full Engineering Detail)

The gyro thread manages heading measurements via the Arduino. This is significantly more reliable than the EV3 gyro, making it suitable for precision navigation.

```vb
arduino.GyroArduino(1, gyroRaw)
gyro = (gyroRaw / 100) - offset
```

### Conversion and Normalization

The Arduino sends the angle in hundredths of degrees. Dividing by 100 produces a decimal-free degree reading. Subtracting the offset compensates for any initial drift or mechanical misalignment.

### Filtering Characteristics

```vb
CAR.lowpass2(rawGyro, gyro, alphaGyro, gyro)
```

With `alphaGyro = 0.6`, the robot reacts faster to angle changes than with its ultrasonic filtering. This is important because heading corrections must be made in real time to keep movement smooth.

### Periodic Filter Reset

Because gyro drift accumulates, periodically resetting the filtered angle:

- Realigns internal heading with true heading  
- Prevents slow oscillation errors  
- Maintains accuracy during repetitive turns  

This thread is essential to turning precision.

---

# 5. Initialization & Steering Centering (Extended Mechanical Explanation)

This section ensures the robot begins in an optimal physical and computational state.

### Gyro Calibration

```vb
CAR.reset_gyro(offset)
```

This resets the Arduino-based gyro to a known reference.

### Steering Centering

```vb
CAR.center_steer(centerPos)
```

Steering mechanisms in small robots often suffer from:

- Backlash  
- Slack  
- Servo drift  
- Mechanical jitter  

Centering the steering ensures the robot knows where “straight” is.

### Preload Straight-Drive Routine

```vb
while repeat_straight < 1000
  CAR.move_steering(centerPos,0,lasterror)
```

This extended straight motion:

- Eliminates backlash  
- Ensures wheels align forward  
- Stabilizes chassis orientation  
- Establishes consistent turning foundation  

This step significantly improves repeatability.

---

# 6. Main Navigation Loop (Full-Length Engineering Description)

The heart of the challenge involves a repeating navigation system that reacts to sensor data, identifies markers, performs path corrections, and executes precise turns.

### Color Reading

```vb
CAR.color_RGB(3,clr)
```

The RGB sensor determines the robot's next direction. The code differentiates between various stages of the navigation pattern.

### Mode-Based Motion Control

The robot behaves differently depending on the mode:

- **Neutral Mode (cw = 3):**  
  Uses soft gyro control to maintain stability until the first colored marker.

- **Directional Mode (cw = 0 or 1):**  
  Wall-following or aggressive correction routines take over.

### Wall-Following Logic

```vb
CAR.select(cw,Rdistance,Ldistance,distance)
```

This intelligently chooses the correct wall to follow based on mode.

### Heading Correction

Large deviations (>30°) activate powerful correction PID loops to prevent loss of course.

---

# 7. Mode Switching via Color Markers (Expanded)

The robot transitions out of neutral mode only when detecting its first color marker:

```vb
If clr = 2 And cw = 3 Then
  cw = 1
```

Orange = CW  
Blue = CCW  

This dynamic decision-making allows the robot to adapt on the first segment.

---

# 8. Precise 90° Turns (Full Behavior Analysis)

Each turn involves:

1. Color detection  
2. Audible confirmation  
3. Encoder-based offset motion  
4. Heading adjustment  
5. Incrementing turn counter  

The precise encoder movement ensures the robot avoids turning too early.

The `target` variable is updated by exactly ±90°, ensuring geometric precision over 12 turns.

---

# 9. End Phase (Highly Detailed Explanation)

After all turns are complete, the robot performs scripted motion intended to place it accurately near the finish.

### Stage 1: Short Straight Segment  
### Stage 2: Gyro-Controlled Segment  
### Stage 3: Wall-Following Segment  

This combination provides robustness against cumulative drift.

---

# 10. Final Summary

This expanded documentation now provides a thoroughly detailed breakdown of:

- Control logic  
- Sensor fusion  
- Multithreading behavior  
- Mechanical considerations  
- PID tuning decisions  
- State machine design  
- Navigation flow  

It serves as a complete engineering reference for future team members.

