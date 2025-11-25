# EV3 Future Engineers – Extended Technical Documentation (Paragraph-Focused Version)

This document provides a comprehensive and highly detailed explanation of the EV3 program created for the WRO Future Engineers Open Challenge. Unlike shorter summaries or bullet‑point‑heavy descriptions, this version deliberately emphasizes longer, more narrative paragraphs. The intention is to mirror the conceptual density of the original program itself, ensuring that every portion of the code is represented with an equivalent depth of explanation, while keeping the structure clear and readable.

---

## 1. Project Structure and Module Imports

The program begins by defining the operational context of the robot, specifying the project folder and loading several specialized modules that collectively form the foundation of all high‑level behavior. These imports establish the robot’s capabilities by providing access to external sensor libraries, steering and motion algorithms, and communication interfaces. The color module adds robust RGB color‑sensing that is required for interpreting the orange and blue markers used on the competition field. The Arduino module provides a communication bridge to an external microcontroller which supplies stable gyro angle measurements, addressing the drift and noise commonly associated with the native EV3 gyro sensor. The CAR module encapsulates advanced functions for steering, PID control, wall-distance corrections, and filter utilities. Ultrasonic handles distance measurements for left and right range sensors, and Tool includes general-purpose helper functions that support the broader program. Together, these modules enable a layered control architecture where the main script can focus on high-level decisions while relying on tested utilities for low-level interactions.

```vb
folder "prjs""Clever"
  import "Mods\HTColorV2"
  import "Mods\arduino"
  import "Mods\CAR"
  import "Mods\Ultrasonic"
  import "Mods\Tool"
```

---

## 2. Global Variables and System State

Immediately after the imports, the program defines a collection of global variables that form the shared state between the main logic and background sensor threads. These variables include gyro angle tracking, ultrasonic sensor measurements, steering mode flags, reference distances, target headings, and integer counters that control the robot’s progress across multiple navigation segments. Each of these variables plays a part in maintaining continuity as the robot moves from one control mode to another. For example, `cw` starts in a neutral state so that the robot does not commit to clockwise or counter‑clockwise directional logic until it encounters the appropriate color marker. The ultrasonic readings are initialized to zero and gradually become meaningful once the sensor thread begins constant background updates. The `t_count` variable tracks the number of completed turns, ensuring that the robot progresses through twelve distinct movement phases as required by the challenge’s layout. The `target` variable serves as the reference heading for all gyro‑related control routines, and it is adjusted by ±90 degrees whenever a color marker triggers a turn. The wall distance target, stored in `t_distance`, ensures that the robot maintains an appropriate buffer from the wall during wall‑following segments. Because different functions across the program depend on these shared values, declaring them globally ensures easy access and continuity throughout the robot’s full navigation path.

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

---

## 3. Ultrasonic Sensor Reading Thread

The ultrasonic thread is designed to run continuously and independently of the main program loop, ensuring that left and right distance measurements remain fresh and stable regardless of the computational load of the main navigation logic. Ultrasonic sensors are known for their noisy and occasionally inconsistent measurements, especially when pointed at angled surfaces or narrow walls, so the thread uses a low‑pass filter to smooth the readings. The filter blends newly acquired values with previous ones according to a high smoothing constant, reducing fluctuations that could disrupt wall‑following routines. At the same time, the code periodically resets the filtered reading to the raw measurement to prevent the filter from drifting too far from reality. This balancing act between smoothness and responsiveness is essential because the ultrasonic readings are used directly in steering corrections during the wall‑following phase. The infinite loop structure inside the thread ensures that the sensors are sampled at the highest possible rate, providing rapid updates to the shared `Ldistance` and `Rdistance` variables without interrupting the robot’s main behavior. By offloading this responsibility to a separate thread, the robot gains both efficiency and stability across all movement phases.

```vb
Sub ReadUltrasonic
  alphaDist = 0.9
  counter = 0
  Ultrasonic.getCm(2, Ldistance)
  Ultrasonic.getCm(4, Rdistance)
  While 1=1
    Ultrasonic.getCm(2, rawLdistance)
    Ultrasonic.getCm(4, rawRdistance)
    CAR.lowpass(rawLdistance, Ldistance, alphaDist, Ldistance)
    CAR.lowpass(rawRdistance, Rdistance, alphaDist, Rdistance)
    counter = counter + 1
    If counter >= 5 Then
      Ldistance = rawLdistance
      Rdistance = rawRdistance
      counter = 0
    EndIf
  EndWhile
EndSub
```

---

## 4. Gyro Reading Thread Through Arduino

Just as the ultrasonic thread stabilizes distance values, the gyro thread maintains a reliable reading of the robot’s heading. Using an Arduino-based gyro provides far better stability and accuracy than the built‑in EV3 gyro sensor, particularly over long navigation distances involving repeated 90‑degree turns. The thread retrieves the angle in scaled units, converts it into degrees, applies low‑pass filtering to reduce jitter, and periodically resets the filtered output to avoid long-term drift. Because the robot’s path is heavily dependent on maintaining correct alignment, this thread provides a near‑continuous stream of valid heading information. Each turn, each wall‑following correction, and each straight segment relies on the gyro value maintained by this background thread. Running this process in parallel prevents the main loop from being delayed by sensor reads and ensures the gyro information is always up to date.

```vb
Sub ReadGyro
  alphaGyro = 0.6
  counter = 0
  arduino.GyroArduino(1, gyroRaw)
  gyro = (gyroRaw / 100) - offset
  While 1=1
    arduino.GyroArduino(1, gyroRaw)
    rawGyro = (gyroRaw / 100) - offset
    CAR.lowpass2(rawGyro, gyro, alphaGyro, gyro)
    counter = counter + 1
    If counter >= 10 Then
      gyro = rawGyro
      counter = 0
    EndIf
  EndWhile
EndSub
```

---

## 5. Initialization and Steering Alignment

Before entering the main navigation loop, the program performs several steps to ensure that the robot begins in a stable and predictable configuration. Gyro calibration is performed first to define a clean zero‑degree position, giving all future turns a consistent reference. The steering alignment routine then centers the wheels by locating a known neutral steering position. This is particularly important because many EV3 robots rely on rack‑and‑pinion or servo‑to‑linkage systems that can accumulate mechanical slack over time. To eliminate this slack, the robot performs a repeated straight‑driving routine that physically settles the steering mechanism into alignment. By running hundreds of small steering adjustments in rapid sequence, the robot ensures that its wheels point forward consistently before major movement begins. The drive motor is then activated, and both sensor threads are launched so that real‑time data becomes available for navigation. This thorough initialization ensures that all movement control routines begin from an accurate mechanical baseline, significantly improving the reliability of later PID corrections and 90‑degree turns.

---

## 6. Main Navigation Loop Across Twelve Segments

The core behavior of the robot is governed by a loop that continues until it completes twelve navigation segments. At each iteration, the robot reads the color sensor and decides whether to continue straight, follow a wall, correct its heading, or perform a turn. The robot begins in a neutral state where it uses gyro-only control to maintain straight movement while waiting to detect its first color marker. Once a marker is detected, the robot sets its direction mode (`cw`) to either clockwise or counter‑clockwise, which then determines the path the robot will follow for all subsequent segments. In each cycle, the robot selects whether it should rely exclusively on gyro correction or whether it should perform wall following by blending gyro and ultrasonic information. Large deviations from the target heading trigger a more aggressive gyro correction routine to bring the robot back into alignment before it resumes normal wall-following. This dynamic selection between different steering modes allows the robot to maintain robustness even as environmental conditions vary. The long paragraphs in this section mirror the complexity and importance of the control logic, emphasizing the layered decision structure that guides the robot across all twelve segments.

---

## 7. First-Time Direction Selection via Color Sensor

The shift from straight mode to directional mode is a key moment in the robot’s path. By reading the RGB sensor continuously, the robot identifies whether the first encountered marker is orange or blue. These colors encode the direction in which the robot must turn throughout all future segments. Switching `cw` from 3 to either 0 or 1 effectively locks the robot into its long-term navigation strategy. This single decision determines which wall the robot will follow, how `CAR.select` chooses between left and right ultrasonic readings, and whether each upcoming turn will adjust the target heading by +90 or −90 degrees. The importance of this transition is why the code isolates the condition to only occur when `cw` is still equal to 3, preventing accidental mode changes later in the course.

---

## 8. Executing Reliable 90-Degree Turns

The robot performs a 90‑degree turn only when it detects a marker color that corresponds to its active direction mode. Before adjusting the heading target, the robot drives forward a fixed encoder distance to clear the marker and stabilize its position. This precaution helps ensure the turn begins from a predictable location. Once clear, it updates the `target` by adding or subtracting 90 degrees depending on turn direction. The PID routines handle the rotation naturally by steering toward the new target value. Each successful turn increments `t_count`, allowing the robot to proceed to the next navigation segment. The repeated structure of this behavior mirrors the modularity of the field layout itself, with each turn forming one phase of the larger twelve‑segment journey.

---

## 9. End Phase and Final Movements

After completing twelve turns, the robot transitions into its end‑phase sequence. This section of the program uses controlled straight driving, followed by gyro-stabilized motion, and finally a wall-following stretch to guide the robot toward its final destination. The combination of encoder-based thresholds and time-based conditions ensures that the robot progresses regardless of minor variations in wheel traction or surface quality. Once the sequence concludes, the robot displays its final angle on the LCD for debugging, then stops its drive motor while keeping the system active for observation. This multi‑stage conclusion ensures both precision and stability as the robot completes its challenge.

---

## Final Notes

This paragraph-oriented documentation was intentionally structured to provide smooth narrative explanation rather than relying heavily on bullet points. Each section mirrors the logic density of the original program, giving future team members not only a clear understanding of how each component works, but also why the code is structured in this way. By expanding every concept into larger, more thorough paragraphs, this README serves as a long‑form reference manual for anyone learning to maintain, modify, or extend this navigation system.
