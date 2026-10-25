# Technical Notes

Implementation details for the [x86 Traffic Light Controller](../README.md).

| Section | Topic |
| ------- | ----- |
| [1](#1-light-output-port-4) | Light output (port 4) |
| [2](#2-video-memory) | Video memory |
| [3](#3-timing) | Timing |
| [4](#4-keyboard-and-modes) | Keyboard and modes |
| [5](#5-program-flow) | Program flow |
| [6](#6-variables) | Variables |
| [7](#7-bios-interrupts-used) | BIOS interrupts used |
| [8](#8-troubleshooting) | Troubleshooting |
| [9](#9-code-map) | Code map |

---

## 1. Light output (port 4)

The lights are driven by one word written with `OUT 4, AX`. Only 12 of the 16 bits are used. They form four groups of three bits, one group per approach.

Inside each group the bits are, from high to low:

| Bit in group | Light |
| ------------ | ----- |
| High | Green |
| Middle | Yellow |
| Low | Red |

The groups are paired by axis. The patterns used in the code show which groups belong together:

| Bits | Group | Axis |
| ---- | ----- | ---- |
| 11-9 | 1 | East-West |
| 8-6 | 2 | North-South |
| 5-3 | 3 | East-West |
| 2-0 | 4 | North-South |

This table is derived from the patterns below. Which physical lamp (north or south, east or west) maps to which group inside an axis depends on the output device.

| State | Binary | Hex | Result |
| ----- | ------ | --- | ------ |
| NS green | `0000_0011_0000_1100` | `030Ch` | Groups 2 and 4 green, groups 1 and 3 red |
| EW green | `0000_1000_0110_0001` | `0861h` | Groups 1 and 3 green, groups 2 and 4 red |
| All yellow | `0000_0100_1001_0010` | `0492h` | Middle bit of every group |
| All red | `0000_0010_0100_1001` | `0249h` | Low bit of every group |
| Emergency | `0000_0100_1001_0010` | `0492h` | All yellow, flashing at 0.5 s |
| Night | `0000_0100_1001_0010` | `0492h` | All yellow, blinking at 1 s |

```asm
mov ax, transition1        ; 0000_0011_0000_1100b  (NS green, EW red)
out 4, ax
```

## 2. Video memory

Text mode 80×25 at segment `0B800h`. Each cell is two bytes: the character, then the attribute.

```
offset = (row × 160) + (column × 2)
```

Example: row 5, column 10 gives `5 × 160 + 10 × 2 = 820`.

The attribute byte has the background in the high nibble and the foreground in the low nibble.

| Attribute | Foreground on background | Used for |
| --------- | ------------------------ | -------- |
| `0Ah` | Light green | Timers, positive messages |
| `0Bh` | Light cyan | Statistics, information |
| `0Ch` | Light red | Warnings, stop messages |
| `0Eh` | Yellow | Cautions, rush hour |
| `0Fh` | White | General text, help |
| `0CEh` | Yellow on red (blinks in default text mode) | Emergency alert |

```asm
PrintTextColor:
    mov ax, 0B800h
    mov es, ax              ; ES = video segment
    mov ax, row
    mov dx, 160
    mul dx                  ; AX = row × 160
    mov di, ax
print_loop:
    lodsb                   ; next character from DS:SI
    cmp al, 0
    je  done
    mov es:[di], al         ; character
    inc di
    mov es:[di], bl         ; attribute
    inc di
    jmp print_loop
done:
    ret
```

Screen layout:

| Row | Content |
| --- | ------- |
| 0 | Control key help |
| 1 | Statistics and countdown timers |
| 3-20 | Scrolling message area, wraps back to row 3 when full |
| 24 | Countdown display |

## 3. Timing

`INT 15h` function `AH=86h` waits for the number of microseconds in `CX:DX`. One second is `000F4240h` (1,000,000).

```asm
DelayOneSecond:
    mov al, 0
    mov cx, 000Fh           ; high word
    mov dx, 4240h           ; low word
    mov ah, 86h
    int 15h
    ret
```

| Procedure | Used in | What it does |
| --------- | ------- | ------------ |
| `DelayWithCountdown` | All-red phase | Beeps each second and shows the countdown |
| `DelayWithLiveTimer` | Green phases | Updates the NS or EW timer each second |
| `DelayWithBothTimers` | Yellow phases | Updates both timers together |

## 4. Keyboard and modes

`CheckModeKeys` reads the keyboard with `INT 16h` (`AH=01h` to test, `AH=00h` to read). It is called after each one-second delay, so a key press is handled within about one second.

| Key | Action |
| --- | ------ |
| `E` | Set `emergency_mode`, save the current mode in `previous_mode` |
| `N` | Clear `emergency_mode`, restore `previous_mode` |
| `P` | Add 3 seconds if `current_phase` is 1 or 3, otherwise deny |
| `R` | Toggle `rush_hour_mode` and switch the timing values |
| `M` | Set `night_mode` |
| `D` | Clear `night_mode` |
| `ESC` | Set `program_exit` |

Pedestrian requests are granted only during a green phase. The extension is added to the running countdown, so in Rush Hour mode the north-south timer can exceed 8 seconds.

## 5. Program flow

```
Initialization (clear screen, show help, reset timers)
        |
   All-red phase (5 s)
        |
   Main loop  <------------------------------+
     |  emergency?  -> emergency loop         |
     |  night?      -> night loop             |
     |  exit?       -> exit program           |
     v                                        |
   1. NS green                                |
   2. All yellow                              |
   3. EW green                                |
   4. All yellow                              |
   5. All red  -------------------------------+
```

## 6. Variables

| Variable | Type | Meaning |
| -------- | ---- | ------- |
| `emergency_mode` | byte | Emergency override active |
| `night_mode` | byte | Night blinking active |
| `rush_hour_mode` | byte | Rush Hour timing active |
| `ped_request` | byte | Pedestrian request pending |
| `program_exit` | byte | Exit requested |
| `previous_mode` | byte | Mode to return to after Emergency |
| `current_phase` | byte | 0 all red, 1 NS green, 2 yellow, 3 EW green |
| `ns_timer`, `ew_timer` | byte | Displayed countdowns |
| `cycle_counter` | word | Completed cycles |
| `ns_cars`, `ew_cars` | word | Estimated vehicles (green seconds × 2) |
| `ns_green_time`, `ew_green_time`, `yellow_time` | byte | Timing values, changed by Rush Hour mode |

## 7. BIOS interrupts used

| Interrupt | Function | Purpose |
| --------- | -------- | ------- |
| `INT 10h` | `AH=06h` | Scroll or clear the screen |
| | `AH=02h` | Set cursor position |
| | `AH=0Eh` | Write a character (also used for the beep) |
| `INT 15h` | `AH=86h` | Microsecond delay |
| `INT 16h` | `AH=01h` | Check for a key press |
| | `AH=00h` | Read the key |
| `INT 21h` | `AH=4Ch` | End the program |

## 8. Troubleshooting

| Problem | Cause | Fix |
| ------- | ----- | --- |
| Delay does not wait | `INT 15h` call without register setup | Set `AL` to 0 before the call, as in `DelayOneSecond` |
| Vehicle counter wraps | 8-bit arithmetic | Use 16-bit: `XOR AH, AH` then `MUL BX` |
| Text runs off the screen | No row check | Compare the row with the maximum and reset it to 3 |
| Mode does not return after Emergency | `previous_mode` not saved | Save it before setting `emergency_mode` |
| Timers stay frozen | Wrong phase test | Check the value of `current_phase` |

## 9. Code map

```
traffic_lights.asm
├── .data
│   ├── Messages (pedestrian, emergency, mode)
│   ├── Signal patterns (transition1-4, all_red, emergency)
│   ├── Timing and counters
│   └── Screen offsets and flags
└── .code
    ├── Main control:  start, main_loop, emergency_loop, night_loop, exit_program
    ├── Input:         CheckModeKeys
    ├── Display:       PrintTextColor, UpdateTimerDisplay, UpdateStatistics,
    │                  PrintDigitToScreen, PrintNumberToScreen,
    │                  ClearScreen, ClearMessageArea
    └── Timing:        DelayWithCountdown, DelayWithLiveTimer, DelayWithBothTimers
```
