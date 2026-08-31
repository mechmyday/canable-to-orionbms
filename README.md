# canable-to-orionbms

Make a cheap [CANable](https://canable.io/)-family USB-CAN adapter running
[`canable-fw`](https://github.com/normaldotcom/canable-fw) work as a
drop-in replacement for [the CANdapter](https://www.candapter.com/) with
Orion BMS Utility (`BMSApp`) — no PC-side workarounds, no proxy software,
just a one-line firmware change. See
[Orion BMS's documentation](https://www.orionbms.com/manuals/) for more on
that software.

Confirmed working end-to-end against a real Orion BMS 2 (firmware
`3.2.3-40`) and the real `BMSApp.jar`/`BMSApp.exe` utility: after flashing
this patch, the board is auto-detected in BMSApp's Connect dialog as
`(CANdapter)` and the full connect / identify / live-data session works,
with **zero other firmware changes** — everything else `canable-fw` already
does (bitrate setup, CAN frame transmit/receive) was already compatible.
The only issue was that `canable-fw`'s version-query reply doesn't match
the format BMSApp's port scanner looks for; see
[`patches/0001-slcan-V-command-candapter-compatible.patch`](patches/0001-slcan-V-command-candapter-compatible.patch)
for the exact one-function fix.

## Tools needed

- A CANable-family board running `canable-fw`. Tested with:
  - The official [canable.io](https://canable.io/) USB-to-CAN board.
  - The [Fysetc UCAN](https://wiki.fysetc.com/docs/UCAN). I used this particular board in my tests.

  Other STM32F042-based CANable-compatible boards running `canable-fw`
  should work the same way.
- [`dfu-util`](http://dfu-util.sourceforge.net/), for flashing over USB —
  see the Linux or Windows steps below for how to get it.

## How to use it

### Option A: flash the prebuilt binary (fastest)

1. Download [`firmware/canable-orionbms-candapter.bin`](firmware/canable-orionbms-candapter.bin)
   from this repo.
2. Put your board into its DFU bootloader: set the BOOT jumper (or hold the
   boot button on a CANable Pro) and plug it in via USB.
3. Flash it with `dfu-util`.

   **On Linux:**
   ```
   sudo apt install dfu-util
   dfu-util -d 0483:df11 -c 1 -i 0 -a 0 -s 0x08000000:leave -D canable-orionbms-candapter.bin
   ```
   Drop the `sudo` on the flash command itself if your distro's udev rules
   already grant your user access to the device; otherwise run it with
   `sudo` too.

   **On Windows:**
   1. While the board is in DFU mode, Windows will enumerate it as an
      unknown/"STM32 BOOTLOADER" device — `dfu-util` can't talk to it
      through the default driver. Install
      [Zadig](https://zadig.akeo.ie/), run it, open **Options > List All
      Devices**, select **STM32 BOOTLOADER** from the device dropdown, set
      the target driver to **WinUSB**, and click **Replace Driver**. (This
      only needs to be done once per computer.)
   2. Get `dfu-util` for Windows: grab a build from the
      [official releases page](http://dfu-util.sourceforge.net/releases/),
      or install it with a package manager if you have one, e.g.
      [Chocolatey](https://community.chocolatey.org/packages/dfu-util):
      ```
      choco install dfu-util
      ```
   3. Open Command Prompt or PowerShell in the folder containing
      `dfu-util.exe` and the downloaded `.bin` file, and run:
      ```
      dfu-util -d 0483:df11 -c 1 -i 0 -a 0 -s 0x08000000:leave -D canable-orionbms-candapter.bin
      ```
4. Move the BOOT jumper back (or release the boot button) and unplug/replug
   the board. It should re-enumerate as a normal CDC-ACM serial port (shows
   up as a new COM port on Windows, `/dev/ttyACM0`-style on Linux).

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
  `"ORIONBMS"` as the docs' prose implies.
- BMSApp sends a genuine ISO-TP Flow Control frame (`30 07 00 ...`, Block
  Size 7) on `response_id - 8` and expects it to be respected before the
  Consecutive Frame — sending the CF proactively without waiting for the
  real FC gets it silently dropped.
- Live data after identification uses Mode `$22` (UDS ReadDataByIdentifier)
  — see the
  [OBD2 PID list](https://www.orionbms.com/downloads/misc/orionbms_obd2_pids.pdf)
  for that table.

## Credits / license

This repo is just a small patch on top of
[`normaldotcom/canable-fw`](https://github.com/normaldotcom/canable-fw),
which is MIT-licensed (see upstream `LICENSE.md`). All credit for the CAN
transport and USB implementation belongs to that project's authors — this
patch changes eight lines in one function.

## Disclaimer
**THIS SOFTWARE WAS WRITTEN WITH ASSISTANCE FROM AN LLM** Here be dragons, proceed carefully.

**USE THIS REPOSITORY ENTIRELY AT YOUR OWN RISK.** Flashing third-party
firmware onto your adapter and connecting it to a battery management
system could be risky. **THE AUTHOR OF THIS REPOSITORY IS NOT
RESPONSIBLE FOR ANY DAMAGE TO YOUR ADAPTER, YOUR BMS, YOUR VEHICLE, OR
ANYTHING ELSE, INCLUDING A BRICKED ADAPTER OR A BRICKED OR DAMAGED BMS,
RESULTING FROM USE OF THIS PATCH, FIRMWARE, OR THE INSTRUCTIONS IN THIS
REPOSITORY.** Reflashing your adapter and/or using it in a manner not
sanctioned by its manufacturer, or connecting third-party hardware to your
BMS, **MAY VOID OR OTHERWISE VIOLATE THE WARRANTY ON YOUR BMS AND/OR YOUR
ADAPTER.** Check your own hardware's warranty terms before proceeding.

THIS SOFTWARE, FIRMWARE, AND DOCUMENTATION ARE PROVIDED "AS IS", WITHOUT
WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE
WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY CLAIM,
DAMAGES, OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT, OR
OTHERWISE, ARISING FROM, OUT OF, OR IN CONNECTION WITH THIS REPOSITORY OR
THE USE OR OTHER DEALINGS IN IT.

Verify your own wiring against your BMS's official documentation before
applying power to anything.
