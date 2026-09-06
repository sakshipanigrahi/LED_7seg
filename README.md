# LED_7seg
# Lab 3 — RGB LED + 7-Segment Display Finite State Machine

## 1. What This Program Does (in plain words)

Think of this program like a **traffic controller with two buttons**:

- The board has one **RGB LED** (a light that can be red, green, blue, or a mix).
- It also has a **4-digit 7-segment display** (like a tiny digital clock face) that shows what's happening.
- There are **two buttons**: SW1 and SW2.

The rules are simple, just like a game:

| If you... | Then... |
|---|---|
| Press **SW1** alone (while running) | The LED **blinks faster or slower** (there are 8 speed levels) |
| Press **SW2** alone (while running) | The LED **changes colour** (there are 8 colours, and one of them is "off") |
| Press **SW1 AND SW2 at the same time** | The LED **freezes** (pauses) — it stops blinking and just holds its colour |
| Press **both again** while paused | It **starts running again** |

The display always shows the current "state" of the machine, like a status label:

- Module 1 (leftmost digit) → `r` (running) or `P` (paused)
- Module 2 → current colour number (0–7)
- Module 3 → the letter `S` (just a label, short for "Speed")
- Module 4 (rightmost) → current speed number (0–7)

## 2. Hardware Map (which wire does what)

| Port Pin | Job |
|---|---|
| PF1, PF2, PF3 | Drive the Red, Blue, Green parts of the LED |
| PF4 | SW1 button input |
| PF0 | SW2 button input |
| PB0–PB7 | Turn on/off the 7 segments (+ dot) of whichever digit is currently lit |
| PA4–PA7 | Choose **which one** of the 4 digit modules is currently lit |

**Why only one digit is "chosen" at a time (PA4–PA7)?**
Because all 4 digits share the same 8 segment wires (PB0–PB7). The trick — called **multiplexing** — is to light up only one digit at a time, but do it so fast (hundreds of times per second) that your eye can't tell they aren't all lit together. It's the same idea as a flip-book: one picture at a time, shown quickly, looks like smooth motion.

## 3. Two Ideas You Need Before Reading the Code

### 3.1 "Delay loops" — how the program waits without a clock chip

There is no built-in stopwatch on this simple setup, so the program "wastes time on purpose" by counting up an empty loop:

```c
for (j = 0; j < 3180; j++);   /* does nothing, just counts to 3180 */
```

This was **measured on the real board**: counting from 0 to 3180 takes about **1 millisecond** (1/1000th of a second), *only* when the compiler optimisation is set to `-O0` (no optimisation). If a faster optimisation setting were used, the compiler might notice the loop does nothing and delete it — so this number is only trustworthy at `-O0`.

Analogy: it's like saying "count to 3180 sheep to know one millisecond has passed." It only works if nobody speeds up your counting.

### 3.2 "Polling" — how the program checks a button

The program doesn't get "interrupted" when you press a button. Instead, every millisecond, it **asks**: "Is the button pressed right now?" This is called polling — like a teacher asking "any questions?" once a minute instead of you being able to raise your hand instantly.

```c
s1now = GPIO_PORTF_DATA_R & 0x10;   /* read SW1 pin right now */
```

Because a button is **read-only-active-low** here (0 = pressed, 1 = not pressed, due to pull-up resistors), the code checks `s1now == 0` to mean "SW1 is currently pressed."

## 4. Program Flow (step by step)

1. Turn on the clocks and set up the pins (input/output directions).
2. Loop forever:
   - Turn the LED **on** with the current colour.
   - Wait for however long the current speed level says (this is the "on" half of the blink).
   - **While waiting**, keep checking the buttons every 1 ms, and keep refreshing the display every 1 ms.
   - Turn the LED **off**.
   - Wait again (the "off" half of the blink).
   - Repeat.

## 5. Problems Faced and How They Were Solved

### 5.1 The "Both Buttons Pressed" Timing Problem (the hardest one)

**The problem:**
Human fingers are not perfect. If you try to press SW1 and SW2 "at the same time," they are almost never pressed on the *exact same* 1-millisecond check. One will land a few milliseconds before the other.

Example — imagine the checks happen once per millisecond, and you press both buttons "together":

| Time (ms) | SW1 | SW2 | What the naive code sees |
|---|---|---|---|
| 0 | up | up | nothing |
| 1 | **down** | up | "Oh, SW1 alone was pressed!" → wrongly changes speed |
| 2 | down | **down** | now both are down, too late — speed already changed |

So a simple "check if both are down right now" approach **misfires**: it treats a "both pressed together" gesture as "SW1 pressed" (or "SW2 pressed") by mistake, because it reacted the instant it saw the *first* button go down, without giving the second button a chance to arrive.

**Approaches tried, in order, and why each was replaced:**

1. **`seen1` / `seen2` accumulator flags** — tried to "remember" that a button was seen recently. Rejected: made the logic harder to reason about without actually fixing the timing gap.
2. **A fixed 30 ms "settle" delay** after seeing any press, before deciding what happened — still failed whenever the two presses were staggered by more than the settle window, or landed awkwardly around its edges.
3. **A single `both` flag that toggles pause immediately when both are seen down at once, and holds off single-button actions until both buttons are released** — closer, but still had the root problem: it could still fire a single-button action if the second finger simply arrived late.

**The fix that worked — "wait-and-see" window:**

Instead of reacting the instant *one* button goes down, the code now **makes a note and waits** for `WAIT_MS` (50 ms) to see if the *other* button also goes down in that window:

```c
else if (s1now == 0 && sw1 != 0 && paused == 0)
{
    waiting = 1;        /* SW1 went down — note it, but don't act yet */
    waitCount = 0;
}
```

- If **both** go down while one is still "pending," it's treated as a simultaneous press → pause/unpause immediately, and the pending note is cancelled.
- If the wait window (50 ms) runs out and the partner button never came → *now* it's safe to treat it as a genuine single-button press, and the speed or colour action fires.

Analogy: it's like waiting a beat before assuming your friend isn't coming to a meeting spot — you give them a short grace period before deciding they're not showing up.

**Trade-off to be aware of:** the longer `WAIT_MS` is, the more reliably it catches staggered "simultaneous" presses — but it also makes single-button presses feel slightly delayed, since the program is intentionally holding off action during that window. `WAIT_MS = 50` was chosen as a practical middle ground and may need tuning on real hardware.

### 5.2 The Rate/Timing Granularity Issue

**The problem:**
The blink speed table (`rates[]`) stores how many "ticks" to wait for each speed level. But a "tick" is only meaningful once you know how long each pass through the loop actually takes.

Early on, the length of one pass through the loop (the "tick") changed — for example, moving from a display-refresh-only loop to one that also polled switches every pass changed how long each iteration actually took in real time. When that happened, **every blink speed silently got faster or slower**, even though the `rates[]` numbers themselves hadn't changed — because the *meaning* of "1 tick" had changed underneath them.

**The fix:**
Whenever the timing of the inner loop changes, the `rates[]` values must be **rescaled to match**. In the final version, the inner "on"/"off" wait loop is calibrated so that **each pass of the outer counting loop (`i`) is about 1 ms**, via the `TICK = 3180` inner constant. The `rates[]` array was then written directly in milliseconds (e.g. `2500, 2000, 1500 … 100`), so speed level 1 blinks roughly every 2.5 seconds and speed level 8 blinks roughly every 0.1 seconds.

**Lesson:** any time the "cost" of one loop pass changes, every table of counts based on that loop (like `rates[]`) must be re-derived — otherwise the numbers stay the same but the real-world timing quietly drifts.

### 5.3 Other Issues Faced

**a) Reading "is it pressed right now" vs. "did it just get pressed"**
Two different questions needed two different techniques:
- *"Did the button just get pressed this instant?"* (an edge) — done by comparing the current reading to the last saved reading (`sw1`/`sw2`).
- *"Are both buttons down at this exact moment?"* (a level) — done by reading the raw pin value directly (`GPIO_PORTF_DATA_R & ...`) rather than relying on the "just changed" flags, since simultaneous-press detection needs to know the current state, not just whether something changed.

**b) `if` vs. `else if` ordering for combined gestures**
Using a separate plain `if` for every condition meant more than one branch could fire in a single pass (e.g., "both pressed" and "SW1 pressed" both being treated as true at once). The fix was to check the **combined condition first**, as a plain `if`, with the single-button checks written as `else if` — so only one branch can ever run per check.

**c) Keeping the display flicker-free**
`ShowDigit()` only lights **one** of the 4 digit modules per call, and advances an internal `turn` counter (0→1→2→3→0…) so the next call lights the next module. This counter increment had to be placed **once, at the very end of the function, outside every `if` branch** — otherwise it was possible for a digit to be skipped or repeated, causing visible flicker or a "stuck" digit.

## 6. Final Design Summary

- **State variables:** `colorIndex` (0–7), `rateIndex` (0–7), `paused` (0/1).
- **Simultaneous-press detection:** handled via the `waiting` / `waitCount` "wait-and-see" mechanism (§5.1).
- **Blink timing:** controlled by `rates[]` (in milliseconds) combined with the `TICK = 3180` constant that represents ~1 ms at `-O0` (§5.2).
- **Display refresh:** `ShowDigit()` is called once per millisecond tick inside the delay loop, cycling through all 4 modules roughly every 4 ms (~250 Hz refresh) so the whole display appears steady and flicker-free.
- **Pause behaviour:** while paused, a dedicated inner `while (paused == 1)` loop keeps refreshing the display and watching only for the "both pressed again" gesture to resume — single-button presses are structurally ignored while paused.
