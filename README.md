# NUC140 Pokémon-Themed Battle Game

A turn-based battle game for the Nuvoton **Nu-LB-NUC140** learning board, written in bare-metal C against the Nuvoton BSP: sprites and status text on the 128×64 monochrome LCD, a 3×3 matrix keypad for all input, and a per-turn countdown driven onto the 7-segment display by a second hardware timer.

> Course project, 2024–2025.

**Demo video:** [YouTube](https://www.youtube.com/watch?v=9VVDpm9JCD0)

---

## Design & Implementation Notes

Every point below is traceable to a specific place in `main.c` / `pokemon.h`.

### 1. Two hardware timers carrying different kinds of time

`Init_Timer0()` opens `TIMER0` in periodic mode using the macros from `MCU_init.h` (`TMR0_OPERATING_MODE`, `TMR0_OPERATING_FREQ` = 1 Hz). `TMR0_IRQHandler()` decrements `battle_count_down` once per second and, when it reaches zero, clears `isPlayerTurn` so the turn passes to the CPU.

`Init_Timer1()` opens `TIMER1` in periodic mode at a hard-coded 200 Hz, giving a 5 ms tick. `TMR1_IRQHandler()` carries all the display and input housekeeping.

So the two timers are not interchangeable: one runs game-rule time at 1 Hz, the other runs hardware housekeeping at 200 Hz.

`TIMER0` is also not free-running. It is started (`TIMER_Start(TIMER0)`) only when a player turn begins with `battle_count_down == 10`, stopped (`TIMER_Stop(TIMER0)`) the instant the player commits a move with key 5, and reset to 10 at the end of the CPU turn — the countdown only advances while the player is actually deciding.

### 2. Keypad: column strobing in the timer ISR, row detection by GPIO edge interrupt

The 3×3 keypad is scanned with no polling in the main loop, split across two interrupt handlers:

- `TMR1_IRQHandler()` — every 20 ticks (100 ms) it advances `index_key_scan` and pulls exactly one column line (`PA3` / `PA4` / `PA5`) low, then re-enables `GPAB_IRQn`.
- `GPAB_IRQHandler()` — the row lines `PA0` / `PA1` / `PA2` are configured for falling-edge interrupts. The handler reads `PA->ISRC` to learn which row fired, then tests which column pin is currently low; the (row, column) pair resolves to a key number 1–9 written into `keyflag`, and the column pin is driven back high before returning.

Bounce is handled in hardware rather than with delay loops — `Init_KEY()` sets `GPIO_SET_DEBOUNCE_TIME(GPIO_DBCLKSRC_LIRC, GPIO_DBCLKSEL_64)` and `GPIO_ENABLE_DEBOUNCE(PA, BIT0|BIT1|BIT2)`. All six pins are put in quasi-bidirectional mode (`GPIO_MODE_QUASI`), which is what a matrix keypad needs in order to both drive and sense on the same port.

The result is that game code never touches keypad hardware at all: it reads `keyflag` and clears it.

### 3. Bitmap renderer

`draw_Bmp_axb(x, y, fgColor, bgColor, bitmap[], a, b)` renders an arbitrary `a`×`b` bitmap. The data layout is column-major with 8 rows packed per byte (`bytes_per_col = (b + 7) / 8`): the byte at `bitmap[i + j*a]` holds rows `y + j*8 … y + j*8 + 7` of column `x + i`, and bit *k* of that byte selects the row. A 32×32 sprite is therefore exactly 128 bytes. Draws that would run off the panel are rejected as a whole against `LCD_Xmax` / `LCD_Ymax` rather than clipped per pixel.

Because the routine takes both a foreground and a background colour, the *same* call erases a sprite when both are passed as `0`. That property is what the animation code in the next section is built on.

`print_Line5x7()` and `print_last_line()` sit on top of `printC_5x7()` with an 8 px character pitch and an 11 px line pitch — 16 characters per line on the 128 px-wide panel. `print_last_line()` pins text to `y = 56`, the bottom row, which is where move names are shown during battle.

### 4. Animation by erase-and-redraw

There is no framebuffer; the LCD is written directly, so animation is done by redrawing:

- `attacker_moving()` runs 5 lunge cycles. Each cycle draws the sprite in background colour at its current x (erasing it), draws it in foreground colour 5 px forward, waits, then reverses the pair to bring it home. Player sprites use `back_bmp` at `(0, 20)`; CPU sprites use `front_bmp` at `(95, 0)`.
- `defender_shine()` runs 4 cycles toggling the same sprite between foreground and background colour in place, producing a hit flash.

Both use `CLK_SysTickDelay(250000)` between frames, so the animation is a blocking busy-wait in the main loop while the two timer interrupts keep running underneath it.

### 5. 7-segment countdown: software multiplexing with leading-zero blanking

`TMR1_IRQHandler()` derives `index_5ms = cnt_5ms % 4` — one slot per digit position. Every entry into the handler calls `CloseSevenSegment()` first and then lights at most one digit, so two digits are never driven at once. Slot 2 shows `battle_count_down / 10`, but only when that value is non-zero, so a single-digit count reads as `9` rather than `09`; slot 3 shows `battle_count_down % 10`. With a 5 ms tick and a 4-slot cycle, each digit is refreshed every 20 ms.

The whole display is gated by the `iscount` flag, so the 7-segment stays blank outside a live countdown, and `CloseSevenSegment()` is called again on both end-of-game screens.

### 6. Data model (`pokemon.h`)

```c
struct mov     { char left_name[20]; char type; int atk; char right_name[20]; };
struct Pokemon { char name[20]; unsigned char *front_bmp; unsigned char *back_bmp;
                 int16_t hp; int16_t fixhp; char type; mov movearr[3]; };
```

- Each move stores **two** pre-formatted 16-character display strings, `left_name` and `right_name` — the same move text padded for the player's side of the screen and for the opponent's side. The render path is then a plain string print with no run-time formatting or alignment arithmetic; the cost is a duplicated string per move.
- `front_bmp` / `back_bmp` are pointers, not embedded arrays, so a whole `Pokemon` can be copied by plain struct assignment (`player = bulbasaur;`, `pc = charm;`) without duplicating 128-byte sprite payloads. The back view is drawn for the player and the front view for the opponent — the standard two-sided battle framing.
- `hp` and `fixhp` hold current and maximum HP as separate `int16_t` fields, which reduces the status line to a single `sprintf("HP:%d/%d>")`.
- Types are single `char` codes (`'N'`, `'F'`, `'W'`, `'G'`) rather than an enum, keeping `damage_ratio()` down to direct character comparisons.
- `Init_PMmov()` populates `movearr` at startup rather than in the initialisers, and reuses one shared normal-type move struct across two of the three characters.

### 7. Damage rule

`damage_ratio()` implements a three-type cycle — F beats G, W beats F, G beats W — returning ×2 and setting the `"Super effective!"` banner. `'N'` (normal) moves always return ×1 and blank the banner. Every remaining pairing returns ×0.5: there is no neutral case in the code other than normal-type moves.

Base damage in the main loop is `(mov_index + 1) * 10`, i.e. derived from the list position of the highlighted move, so the third move is always the strongest. (`struct mov` also carries an `atk` field holding the same 10/20/30 values, but it is not read on the damage path.) `damage_ratio()` returns `float`, and this multiply is the only floating-point arithmetic in the program; the product is truncated back into the `int16_t` HP field.

### 8. Randomness without an RNG peripheral

`srand(cnt_5ms)` is called on every `TIMER1` interrupt, so the seed tracks the free-running 5 ms counter. The opponent is then picked with `rand() % 3` after the player confirms a starter, and the CPU's move is picked with `rand() % 3` on each of its turns. Because the seed depends on how many 5 ms ticks have elapsed at the moment of the call, the outcome varies with the player's own input timing rather than being fixed at reset — no hardware RNG or RTC is involved.

### 9. State flow

`main()` is a two-phase structure:

1. **Selection** — `login_page()` draws all three starters at x = 0 / 40 / 80. `select_PM_page()` moves the cursor with keys 4 and 6 and marks the selection by swapping the plain sprite for a framed variant (`bulb_frame`, `char_frame`, `sqr_frame`) instead of drawing a highlight box at run time. The loop exits on key 5.
2. **Battle** — a `while(1)` loop that repaints `Now_battle_page()` each pass, moves `mov_index` within 0–2 on keys 4 and 6, and on key 5 during a player turn runs: apply damage → `attacker_moving()` → `defender_shine()` → repaint → `player_switch()`. The CPU branch runs the same sequence with a random `mov_index` and resets the countdown to 10. Either side reaching HP ≤ 0 clears the LCD, draws the winner's sprite, blanks the 7-segment and returns.

`isPlayerTurn` is the single piece of turn state, and it is written from two places — the main loop via `player_switch()`, and `TMR0_IRQHandler()` on timeout. That second writer is what lets the countdown take the turn away from the player.

### 10. Memory and layout trade-offs

- Nine sprite arrays of 128 bytes each (≈1.1 KB): three characters × (front, back), plus three framed variants used only by the selection screen. Storing the framed copies trades storage for a simpler selection renderer.
- The sprite arrays are declared as plain file-scope `unsigned char[]` rather than `const`, so they are placed in writable data.
- All status strings are fixed 16-byte buffers pre-filled with spaces (`pchp`, `playername`, `playerhp`, `playermove`, `effect`) and rewritten in place with `sprintf`. There is no dynamic allocation anywhere in the program.

---

## Peripherals driven

| Peripheral | Use | Where in the code |
|---|---|---|
| SPI3 → 128×64 monochrome LCD | sprites, HP/status text, banners | `MCU_init.h` (SS0 `PD8`, SCLK `PD9`, MISO0 `PD10`, MOSI0 `PD11`); `init_LCD()`, `draw_Bmp_axb()`, `print_Line5x7()` |
| GPIO Port A (`PA0`–`PA5`) | 3×3 matrix keypad — quasi-bidirectional, falling-edge interrupt, hardware debounce | `Init_KEY()`, `GPAB_IRQHandler()` |
| TIMER0 | 1 Hz turn countdown, started and stopped per turn | `Init_Timer0()`, `TMR0_IRQHandler()` |
| TIMER1 | 200 Hz / 5 ms tick: 7-segment multiplexing, keypad column strobe, RNG reseed | `Init_Timer1()`, `TMR1_IRQHandler()` |
| 7-segment display | two-digit countdown, software-multiplexed | `ShowSevenSegment()` / `CloseSevenSegment()` in `TMR1_IRQHandler()` |
| GPIO Port C (`PC12`–`PC15`) | LED pins configured as outputs and initialised high | `Init_GPIO()` |
| SysTick | blocking inter-frame delays during animation | `CLK_SysTickDelay()` |

Clocking comes from `MCU_init.h`: HXT source, 50 MHz system clock, TIMER0 and TIMER1 both clocked from HXT with divider 1.

This project uses no UART, I²C, ADC or PWM — all input is digital, and there is no audio output.

---

## Build / Run

**Toolchain:** Keil MDK-ARM with the Nuvoton NUC100 Series BSP.

This repository holds the application sources only. To build:

1. Create a Keil MDK-ARM project for the NUC140 device on the Nu-LB-NUC140 board and add the Nuvoton NUC100 Series BSP driver sources.
2. Add `main.c`, and put `pokemon.h` and `MCU_init.h` on the include path.
3. The following headers and their implementations come from the BSP / board support code and are **not** included here: `NUC100Series.h`, `SYS_init.h`, `SYS.h`, `SPI.h`, `GPIO.h`, `LCD.h` (`init_LCD`, `clear_LCD`, `draw_Pixel`, `printC`, `printC_5x7`, `LCD_Xmax`, `LCD_Ymax`) and `Seven_Segment.h` (`ShowSevenSegment`, `CloseSevenSegment`).
4. Build and flash over Nu-Link.

**Controls** (3×3 keypad):

| Key | Action |
|:--:|---|
| `4` | cursor left / previous move |
| `6` | cursor right / next move |
| `5` | confirm starter / attack |

**Gameplay:** pick one of three starters, then alternate turns with the CPU. Each player turn carries a 10-second countdown on the 7-segment display; letting it reach zero hands the turn over. The first side to reduce the other to 0 HP wins.

---

## File structure

```
main.c        Game logic, ISRs, LCD/keypad/timer setup, and sprite bitmap data
pokemon.h     struct mov / struct Pokemon definitions
MCU_init.h    Clock source, SPI3-for-LCD pin mapping, TIMER0-3 configuration macros
LICENSE       MIT
```

---

## Notes

Course project for Microprocessors, Department of Computer Science and Information Engineering, Feng Chia University (Prof. Yi-Wen Wang).

The Nuvoton BSP drivers and `MCU_init.h` used by this project are supplied by Nuvoton and are not the author's work; `main.c` and `pokemon.h` are.

Character names and sprite artwork are used only as placeholder game content for a classroom exercise. This project is not affiliated with, endorsed by, or connected to the owners of those trademarks.

The MIT license in `LICENSE` applies to the author's own source in this repository.
