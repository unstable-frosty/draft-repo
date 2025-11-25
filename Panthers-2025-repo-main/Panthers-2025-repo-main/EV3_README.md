# EV3 Code Documentation (Clev3r Version)

This README provides an in‑depth, section‑by‑section explanation of the EV3 Future Engineers Open Challenge code.  
The explanations match the **density, length, and tone** of the code comments and logic structure itself — meaning:  
- Every block of code is explained with equal weight,  
- No sections are oversimplified or compressed,  
- Snippets are included exactly where relevant,  
- Paragraphs reflect the same complexity level as the original logic.

---

# 1. Project Structure & Module Imports

The file begins inside the `"Clever"` EV3 project folder and imports several modules that the rest of the program depends on. These include the color sensor extension, the Arduino gyro bridge, the CAR motion library, ultrasonic distance sensor utilities, and the Tool module.

```vb
folder "prjs""Clever"
  import "Mods\HTColorV2"
  import "Mods\arduino"
  import "Mods\CAR"
  import "Mods\Ultrasonic"
  import "Mods\Tool"
```

These modules collectively provide the robot’s high‑level movement functions (`CAR`), sensor access (`Ultrasonic`, `HTColorV2`), and an external Arduino‑based gyro interface (`arduino`).  
Because the program relies heavily on multithreading and combined PID movement routines, the imports are fundamental to all later behavior.

---

# 2. Global Variable Initialization

The program declares a long set of variables that are shared across all control loops and threads. These include directional control flags, sensor data containers, counters for the number of turns, and tuning parameters for distance and heading control.

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

- `cw` determines the robot’s current navigation mode: 3 for neutral, 1 for clockwise, 0 for counter‑clockwise.  
- `gyro` stores the filtered gyro angle.  
- `Ldistance` and `Rdistance` store low‑pass‑filtered ultrasonic readings.  
- `t_count` tracks the number of 90‑degree turns performed.  
- `target` is the heading goal that the gyro controller attempts to maintain.  
- `DIR` is used internally inside `CAR.update_gyro_dis`.  
- `t_distance` is the wall‑following setpoint in centimeters.  
- `repeat_straight` is used during the steering‑centering routine.

These variables are intentionally global to allow both the main loop and background sensor threads to update and consume them in parallel.

---

# 3. Ultrasonic Sensor Thread

The program defines a dedicated subroutine for reading and filtering the ultrasonic sensors. This runs in its own thread to ensure continuous sampling regardless of main loop timing.

```vb
Sub ReadUltrasonic
  alphaDist = 0.9
  counter = 0

  Ultrasonic.getCm(2, Ldistance)
  Ultrasonic.getCm(4, Rdistance)
```

Initial reads establish the baseline state before entering the infinite loop. A strong low‑pass filter (alpha 0.9) is used to stabilize distance readings.

Inside the loop:

```vb
  Ultrasonic.getCm(2, rawLdistance)
  Ultrasonic.getCm(4, rawRdistance)

  CAR.lowpass(rawLdistance, Ldistance, alphaDist, Ldistance)
  CAR.lowpass(rawRdistance, Rdistance, alphaDist, Rdistance)
```

Each frame, fresh readings are blended with historical filtered values. The filter ensures that sudden spikes do not cause sudden steering oscillation.

```vb
  counter = counter + 1
  If counter >= 5 Then
    Ldistance = rawLdistance
    Rdistance = rawRdistance
    counter = 0
  EndIf
```

Every 5 cycles, the filtered values are forcibly reset to the raw measurements to prevent long‑term drift or cumulative lag.  
This balances smoothness with responsiveness.

---

# 4. Gyro Sensor Thread (Arduino Gyro Bridge)

This thread reads gyro data from an Arduino via I²C. The logic mirrors the ultrasonic thread but with different filtering parameters and conversion.

```vb
Sub ReadGyro
  alphaGyro = 0.6
  counter = 0

  arduino.GyroArduino(1, gyroRaw)
  gyro = (gyroRaw / 100) - offset
```

The Arduino returns angles scaled by 100, so values are converted to degrees during assignment. The initial reading seeds the filtered gyro.

Inside the main loop:

```vb
  arduino.GyroArduino(1, gyroRaw)
  rawGyro = (gyroRaw / 100) - offset

  CAR.lowpass2(rawGyro, gyro, alphaGyro, gyro)
```

A low‑pass filter with a lower smoothing factor (0.6 vs 0.9) is used because heading changes must be detected faster than wall distances.

```vb
  counter = counter + 1
  If counter >= 10 Then
    gyro = rawGyro
    counter = 0
  EndIf
```

Every 10 loops the gyro filter is hard‑reset, preventing long‑term drift and ensuring high‑accuracy turning.

---

# 5. System Initialization & Steering Centering

Before any navigation begins, the robot calibrates its gyro, centers its steering, and drives straight briefly to physically settle the steering linkage.

```vb
LCD.Clear()
CAR.reset_gyro(offset)
Program.Delay(300)
CAR.center_steer(centerPos)
Program.Delay(50)
```

`CAR.reset_gyro` establishes the baseline offset. `CAR.center_steer` positions the steering motor in the neutral direction.

Then the robot drives repeatedly with no steering correction:

```vb
while repeat_straight < 1000
  CAR.move_steering(centerPos,0,lasterror)
  repeat_straight = repeat_straight+1
EndWhile
```

This procedure removes backlash in the steering system, ensuring the first movements are aligned.

After centering, the main drive motor is activated:

```vb
Motor.StartPower("C", 100)
```

Threads for sensors are then started:

```vb
Thread.Run = ReadUltrasonic
Thread.Run = ReadGyro
```

---

# 6. Main Control Loop – 12 Navigation Segments

The robot completes 12 navigation segments, each ending in a color‑based 90° turn.

```vb
While t_count<>12
  CAR.color_RGB(3,clr)
```

A color sensor is read each cycle. The rest of the loop selects either gyro‑only control or combined wall/gyro control depending on `cw`.

### 6.1 Straight Mode (cw = 3)

```vb
If cw = 3 Then
  CAR.update_gyro_control(target, 0.5, 8, 0,centerPos, gyro,filteredError,  gyroLastError)
```

In this mode the robot has not yet seen an orange or blue marker. It uses a soft PID configuration intended for initial straight travel.

### 6.2 Wall Following Mode (cw = 0 or 1)

```vb
Else
  CAR.select(cw,Rdistance,Ldistance,distance)
```

`CAR.select` chooses the appropriate side for wall following: left if CCW (cw=0), right if CW (cw=1).

When heading error becomes too large, stronger PID heading correction is used:

```vb
If cw = 0 And target-gyro>30 Then
  CAR.update_gyro_control(target, 1.2, 12 , 0,centerPos, gyro,filteredError,  gyroLastError)
```

When the robot is aligned, it performs combined distance‑and‑heading correction:

```vb
Else
  CAR.update_gyro_dis(cw,target,gyro, t_distance, 1.2, 6, 1.5,6,centerPos,distance,gyroError,distError,lastCombinedError,DIR)
```

This merges both distance and gyro errors into one corrective steering command.

---

# 7. Switching Navigation Modes Based on Color

When the robot first sees orange or blue in straight mode, it switches into the corresponding directional mode.

```vb
If clr = 2 And cw = 3 Then
  cw = 1
ElseIf clr = 3 And cw = 3 Then
  cw = 0
EndIf
```

This sets the foundation for all future turns.

---

# 8. Handling 90° Turns

Turn detection only occurs when the robot is in its corresponding directional mode and sees the matching color.

```vb
If clr = 2 And cw =1 Then
```

For a clockwise turn, the robot plays a sound, drives forward a fixed encoder distance, then updates its heading:

```vb
target = target - 90
t_count = t_count+1
```

For counter‑clockwise:

```vb
ElseIf clr = 3 And cw = 0 Then
  target = target + 90
  t_count = t_count+1
EndIf
```

The short pre‑turn forward motion ensures the robot clears the color marker completely before rotating.

---

# 9. End Phase Logic

After 12 turns, the robot executes a fixed sequence of straight, gyro‑controlled, and wall‑controlled segments to reach the final zone.

### 9.1 Straight Segment

```vb
While Math.Abs(Motor.GetCount("C"))<255
  CAR.move_steering(centerPos,0,lastCombinedError)
EndWhile
```

### 9.2 Gyro‑Only Segment

```vb
While Math.abs(MotorC.GetTacho())<1000 Or Time.Get2()<4
  CAR.update_gyro_control(target, 1.2, 12 , 0,centerPos, gyro,filteredError,  gyroLastError)
EndWhile
```

This uses a stronger PID configuration suitable for long straight travel.

### 9.3 Final Wall‑Follow Segment

```vb
while Math.abs(MotorC.GetTacho())<3100 Or Time.Get2()<4
  CAR.select(cw,Rdistance,Ldistance,distance)
  CAR.update_gyro_dis(...)
EndWhile
```

After all motion completes, the gyro value is displayed for debugging:

```vb
LCD.Text(1, 0, 0, 2, "Angle: " + gyro)
```

---

# 10. Summary

This README has presented a dense, section‑balanced explanation where every part of the program — imports, globals, sensor threads, initialization, navigation logic, turn handling, and end‑phase behavior — is documented with the same depth and weight as the original code structure.  
The goal is to provide a future team member with a fully technical interpretation that mirrors the coding style, control flow, and reasoning used throughout the program.

