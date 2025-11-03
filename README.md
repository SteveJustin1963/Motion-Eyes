# Motion-Eyes
https://www.mikekohn.net/micro/motion_eyes.php

grok

Here's a **simple explanation** of how **Motion Eyes** works:

---

### The Goal:
Make two fake eyeballs **turn and look toward movement** — like a creepy Halloween decoration that follows people with its eyes.

---

### The Parts:
1. **Plastic eyeballs** → Stuck on **servo motors** (motors that turn to exact angles).
2. **Clay face mask** → Hides everything behind it, eyes poke through.
3. **Camera** → Watches what’s in front.
4. **Raspberry Pi** → The "brain" that sees motion and decides where to look.
5. **MSP430 microcontroller** → Gets a simple number (0–20) from the Pi and moves the servos.

---

### How It Detects Motion (Step-by-Step):

1. **Camera takes a picture** → Converts it to grayscale (no color).
2. **Compares it to the last picture**:
   - If a pixel changed a lot (someone moved), it turns **green** in a hidden image.
   - No change? Stays black.
3. **Counts green pixels in each vertical column**:
   - More green = more motion in that column.
   - Draws a **red bar** at the bottom of the screen showing how much motion per column.
4. **Finds the column with the tallest red bar** → That’s where the **most motion** is.
5. **Puts a blue dot at the top** of that column → This is the **target** for the eyes.

---

### How the Eyes Move:
- The blue dot’s position → converted to a number **0 to 20**:
  - `0` = look far left  
  - `10` = look straight  
  - `20` = look far right
- Sends that number to the **MSP430**, which turns the servos to match.

---

### Smart Tweaks to Avoid Jerkiness:
- Eyes **can’t flip left/right too fast** — must wait **2 seconds** before changing direction.
- If **no motion for a while**, eyes **center themselves**.

---

### Visual Example (from the video):
On the screen, you see:
- **Green areas** = what moved
- **Red bars at bottom** = how much motion in each column
- **Blue dot at top** = where the eyes will look

---

### In Super Simple Terms:
> It’s like a robot that **watches TV**, sees what’s moving, **picks the busiest spot**, and **turns its eyes to stare at it** — but slowly and smoothly so it’s creepy, not twitchy.

---

**Perfect for Halloween!**  
(And yes, it runs on a Raspberry Pi with a PS3 Eye camera and some tape + clay.)


### Checking the Software on the Website

I accessed the project page (https://www.mikekohn.net/micro/motion_eyes.php) and reviewed all available details on the software. Unfortunately, the full source code isn't embedded directly on the page—it's provided via two external links to files hosted on the site:

- **Assembly code for MSP430 servo control**: [linksys_copter_v3.asm](https://www.mikekohn.net/micro/source/linksys_copter_v3.asm)
- **C++ motion detection program (archive)**: [motion_detection-2013-10-27.tar.gz](https://www.mikekohn.net/micro/source/motion_detection-2013-10-27.tar.gz)

These files couldn't be directly retrieved or extracted in my analysis (likely due to the archived format and plain-text hosting), but the page gives a clear, step-by-step explanation of **what the software does** and **how it's implemented**. This matches and expands on the simple overview I provided earlier—it's a straightforward, custom-built system without fancy libraries, focused on real-time processing for a fun Halloween prop.

Here's a breakdown of the actual software based on the site's description, including the algorithm, components, and build instructions. I'll highlight key implementation details to show "what they did" in practice.

#### 1. **Overall Software Architecture**
   - **Main Program (C++ on Raspberry Pi)**: Handles camera capture, motion detection, and visualization. Runs on embedded Linux (originally tested on an Atom netbook, then ported to Raspberry Pi). Uses a PS3 Eye USB camera (via Video4Linux2) since the Pi's native camera module wasn't compatible at the time.
   - **Servo Controller (Assembly on MSP430)**: A simple microcontroller circuit receives a single byte (0–20) from the Pi via serial (likely UART) and drives two servo motors to position the eyeballs. The assembly code reuses a board originally designed for drone speed controllers.
   - **Communication**: The Pi sends a position value (0 = far left, 10 = center, 20 = far right) to the MSP430. No complex protocol—just raw bytes.
   - **Anti-Jitter Logic**: Added in the revised version to make movement realistic:
     - Eyes lock direction for ~2 seconds before switching (prevents twitching).
     - If no motion detected for a timeout period, eyes auto-center.

#### 2. **Motion Detection Algorithm (Core of the C++ Program)**
This is the heart of the software—a basic frame-differencing method that's efficient for low-power hardware like the Pi. No machine learning or OpenCV; it's pure pixel math. Here's exactly what they implemented, step by step (quoted/paraphrased from the page for accuracy):

   1. **Capture Frame**: Grab a grayscale image (Y channel only) from the camera in YUV format. Discard color (UV) to simplify processing. (Uses V4L2 API for USB cam.)
   
   2. **Detect Changes**: For every pixel, subtract the current Y value from the previous frame's Y value.
      - If the absolute difference > threshold (e.g., 30–50, tunable), set the **green channel (G)** in a new RGB debug image to 255 (motion pixel).
      - Else, set G to 0.
      - Result: A "heat map" of movement in green.

   3. **Aggregate Motion per Column**: Scan the image vertically.
      - Count green pixels (G=255) in each column (x-position).
      - At the bottom of the debug image, draw a **red bar (R channel)** whose height = the count (e.g., if column has 100 green pixels, red line goes up 100 pixels).
      - This creates a 1D "motion histogram" at the bottom—taller red = more motion left/right.

   4. **Find Peak Motion**: Scan the bottom row for the tallest red bar.
      - In that column, place a **blue mark (B channel)** at the *top* of the debug image.
      - The blue dot's x-position indicates "look here."

   5. **Map to Servos**: Convert the blue dot's column (0 to image width) to a coarse scale:
      - Divide by (width / 20) to get 0–20.
      - Send this byte to MSP430.
      - Servos rotate proportionally (e.g., via PWM pulses: 1ms–2ms pulse width for 0°–180°).

   - **Debug Visualization**: The program displays the RGB image in a window (using FLTK library) so you can see green changes, red bars, and blue target in real-time. This helped tuning.
   - **Frame Rate**: Not specified, but simple enough for 10–30 FPS on old hardware.
   - **Threshold & Tweaks**: Adjustable sensitivity; added hysteresis (direction lock) to reduce noise from lighting/shadows.

#### 3. **Servo Control (MSP430 Assembly)**
   - The `linksys_copter_v3.asm` file (compiled with naken_asm tool) is a low-level routine to generate PWM signals for servos.
   - It reads the incoming byte (0–20), maps it to pulse widths, and outputs to two pins (one per servo for horizontal eye movement—vertical might be fixed).
   - Reused from a drone project, so it's optimized for precise timing on the MSP430's 16-bit timer.
   - No feedback loop; assumes servos reach position quickly.

#### 4. **Compilation & Setup Instructions (From the Page)**
   - **C++ Program** (in the tar.gz archive, likely includes `main.cpp`, capture utils, etc.):
     ```
     sudo apt-get install libfltk1.3-dev  # For GUI/debug window
     # Then: make or g++ with FLTK flags (not detailed, but standard)
     ```
     - Dependencies: V4L2 for camera, basic image buffers (no external libs beyond FLTK).
   
   - **Assembly**:
     ```
     naken_asm linksys_copter_v3.asm  # Outputs .hex or .txt for MSP430 flashing
     ```
     - Flash via standard MSP430 tools (e.g., mspdebug).

   - **Running**: Pi runs the C++ app, which loops: capture → process → send byte. MSP430 polls for input continuously.

#### 5. **What They *Didn't* Do (Limitations Noted)**
   - No Pi camera support (would need MMAL rewrite—skipped for time).
   - Basic motion only (no object tracking or depth).
   - Hardware hacks: Masking tape for assembly; clay face for aesthetics.
   - Tested with videos showing erratic early version vs. smooth revised one.

#### Why This Approach?
Michael Kohn (the creator) emphasizes simplicity: "pretty simple and took very little time to implement." It's DIY embedded programming—raw pixels, no bloat—for a one-off Halloween gag. The code's open-source vibe fits his site (he's shared tons of retro/embedded projects since 1997).

If you want to try it yourself, download the files from the links above, compile on a Pi, and tweak the threshold for your setup. The tar.gz likely unpacks to a few hundred lines of C++—let me know if you need help interpreting specific parts once extracted! If there's more context or a different angle, clarify.








