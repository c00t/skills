---
name: jlink
description: >
  Drive a SEGGER J-Link (JLink.exe Commander) non-interactively from the command
  line to debug an ARM Cortex-M target. Use this to halt the core, dump CPU
  registers, read/write memory-mapped peripheral registers, analyze HardFaults
  (IPSR/HFSR/CFSR), dump flash to a file, erase, and program hex/bin images.
  Invoke whenever a task needs live on-chip register/memory access, fault
  diagnosis, or flashing over SWD/JTAG without opening the J-Link GUI.
---

# Using J-Link (JLink.exe) from the command line

SEGGER's `JLink.exe` (J-Link Commander) is scriptable: you feed it a **command
file** and it executes each line against the target, then exits. This is the
reliable, headless way to read/write registers, dump/flash memory, and diagnose
faults — no GUI, fully captured output.

## 1. Connection parameters

You need four things to connect. Discover them once, reuse them every call:

| Flag | Meaning | How to find it |
|---|---|---|
| `JLink.exe` path | The Commander executable | `(Get-Command JLink.exe).Source`, or search the toolchain/IDE install dir |
| `-device <NAME>` | Exact device name | SEGGER's supported-device list, or the MCU vendor's J-Link docs |
| `-if <SWD\|JTAG>` | Debug interface | Usually `SWD` on Cortex-M |
| `-speed <kHz>` | Interface clock | `4000` is a safe default for most parts |

## 2. The invocation pattern (always use a command file)

Do **not** try to pipe commands on stdin. Write a `.jlink` command file, then run:

```powershell
# PowerShell — use the & call operator if the path has spaces
& "<path>\JLink.exe" `
    -device <DEVICE> -if SWD -speed 4000 -autoconnect 1 `
    -CommandFile "<path>\commands.jlink" 2>&1
```

```bash
# Bash (Git Bash) equivalent
"<path>/JLink.exe" \
    -device <DEVICE> -if SWD -speed 4000 -autoconnect 1 \
    -CommandFile "<path>/commands.jlink" 2>&1
```

`-autoconnect 1` connects without prompting. `2>&1` captures all output.

### Filter the noise

J-Link prints a banner. Keep only the lines you care about with `Select-String`:

```powershell
... -CommandFile "cmds.jlink" 2>&1 | Select-String -Pattern `
  "^[0-9A-F]{8} =|^R\d|^PC |^XPSR|^MSP|^PSP|^LR |^SP |Could not|IPSR"
```

- `^[0-9A-F]{8} =` matches memory-read output (`<ADDR> = <VALUE>`).
- `^R\d|^PC |^XPSR|...` matches `regs` output.
- `Could not` catches `Could not read memory.` (see gotcha #3 — it's diagnostic).

## 3. Command-file skeleton

Every read/write session follows this shape:

```
h                        <- HALT first; reads are unreliable on a running core
regs                     <- (optional) dump core registers
mem32 <addr> 1           <- read one 32-bit word
w4    <addr> <data>      <- write one 32-bit word
Sleep 150                <- wait 150 ms (e.g. let a peripheral react)
mem32 <addr> 1           <- read again to see the effect
go                       <- RESUME the core (leave it running)
exit                     <- quit JLink.exe
```

**Always end with `go` then `exit`** so you leave the target running, not halted.

## 4. Command reference (the ones you actually need)

| Command | Meaning |
|---|---|
| `h` / `halt` | Halt the CPU. **Required before reliable memory/register access.** |
| `go` / `g` | Resume execution. |
| `r` | Reset (and halt) the target. `RSetType <n>` picks the reset kind. |
| `regs` | Dump R0–R15, PC, SP, LR, XPSR (includes IPSR — the exception number). |
| `mem32 <addr> <n>` | Read `n` 32-bit words. **Use `n=1`** (block reads can fail — gotcha #2). |
| `mem16 <addr> <n>` / `mem8 <addr> <n>` | Read halfwords / bytes. |
| `w4 <addr> <data>` | Write a 32-bit word. |
| `w2 <addr> <data>` / `w1 <addr> <data>` | Write halfword / byte. |
| `Sleep <ms>` | Delay. Useful between a write and the read that observes its effect. |
| `savebin <file> <addr> <numBytes>` | Dump memory/flash to a binary file. |
| `loadfile <file> [addr]` | Program a `.hex`/`.mot`/`.srec`/`.bin` image (erases as needed). |
| `loadbin <file> <addr>` | Program a raw binary at `addr`. |
| `erase` | Erase the device's flash (whole chip or configured range). |
| `verifybin <file> <addr>` | Verify flash against a binary. |
| `q` / `exit` / `qc` | Quit. Prefer an explicit `go` + `exit`. |

Output format for reads is `ADDRESS = VALUE` (value is hex, no `0x`). A failed
read prints `Could not read memory.`

## 5. Reading registers & diagnosing a HardFault

`regs` gives PC and `XPSR`. The **IPSR** field (low byte of XPSR) is the active
exception number: `000`=thread (no exception), `003`=HardFault, `004`=MemManage,
`005`=BusFault, `006`=UsageFault. J-Link annotates it, e.g.
`XPSR = ...0003: ... IPSR = 003 (HardFault)`.

When `IPSR` shows a fault, read the Cortex-M System Control Block fault registers.
**These addresses are architectural — identical on every Cortex-M:**

| Register | Address | What it tells you |
|---|---|---|
| ICSR | `0xE000ED04` | `VECTACTIVE` = current exception; `VECTPENDING`. |
| VTOR | `0xE000ED08` | Vector table base — tells you which image's vectors are live. |
| SHCSR | `0xE000ED24` | Which fault handlers are enabled / active. |
| CFSR | `0xE000ED28` | Configurable Fault Status (MemManage/Bus/Usage sub-faults). |
| HFSR | `0xE000ED2C` | HardFault Status. `FORCED`(bit30) = escalated config fault; `VECTTBL`(bit1) = bad vector fetch. |
| MMFAR | `0xE000ED34` | Faulting data address for MemManage (valid if `CFSR.MMARVALID`). |
| BFAR | `0xE000ED38` | Faulting data address for BusFault (valid if `CFSR.BFARVALID`). |

To recover the **faulting instruction address**, read the stacked exception
frame. The handler's `LR` (EXC_RETURN) bit2 selects the stack: 0→MSP, 1→PSP.
Read that SP; the 8-word frame is `R0,R1,R2,R3,R12,LR,PC,xPSR`, so the original
**PC is at `SP+0x18`** and **xPSR at `SP+0x1C`**:

```
h
regs                 <- note SP (and LR to know MSP vs PSP)
mem32 <SP> 8         <- read the 8-word stack frame (PC = word[6], at SP+0x18)
go
exit
```

## 6. Writing memory (peripheral pokes)

`w4` writes a word; combine with `Sleep` + re-read to observe a side effect.
Example shape — write a control register, wait, then read a status register:

```
h
mem32 <status_reg> 1     <- baseline
w4    <control_reg> <val><- poke the peripheral
Sleep 150
mem32 <status_reg> 1     <- observe the effect
go
exit
```

## 7. Dump / erase / flash

```
# Dump flash to a binary (forensic capture before touching anything)
h
savebin "<path>\dump.bin" <start_addr> <num_bytes>
go
exit
```

```
# Erase then program an image, then verify
r
erase
loadfile "<path>\image.hex"
verifybin "<path>\image.bin" <load_addr>
r
go
exit
```

`loadfile` understands Intel-HEX/Motorola-S/bin by extension and erases the
sectors it writes. For a clean reflash use `erase` first (whole-chip) or let
`loadfile` sector-erase.

## 8. Gotchas (learned the hard way)

1. **Halt before reading.** Reads on a free-running core are unreliable; always
   start the command file with `h`, finish with `go`.
2. **Read one word at a time.** `mem32 <addr> 1` is reliable; multi-word block
   reads (`mem32 <addr> 16`) sometimes error out — loop single reads instead.
3. **`Could not read memory.` is a clue, not just an error.** A clock-gated or
   unpowered peripheral is unreadable. If a peripheral read fine before and
   suddenly doesn't, suspect its clock got gated off (check the clock/enable
   register). OTP/NVR/option banks may need an indirect access sequence rather
   than a direct `mem32`.
4. **Output pins can read 0 on the input-data register.** On parts that disable
   an output pin's input buffer, the input-data register reads 0 regardless of
   the real level. Read the **output latch (data-output) register** to see what
   you're actually driving.
5. **Know which peripherals run while halted.** Some peripherals keep running
   while the CPU is halted (their live state is valid during a halt); others
   freeze. Check the peripheral's freeze/debug-mode bit before trusting a read.
6. **PowerShell needs the `&` call operator** for a quoted exe path, and use
   `2>&1` to capture J-Link's stderr/banner.

## 9. Safety — irreversible operations

- **Never blind-write NVR/OTP/option bytes.** One-time-programmable fields don't
  come back, and a SWD/JTAG-disable bit (in customer NVR / option bytes) will
  **permanently kill the debug port** — an unrecoverable brick. Read-only
  baselines are fine; writes need an explicit, deliberate reason.
- **`erase` / `loadfile` destroy current contents.** `savebin` a full dump first
  if the on-chip image is forensic evidence you might need again.
- Leave the target **running** (`go`) unless the user wants it halted.
