# LPC1788 NuttX Touchscreen Project — Continuation Prompt

Continue the existing LPC1788 touchscreen programmer/diagnostics project in this workspace. Make changes only when requested, preserve all existing local work, and verify any firmware or software changes before claiming success.

## Project and hardware

- Target: Embedded Artists LPC1788 OEM board with the EA OEM Base Board Rev C.
- RTOS: NuttX. Active NuttX tree: `/home/bg9211014/lpc1788_workspace/nuttx`; apps tree: `/home/bg9211014/lpc1788_workspace/nuttx/apps`.
- Main application repository: `/home/bg9211014/lpc1788_workspace/ealpc1788`.
- EA sample source and legacy demo binaries are **reference material only**. Do not integrate or flash their standalone binaries.
- Preserve the user's local modifications. The NuttX worktree is dirty and substantially behind upstream; do not reset, stash, rebase, or discard unrelated work.
- The user wants to conserve the lower-rate flashing setup and credits. For LPC ISP use 57600 baud and 12000 kHz oscillator unless verified otherwise. No live credit meter is available; if the user or runtime indicates credits are nearly exhausted, use the remainder to produce a detailed continuation prompt.
- Keep ESP integration deferred; it was removed from the mock UI and is not part of the active build.

## Current software and touchscreen work

The active NuttX configuration builds successfully. Touchscreen/NXWM changes already present include:

- ADS7843E sampling/filtering over seven samples for X/Y/Z1/Z2.
- Corrected ADC data shift and TI-style pressure/resistance calculation with pressure-valid reporting.
- X-plate resistance set to 630 ohms.
- 30 ms touch polling and 25 px calibration margin.

Relevant files:

- `nuttx/drivers/input/ads7843e.c`
- `nuttx/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_touchscreen.c`
- `nuttx/boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_bringup.c`
- `nuttx/boards/arm/lpc17xx_40xx/open1788/src/open1788.h`
- `nuttx/boards/arm/lpc17xx_40xx/open1788/configs/ea_gui/defconfig`
- `nuttx/.config`

The app's ESP32/ESP8266 target, firmware placeholders, and `esptool` entries were removed from `src/App.tsx`. The UI build/lint was not run because Node/npm were unavailable.

## Board-level findings and constraints

- **CAN2 must remain disabled** while the LCD is active. The Rev C schematic marks CAN2 transceiver U14 “not mounted”; default CAN2 pins P0.4/P0.5 overlap LCD VD0/VD1, and alternate P2.7/P2.8 overlap LCD VD5/VD6.
- CAN1 through transceiver U15 and J16 pins 7/8 is a candidate only after physically confirming jumpers, termination, and bus requirements.
- UART0 is the FTDI console. UART1's Port 3 mapping conflicts with the 32-bit EMC/SDRAM bus and remains disabled.
- UART2 uses P0.10/P0.11; JP6 is required for the DB9 route. The DB9 is RS-232 voltage, not TTL.
- USB host initialization support and vendor sample references exist, but host/OTG jumper, port power, and wiring must be confirmed before enabling or testing USB host.
- EA joystick sample pins P2.22/P2.23/P2.25/P2.26/P2.27 do not match current NuttX definitions in `open1788.h`; resolve the pin mapping before enabling joystick support.

## Build and flash status — do not assume firmware is usable

- The NuttX build completed with `make -j4`.
- Reported ELF footprint: 295,408 / 524,288 bytes flash (56.34%), leaving 228,880 bytes (43.66%); SRAM 24,456 / 65,536 bytes (37.32%).
- Image: `/home/bg9211014/lpc1788_workspace/nuttx/nuttx.hex`.
- `lpc21isp -control -detectonly` previously identified the LPC1788, 512 KiB flash, at 57600 baud / 12 MHz oscillator.
- An upload without `-writedelay` failed while writing sector 12. A retry using `-writedelay -verify -try100` reached sector 17 but was terminated by a 300-second timeout (exit 124). There was no final checksum write or verification result.
- A later short detect-only attempt returned exit code 1 without confirming the bootloader. The subsequent attempt to start a long detect-only wait was denied/interrupted. Do not assume the MCU is in ISP mode, that a flasher is running, or that the image is complete.
- No full-chip erase was used. The existing firmware/flash state is uncertain.

### Safe flashing sequence

1. Confirm the serial device and ensure no console or other process owns it. Do not reset the MCU or start another flash until the user has explicitly confirmed the board is ready in ISP mode.
2. Detect the LPC1788 in ISP mode at 57600 baud / 12000 kHz before writing.
3. If detection succeeds and the user confirms the board is ready, flash the complete `nuttx.hex` with verification and a timeout long enough to finish the delayed upload. Do not use a whole-chip wipe.
4. Require the flasher's explicit completion and verification result. A timeout, partial sector progress, or lack of a final verify message is failure/uncertain state.
5. Only after a verified flash, reset into NuttX, reconnect the console at 115200 baud, and test NSH boot, LCD/backlight, NXWM/calibration, touch at center and edges, pressure readings, and SD/FAT operations.

## Next work after verified boot

1. Measure and correct touchscreen alignment based on center and edge testing; do not assume the 25 px calibration margin is correct for the hardware.
2. Resolve the EA/NuttX joystick pin mismatch from the actual board schematic and wiring before enabling it.
3. Confirm physical USB host routing and power before enabling USB MSC testing.
4. Leave CAN2 disabled; evaluate CAN1 only after confirming board jumpers and bus requirements.
5. Add application capabilities incrementally (for example SD firmware browsing, HEX inspection, serial/YMODEM transfer, and hardware diagnostics). Rebuild and remeasure flash after each change.
6. Keep ESP deferred until target hardware and wiring are specified.

## Works cited / local references

- Embedded Artists, **OEM Base Board Rev C schematic**: `ealpc1788/public/assets/aistudio/OEM_Base_Board_rev_C_954e1d3b.pdf`.
- Embedded Artists, **EA LPC1788 Linux BSP Guide** (reference only; Linux support does not prove NuttX support): `ealpc1788/public/assets/aistudio/linux-EA-LPC1788-bspg-1.12.0_4d1d3944.pdf`.
- Embedded Artists, **UART2 RS-232 sample README**: `/home/bg9211014/ss4/eanatty/lpc1788_samples/s_ob_uart_rs232/readme.txt`.
- Embedded Artists, **USB host sample README**: `/home/bg9211014/ss4/eanatty/lpc1788_samples/s_usbhostlite/ReadMe.txt`.
- Embedded Artists, **joystick sample source**: `/home/bg9211014/ss4/eanatty/lpc1788_samples/s_ob_joystick/src/main.c`.
- NuttX board and driver implementation: the source files listed above. Consult the current checked-out configuration and source before making new claims about enabled functionality.

Use the vendor samples and BSP strictly as hardware/behavior references. Verify claims against the Rev C schematic and current NuttX sources; do not infer that a vendor Linux driver or demo is directly usable in NuttX.
