# LED_7seg
# Lab 3 — RGB LED + 7-Segment Display Finite State Machine

## 1. Overview

This program implements a finite state machine on the TM4C123GH6PM (Tiva C LaunchPad, EduARM4 add-on board) that controls an RGB LED and a 4-digit multiplexed 7-segment display using two push-button inputs.

## 2. Hardware Pin Map

| Port Pin | Function |
|---|---|
| PF1, PF2, PF3 | RGB LED — Red, Blue, Green outputs |
| PF4 | SW1 input |
| PF0 | SW2 input |
| PB0–PB7 | 7-segment segment data (a–g, dp) for the currently active digit |
| PA7 | Digit module select — Module 1 (status: `r` / `P`) |
| PA6 | Digit module select — Module 2 (colour index) |
| PA5 | Digit module select — Module 3 (`S` label) |
| PA4 | Digit module select — Module 4 (rate index) |

## 3. Features

- **8 blink rates.** SW1, pressed alone, advances the LED blink rate through 8 levels (`rates[]`), wrapping after the 8th level back to the 1st. The active level is shown on Module 4 as a digit from `1` to `8`. The sequence, from slowest to fastest, is:

  | Displayed digit | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
  |---|---|---|---|---|---|---|---|---|
  | Half-cycle period (ms) | 2500 | 2000 | 1500 | 1250 | 1000 | 500 | 250 | 100 |

- **8 colours.** SW2, pressed alone, advances the LED colour through 8 values (`colors[]`), wrapping after the 8th value back to the 1st. Colour index 0 corresponds to the LED fully off; the display still shows digit `0` for this state rather than leaving the digit blank. The active colour index is shown on Module 2 as a digit from `0` to `7`. The sequence, derived from the PF1 (Red), PF2 (Blue), PF3 (Green) bit pattern written to `colors[]`, is:

  | Displayed digit | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
  |---|---|---|---|---|---|---|---|---|
  | Colour | Off | Red | Green | Blue | White | Magenta | Yellow | Cyan |
- **Pause / resume.** SW1 and SW2 pressed together toggle a paused state. While paused, the LED stops blinking and remains switched on, holding its current colour steady rather than going dark. Pressing both buttons together again resumes blinking from where the cycle left off.
- **Status display.** Module 1 shows `r` while running and `P` while paused.
- **Label display.** Module 3 always shows the letter `S`, identifying Module 4 as the rate digit.
- **Multiplexed display refresh.** All four digit modules are refreshed in rotation at approximately 250 Hz, so the display appears continuously lit without visible flicker.
- **Tick.** A shared ~1 ms unit of time, `TICK` (3180 empty-loop iterations, valid only at `-O0` optimisation), used as the common timing base for the display refresh, the blink rates in `rates[]`, and the button wait window `WAIT_MS`.
- **Waiting (button hold-off window).** When a single button is seen going down, its associated action (rate or colour change) is not applied immediately. Instead a wait counter (`waiting`, `waitCount`) holds it for `WAIT_MS` (50 ticks, ≈50 ms), giving the second button a window to arrive and be recognised as a simultaneous press instead. If no second button arrives within the window, the held action is committed. This also serves the function of debouncing: once the window starts, a bounced re-press on the same switch does not restart it — the window simply continues waiting for a press on the other switch, so bounce does not produce multiple actions.

## 4. Code Flow

1. **Clock and pin initialisation.** Enable clocks for Ports A, B, and F (`SYSCTL_RCGC2_R`). Unlock and configure PF0 for use as a digital input (NMI override). Configure Port F: PF1–PF3 as outputs (LED), PF0 and PF4 as inputs with pull-ups enabled (switches). Configure Port A: PA4–PA7 as digital outputs (digit select). Configure Port B: PB0–PB7 as digital outputs (segment data).
2. **Main loop — ON half.** The LED is switched on with the current colour. A `for` loop runs for a number of iterations equal to the current rate value (`rates[rateIndex]`), with each iteration representing approximately 1 ms:
   - Read the current state of SW1 and SW2.
   - Evaluate button state against the previous reading to detect a new simultaneous press (toggles `paused`), a new single SW1 press (starts the rate-change wait window), or a new single SW2 press (starts the colour-change wait window).
   - If a wait window is active, increment its counter; if it reaches `WAIT_MS` without a simultaneous press occurring, commit the pending rate or colour change.
   - Store the current button readings for the next iteration's comparison.
   - If `paused` is set, enter an inner loop that continues refreshing the display and checking only for the simultaneous-press gesture to resume, ignoring single-button input.
   - Re-assert the LED colour, call `ShowDigit()`, and execute the `TICK`-length delay loop (~1 ms).
3. **Main loop — OFF half.** Identical in structure to the ON half, except the LED is switched off at the start of the loop and switched off again after each tick. If `paused` is set during this half, the inner pause loop forces the LED to display the current colour for the duration of the pause; on resume, the LED is explicitly set back to off so the OFF half continues correctly.
4. **Repeat** from step 2 indefinitely.

## 5. The `turn` Variable in `ShowDigit()`

`ShowDigit()` updates exactly one of the four digit modules per call, using a `static` variable `turn` to track which module is due for update:

| `turn` value | Module updated | Content shown |
|---|---|---|
| 0 | Module 2 (PA6) | Colour index digit |
| 1 | Module 3 (PA5) | `S` label |
| 2 | Module 4 (PA4) | Rate index digit |
| 3 | Module 1 (PA7) | Status (`r` or `P`) |

On each call, all digit-select lines are first cleared, then the segment data and digit-select line corresponding to the current `turn` value are set. `turn` is incremented exactly once, at the end of the function, outside all conditional branches, and wraps from 3 back to 0. Since the function is called once per ~1 ms tick, this produces a full 4-module refresh cycle roughly every 4 ms.

## 6. Design Considerations

### 6.1 Simultaneous Press Detection

Button state is sampled once per 1 ms tick. Two presses intended by the user to be simultaneous rarely land on the same sampling instant — one is typically registered one or more ticks before the other. A direct "act as soon as one button is seen down" implementation therefore misidentifies a two-button gesture as a single-button press before the second button is detected.

This is addressed by deferring action on a single-button press: when one button is seen going down, the event is recorded and a wait counter starts, rather than the corresponding action (rate or colour change) firing immediately. If the second button goes down while the wait counter is active, the pair is treated as a simultaneous press and `paused` is toggled instead. If the wait counter reaches `WAIT_MS` (50 ticks, ≈50 ms) without the second button appearing, the single-button action is committed. This introduces a fixed latency of up to `WAIT_MS` on single-button actions in exchange for reliable simultaneous-press detection.

### 6.2 LED State During the OFF Half While Paused

The blink cycle alternates between an ON half (LED lit) and an OFF half (LED dark). If the pause gesture is registered during the OFF half, holding the LED state unchanged at the moment of pause results in the LED remaining dark for the duration of the pause, which is visually indistinguishable from a fault condition.

This is addressed by explicitly forcing the LED to the current colour value for the duration of the pause loop when pause occurs during the OFF half, so that the paused state is always visually represented as a steady, lit LED regardless of which half of the cycle it was triggered in. On resume, the LED is explicitly set back to off before the OFF half continues, so the blink timing is not altered by the pause.
