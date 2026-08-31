# canable-to-orionbms

Make a cheap [CANable](https://canable.io/)-family USB-CAN adapter running
[`canable-fw`](https://github.com/normaldotcom/canable-fw) work as a
drop-in replacement for the commercial **CANdapter** with **Ewert Energy
Systems' Orion BMS Utility** (`BMSApp`) — no PC-side workarounds, no proxy
software, just a one-line firmware change.

Confirmed working end-to-end against a real Orion BMS 2 (firmware
`3.2.3-40`) and the real `BMSApp.jar`/`BMSApp.exe` utility: after flashing
this patch, the board is auto-detected in BMSApp's Connect dialog as
`(CANdapter)` and the full connect / identify / live-data session works,
with **zero other firmware changes** — everything else `canable-fw` already
does (bitrate setup, CAN frame transmit/receive) was already compatible.

## The problem

BMSApp's "Connect to BMS" dialog scans serial ports for a CANdapter by
sending the slcan `V` (version query) command and checking the reply
format. The real CANdapter replies `Vaabbcc\r` (a 6-digit version string).

Stock `canable-fw` replies to `V` with a git-describe string instead, e.g.:
```
9fddea4 github.com/normaldotcom/canable-fw.git
```
That doesn't match what BMSApp's scanner expects, so a stock CANable board
never shows up in the dropdown — BMSApp reports "None Found" even though
the board's CAN transport is otherwise already fully protocol-compatible.

## The fix

One-line change to the `V` command handler in `src/slcan.c`: reply with a
fixed CANdapter-compatible version string (`V010203\r`) instead of the
git-describe string. See [`patches/0001-slcan-V-command-candapter-compatible.patch`](patches/0001-slcan-V-command-candapter-compatible.patch)
for the exact diff, applicable against
[`normaldotcom/canable-fw`](https://github.com/normaldotcom/canable-fw)
commit `9fddea4`.

Nothing else needs to change. `canable-fw`'s existing handling of the
CANdapter/Lawicel-style command set (`S` set-bitrate, `O`/`C` open/close,
`t`/`T` frame transmit, incoming frame relay) already matches what BMSApp
expects.

## How to use it

### Option A: flash the prebuilt binary (fastest)

1. Download [`firmware/canable-orionbms-candapter.bin`](firmware/canable-orionbms-candapter.bin)
   from this repo.
2. Put your board into its DFU bootloader: set the BOOT jumper (or hold the
   boot button on a CANable Pro) and plug it in via USB.
3. Flash it with [`dfu-util`](http://dfu-util.sourceforge.net/):
   ```
   dfu-util -d 0483:df11 -c 1 -i 0 -a 0 -s 0x08000000:leave -D canable-orionbms-candapter.bin
   ```
4. Move the BOOT jumper back (or release the boot button) and unplug/replug
   the board. It should re-enumerate as a normal CDC-ACM serial port.

### Option B: build it yourself

```
git clone --recurse-submodules https://github.com/normaldotcom/canable-fw.git
cd canable-fw
git checkout 9fddea4
git apply /path/to/patches/0001-slcan-V-command-candapter-compatible.patch
make
```
Requires an `arm-none-eabi-gcc` toolchain (e.g. the
[GNU Arm Embedded Toolchain](https://developer.arm.com/downloads/-/gnu-rm))
and `make`. Flash `build/canable-*.bin` the same way as Option A.

## Verifying it worked

With the board plugged in normally (not in DFU mode), open a serial
terminal at any baud rate (USB-CDC ignores it) and send `V` followed by a
carriage return. You should get back:
```
V010203
```
Then open BMSApp's Connect dialog — the board's port should be listed with
`(CANdapter)` next to it, selectable and connectable like a real one.

## Protocol notes (for anyone extending this further)

Captured from a real BMSApp <-> real Orion BMS 2 session, in case it's
useful for other adapters or tooling:

- BMSApp's connect sequence: `C` (close) -> `S6` (500 kbit/s) -> `O` (open)
  -> a broadcast OBD2 identification probe.
- Identification probe: standard OBD2 **Mode $9 PID 0x0B** sent to CAN ID
  **0x7DF**, requesting `t7DF802090B0000000000`.
- The Orion BMS answers via a real **ISO-TP** (ISO 15765-2) multi-frame
  message on its physical response ID (`0x7EB` for the default `0x7E3`
  profile request ID), with a positive Mode 9 response:
  `49 0B 01` + ASCII `"ORIONBMS03"` + a trailing `0x00` byte (14 bytes
  total) — note the identification string is `"ORIONBMS03"`, not just
  `"ORIONBMS"` as Orion's own docs' prose implies.
- BMSApp sends a genuine ISO-TP Flow Control frame (`30 07 00 ...`, Block
  Size 7) on `response_id - 8` and expects it to be respected before the
  Consecutive Frame — sending the CF proactively without waiting for the
  real FC gets it silently dropped.
- Live data after identification uses Mode `$22` (UDS ReadDataByIdentifier)
  — see Orion's own
  [OBD2 PID list](https://www.orionbms.com/downloads/misc/orionbms_obd2_pids.pdf)
  for that table.

## Credits / license

This repo is just a small patch on top of
[`normaldotcom/canable-fw`](https://github.com/normaldotcom/canable-fw),
which is MIT-licensed (see upstream `LICENSE.md`). All credit for the CAN
transport and USB implementation belongs to that project's authors — this
patch changes eight lines in one function.

## Disclaimer

You're flashing third-party firmware onto hardware and connecting it to a
battery management system. Reflashing carries the usual small risk of
bricking your board (recoverable via DFU in almost all cases); miswiring a
BMS can damage it or be unsafe around a live battery pack. Verify your own
wiring against your BMS's official documentation before applying power.
Provided as-is, no warranty — see the LICENSE.
