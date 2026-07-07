# Embedded Systems — OA/MCQ Revision Sheet

---

## 1. ARM Cortex-M Basics

**Concept:** Cortex-M is a 32-bit RISC processor family (M0/M3/M4/M7) used in most microcontrollers (STM32, LPC, Nordic, TI). Harvard architecture, 3-stage pipeline (M0/M3), single-cycle execution for most instructions.

**Key ideas / registers:**
| Item | Detail |
|---|---|
| Registers | R0–R12 (general), R13 (SP), R14 (LR – return address), R15 (PC) |
| Special regs | PSR (Program Status), PRIMASK/FAULTMASK/BASEPRI (interrupt masking) |
| Modes | Thread mode (normal code), Handler mode (ISR execution) |
| Stacks | MSP (Main Stack Pointer – used at reset/ISR), PSP (Process Stack Pointer – used by RTOS tasks) |
| Memory map | Fixed regions: Code (0x0000_0000), SRAM (0x2000_0000), Peripherals (0x4000_0000), System (0xE000_0000 – NVIC, SysTick) |
| Bit-banding | Alias region allows atomic bit set/clear via word access (M3/M4) |
| NVIC | Nested Vectored Interrupt Controller — handles priorities & nesting |
| SysTick | 24-bit down-counting timer built into core, used for OS ticks/delays |

**Workflow (boot sequence):**
1. Reset → PC loaded from vector table entry 1, SP loaded from entry 0.
2. Startup code copies `.data` to RAM, zeroes `.bss`.
3. `main()` called.
4. Peripherals configured via memory-mapped registers (read-modify-write).

**MCQ traps:**
- Cortex-M0 has **no MPU**, no divide instruction (some variants); M4 adds **DSP + FPU (optional)**.
- Thumb-2 instruction set (mix of 16/32-bit) — smaller code size than ARM mode.
- Little-endian by default.

---

## 2. GPIO (General Purpose I/O)

**Concept:** Digital pins configurable as input or output, individually or as a port. Controlled via memory-mapped registers.

**Key registers (typical naming, e.g., STM32-style):**
| Register | Purpose |
|---|---|
| MODER / DDR | Set pin as Input / Output / Alternate Function / Analog |
| ODR (Output Data Reg) | Write value to output pins |
| IDR (Input Data Reg) | Read current pin state |
| BSRR | Atomic set/reset of individual bits (avoids read-modify-write race) |
| PUPDR | Enable internal pull-up/pull-down |
| OTYPER | Push-pull vs Open-drain output |

**Workflow:**
1. Enable peripheral clock for GPIO port (common MCQ: "why does GPIO not work?" → clock not enabled).
2. Set direction (input/output) and mode (push-pull/open-drain/AF).
3. Configure pull-up/down if needed.
4. Read (IDR) or Write (ODR/BSRR).

**GPIO Input vs Output — comparison:**
| Aspect | Input | Output |
|---|---|---|
| Register used | IDR | ODR / BSRR |
| Typical use | Button, sensor digital signal | LED, relay, enable signal |
| Floating risk | Yes — needs pull-up/down | N/A |
| Drive types | N/A | Push-pull (drives both H/L) vs Open-drain (drives only L, needs external pull-up) |

**Common MCQ points:**
- Open-drain output needs external pull-up resistor to read logic HIGH.
- Debouncing needed for mechanical switches (software delay or RC filter).
- Floating input pin gives unpredictable/random readings.

---

## 3. Interrupts

**Concept:** Hardware mechanism that pauses normal execution to service an urgent event (peripheral flag, external pin, timer overflow) via an ISR (Interrupt Service Routine), then resumes.

**Key ideas:**
| Term | Meaning |
|---|---|
| ISR | Function executed on interrupt trigger, registered via vector table |
| IVT (Interrupt Vector Table) | Table of ISR addresses at fixed memory location |
| NVIC | Manages priority, enabling/disabling, nesting of interrupts (Cortex-M) |
| ISER/ICER | Interrupt Set/Clear Enable Registers |
| IP (Priority reg) | Sets interrupt priority (lower number = higher priority typically) |
| Pending flag | Set when interrupt occurs but not yet serviced |
| Interrupt latency | Time from trigger to first instruction of ISR |
| Context save/restore | CPU pushes registers (R0-R3, R12, LR, PC, PSR) onto stack automatically on entry (Cortex-M) |

**Workflow (typical sequence):**
1. Event occurs → peripheral sets interrupt flag.
2. NVIC checks flag is enabled and has valid priority → asserts interrupt to CPU.
3. CPU finishes current instruction, saves context (stacking).
4. PC jumps to ISR address from vector table.
5. ISR executes → **must clear the flag** (else re-triggers immediately).
6. Context restored, execution resumes at interrupted point.

**Edge-triggered vs Level-triggered:**
| Type | Trigger Condition | Behavior | Example |
|---|---|---|---|
| Edge-triggered | Signal transition (rising/falling) | Fires once per edge, missed if too fast | Button press, external INT pin |
| Level-triggered | Signal held at a level (High/Low) | Keeps firing while level persists until cleared | Some UART RX, shared interrupt lines |

**Interrupt vs Polling:**
| Aspect | Interrupt | Polling |
|---|---|---|
| CPU usage | Free to do other work; event-driven | CPU continuously checks flag — wastes cycles |
| Response time | Fast, near-immediate | Depends on polling loop frequency |
| Complexity | More complex (ISR, priorities, race conditions) | Simple to implement |
| Best for | Rare, time-critical events | Simple systems, predictable timing, low event rate |
| Risk | Priority inversion, nested interrupt bugs | Missed events if polling too slow |

**Common MCQ traps:**
- Forgetting to clear interrupt flag inside ISR → infinite re-trigger.
- ISR should be **short and fast**; heavy work deferred to main loop (flag-based deferred processing).
- Nested interrupts: higher-priority interrupt can preempt a lower-priority ISR (Cortex-M NVIC supports this).
- Global interrupt enable/disable via `CPSIE`/`CPSID` or `__enable_irq()`/`__disable_irq()`.

---

## 4. Timers

**Concept:** Hardware counter clocked by a source (system clock/prescaled), used for delays, event counting, input capture, output compare, and PWM generation.

**Key registers/ideas:**
| Register | Purpose |
|---|---|
| CNT (Counter) | Current count value, increments/decrements each tick |
| PSC (Prescaler) | Divides input clock to slow down counting rate |
| ARR (Auto-Reload Register) | Value at which counter resets (defines period) |
| CCR (Capture/Compare Reg) | Used for PWM duty cycle / input capture timestamp |
| UIF (Update Interrupt Flag) | Set on overflow/reload — triggers timer interrupt |

**Timer frequency formula (very commonly asked):**
```
Timer tick frequency = Clock_freq / (PSC + 1)
Overflow (update) frequency = Clock_freq / [(PSC + 1) x (ARR + 1)]
Time period = (PSC + 1) x (ARR + 1) / Clock_freq
```

**Modes:**
- **Up-counting** — 0 → ARR → overflow → reset to 0.
- **Down-counting** — ARR → 0 → underflow → reset to ARR.
- **Input Capture** — records CNT value on external edge (measure pulse width/frequency).
- **Output Compare** — toggles/sets pin when CNT matches CCR (used for precise timing signals).
- **One-shot vs Continuous (auto-reload):** one-shot stops after one period; continuous free-runs.

**Workflow (basic delay using timer):**
1. Configure PSC and ARR for desired overflow period.
2. Enable timer (CEN bit).
3. Wait for UIF flag (poll) or enable timer interrupt.
4. On overflow, flag set → clear flag → action taken.

**Timer vs PWM:**
| Aspect | Timer (generic) | PWM (timer mode) |
|---|---|---|
| Purpose | Time delays, counting, scheduling | Generate variable duty-cycle square wave |
| Output | Usually just internal flag/interrupt | Physical output pin toggling |
| Key register | ARR defines period | ARR = period, CCR = duty cycle |

---

## 5. UART (Universal Asynchronous Receiver/Transmitter)

**Concept:** Asynchronous serial protocol — no shared clock; both sides agree on baud rate beforehand. Point-to-point (1 TX, 1 RX line).

**Key ideas:**
| Term | Meaning |
|---|---|
| Baud rate | Bits per second (e.g., 9600, 115200) — must match on both ends |
| Frame format | Start bit(1) + Data bits(5-9, usually 8) + Parity(optional) + Stop bit(1-2) |
| Idle state | Line held HIGH when idle |
| Start bit | Line goes LOW to signal beginning of frame |
| Stop bit | Line returns HIGH to mark end |
| Parity | Optional error-check bit (even/odd) |
| Registers | UDR/DR (Data Register), USR/SR (Status — TXE, RXNE flags), UBRR (Baud rate register) |

**Frame sequence (waveform, LSB first typically):**
```
Idle(High) → Start(Low) → D0 D1 D2 D3 D4 D5 D6 D7 → [Parity] → Stop(High) → Idle
```

**Workflow:**
1. Configure baud rate register (based on clock and desired baud).
2. Set frame format (data bits, parity, stop bits).
3. TX: Write data to DR → wait for TXE (transmit empty) flag → repeat.
4. RX: Wait for RXNE (receive not empty) flag → read DR.

**Common MCQ points:**
- No clock line → both devices must use **same baud rate** or data misreads.
- Baud rate mismatch → garbled/incorrect characters received.
- Full-duplex (TX and RX simultaneously possible, separate lines).
- Common baud error tolerance: ~2-3% max mismatch tolerated.

---

## 6. SPI (Serial Peripheral Interface)

**Concept:** Synchronous, full-duplex, master-slave protocol. Fast, used for sensors, SD cards, displays.

**Signal lines:**
| Signal | Function |
|---|---|
| SCLK | Clock generated by Master |
| MOSI | Master Out Slave In |
| MISO | Master In Slave Out |
| SS/CS (Slave Select/Chip Select) | Active-low line to select a slave; one per slave device |

**Key ideas:**
- Master generates clock — no baud rate matching needed (synchronous).
- Data shifted out on one clock edge, sampled on the other — defined by **CPOL** (Clock Polarity) and **CPHA** (Clock Phase), giving 4 SPI modes (0-3).
- Multiple slaves: separate CS lines (common OA question: "how many CS lines for N slaves in standard SPI?" → N lines).

**Sequence (one byte transfer):**
```
Master pulls CS Low → Master toggles SCLK 8 times →
  on each clock edge: MOSI shifts out bit, MISO shifts in bit simultaneously →
Master pulls CS High (transfer done)
```

**Workflow:**
1. Configure SPI mode (CPOL/CPHA), clock speed (prescaler), master/slave role.
2. Pull target's CS line LOW.
3. Write byte to data register → clock automatically shifts data both ways.
4. Read received byte from data register.
5. Pull CS HIGH.

**Common MCQ points:**
- SPI is full-duplex (send & receive at same time); I²C and UART are typically not both full duplex in the same simple sense (UART is full-duplex too, but I²C is half-duplex).
- No addressing scheme — device selected physically via CS line.
- Much faster than I²C/UART (can reach tens of MHz).

---

## 7. I²C (Inter-Integrated Circuit)

**Concept:** Synchronous, half-duplex, multi-master/multi-slave protocol using only 2 wires.

**Signal lines:**
| Signal | Function |
|---|---|
| SCL | Serial Clock (driven by master) |
| SDA | Serial Data (bidirectional) |

Both lines are **open-drain** — require external pull-up resistors.

**Key ideas:**
- Each slave has a unique 7-bit (or 10-bit) address.
- START condition: SDA goes LOW while SCL is HIGH.
- STOP condition: SDA goes HIGH while SCL is HIGH.
- ACK/NACK: receiver pulls SDA LOW (ACK) or leaves HIGH (NACK) after each byte.

**Sequence (typical write transaction):**
```
START → 7-bit Slave Address + R/W bit → ACK →
Register Address byte → ACK →
Data byte(s) → ACK (per byte) →
STOP
```

**Workflow:**
1. Master generates START condition.
2. Master sends slave address + R/W bit.
3. Addressed slave sends ACK.
4. Data transferred byte-by-byte, each followed by ACK/NACK.
5. Master generates STOP condition to release the bus.

**Common MCQ points:**
- Only 2 wires regardless of number of devices (vs SPI needing extra CS per slave).
- Open-drain + pull-up resistors are mandatory — line only pulled LOW actively, HIGH via resistor.
- Slower than SPI (Standard 100kHz, Fast mode 400kHz, Fast mode+ 1MHz, High-speed 3.4MHz).
- Supports multi-master (with arbitration) — SPI does not natively.

**UART vs SPI vs I²C — master comparison table:**
| Feature | UART | SPI | I²C |
|---|---|---|---|
| Wires | 2 (TX, RX) | 4+ (SCLK, MOSI, MISO, CS×N) | 2 (SCL, SDA) |
| Clock | None (async) | Yes (synchronous) | Yes (synchronous) |
| Duplex | Full | Full | Half |
| Speed | Low-Medium | Highest | Medium |
| Devices | Point-to-point (2 devices) | 1 master, multiple slaves (CS per slave) | Multiple masters/slaves (addressed) |
| Addressing | None | None (CS line) | Yes (7/10-bit address) |
| Complexity | Simple | Simple, more pins | Moderate (ACK/START/STOP logic) |

---

## 8. ADC (Analog-to-Digital Converter)

**Concept:** Converts a continuous analog voltage into a discrete digital value for processing by the MCU.

**Key ideas:**
| Term | Meaning |
|---|---|
| Resolution | Number of bits (e.g., 10-bit → 1024 levels, 12-bit → 4096 levels) |
| Reference voltage (Vref) | Defines the full-scale range mapped to max digital value |
| Sampling rate | How many samples/sec the ADC can take |
| Quantization error | Rounding error inherent in discretizing analog signal |
| Resolution formula | `Digital value = (Vin / Vref) x (2^n - 1)`, n = bits |

**Workflow:**
1. Select ADC channel (multiplexed input pin).
2. Configure resolution, sampling time, reference voltage.
3. Trigger conversion (software or timer-triggered).
4. Wait for End Of Conversion (EOC) flag.
5. Read digital result from data register.

**Common MCQ points:**
- Higher resolution = finer precision but larger digital range/more processing.
- Sampling below Nyquist rate (2x max signal frequency) causes **aliasing**.
- Types: SAR (Successive Approximation — most common in MCUs), Flash (fastest, expensive), Sigma-Delta (high precision, slow — used in audio/precision measurement).

**ADC vs Digital Input:**
| Aspect | ADC | Digital Input (GPIO) |
|---|---|---|
| Signal type | Continuous analog (any voltage in range) | Binary (HIGH/LOW only) |
| Use case | Sensors: temperature, light, potentiometer | Buttons, switches, digital sensor outputs |
| Data | Multi-bit numeric value | Single bit (0/1) |
| Conversion time | Takes finite conversion cycles | Instant read |

---

## 9. PWM (Pulse Width Modulation)

**Concept:** Digital technique to represent an analog-like average voltage/power by rapidly switching a signal ON/OFF, varying the ON time (duty cycle) within a fixed period.

**Key ideas:**
| Term | Meaning |
|---|---|
| Period (T) | Total time of one ON+OFF cycle = 1/frequency |
| Duty Cycle | % of period the signal stays HIGH = (Ton / T) x 100% |
| Frequency | How many cycles per second |
| Generated via | Timer's ARR (period) and CCR (compare value = ON time) |

**Waveform description:**
```
|----- Ton (High) -----|----- Toff (Low) -----|   <- one period T
Duty Cycle = Ton / (Ton + Toff) x 100%
```
Example: 50% duty cycle → equal HIGH and LOW time. 25% duty → shorter HIGH pulse, mostly LOW.

**Workflow:**
1. Configure timer with desired period (via PSC + ARR) → sets PWM frequency.
2. Set CCR (compare register) value → sets duty cycle (Ton portion).
3. Timer counts up; output pin HIGH while CNT < CCR, LOW when CNT ≥ CCR (or per configured PWM mode).
4. Repeats every period continuously (auto-reload).

**Common applications:** LED brightness control, DC motor speed control, servo motor angle control.

**Common MCQ points:**
- PWM does NOT produce true analog voltage — average voltage effect achieved via fast switching (needs filtering like RC/LPF for true analog output).
- Higher duty cycle → higher average power/brightness/speed.
- Servo motors use PWM pulse **width** (e.g., 1-2ms pulse in a 20ms period) to set angle, not just duty % — a nuanced but often tested distinction.

---

## Quick Cross-Topic Comparison Recap

| Comparison | Key takeaway |
|---|---|
| GPIO Input vs Output | Input reads external state (needs pull-up/down); Output drives a pin (push-pull vs open-drain) |
| Interrupt vs Polling | Interrupt = event-driven, efficient; Polling = simple, wastes CPU cycles |
| Edge vs Level triggered | Edge = fires on transition (can miss fast pulses); Level = fires while condition holds |
| UART vs SPI vs I²C | UART = async/2-wire/no addressing; SPI = sync/fast/CS-based; I²C = sync/2-wire/addressed/slower |
| Timer vs PWM | Timer = generic counting/timing; PWM = timer output shaped for variable duty cycle |
| ADC vs Digital Input | ADC = multi-bit analog value; Digital input = simple binary state |

---
*End of revision sheet — built for quick pre-OA recall, not exhaustive datasheet-level detail.*
