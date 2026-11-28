# LPC1788 NuttX Touchscreen & NXWM Project — Master Continuation Prompt

Continue the existing LPC1788 touchscreen/NXWM project in this workspace. Make changes only when requested, preserve all existing local work, and verify any firmware or software changes before claiming success.

## Project and hardware

- Target: Embedded Artists LPC1788 OEM board (Rev D/E) with the EA OEM Base Board Rev C and 4.3" LCD Board Rev A (`480x272`, resistive ADS7843E/TSC2046 touchscreen).
- RTOS: NuttX (`NuttX-13.1.1-RC0`, `open1788:ea_gui`, `CONFIG_BUILD_FLAT=y`).
- Active NuttX tree: `/home/bg9211014/lpc1788_workspace/nuttx`; apps tree: `/home/bg9211014/lpc1788_workspace/apps` (`CONFIG_APPS_DIR="../apps"`).
- Main application repository: `/home/bg9211014/lpc1788_workspace/ealpc1788`.
- EA sample source and legacy demo binaries are **reference material only**. Do not integrate or flash their standalone binaries.
- Preserve the user's local modifications. Both `nuttx` and `apps` worktrees contain extensive local changes; **never** run `git reset`, `git stash`, `git rebase`, or discard unrelated work.
- For LPC ISP flashing, **always use 57600 baud and 12000 kHz oscillator** (`lpc21isp -control -try100 -verify -bin ... /dev/ttyUSB0 57600 12000`) per LPC177x/8x erratum ISP.1, and **never** use whole-chip erase (`-wipe`).
- Toolchain note: host provides `arm-none-eabi-*`, while NuttX config expects `arm-nuttx-elf-*`. Build with `/tmp/lpc1788-toolshim` symlinks (`arm-nuttx-elf-* -> arm-none-eabi-*`) prepended to `PATH`.
- Keep ESP integration deferred; it is not part of the active build.

## Current working build state (2026-11-27)

- **Verified working on hardware:** Boot to NSH (`115200` baud on `/dev/ttyUSB0`), `nxwm`, 4-point touchscreen calibration, and **NxWM Home Screen application launching** now work cleanly.
- **Verified binary metrics:**
  - `arm-none-eabi-size nuttx`: `text=468192`, `data=3220`, `bss=31904`, `dec=503316` (`0x7ae14`)
  - `nuttx.bin`: `471,412` / `524,288` bytes (`89.91%` flash, `52,876` bytes free), SHA-256 `74294c8de8e6921a7153f9a96059d0a5a90d2ad13c9391676c8843d1c7633820`
  - `nuttx.hex`: `1,326,022` bytes
- Active board/config: `open1788:ea_gui` in `nuttx/.config` (`CONFIG_BUILD_FLAT=y`, `CONFIG_MM_REGIONS=3`, `CONFIG_NX=y`, `CONFIG_NX_BLOCKING=y`, `# CONFIG_NX_RAMBACKED is not set`, `CONFIG_NXSTART_SERVERPRIO=125`, `CONFIG_SCHED_LPWORK=y` prio `110` / stack `4096`, `CONFIG_DEBUG_GRAPHICS=y`, `CONFIG_DEBUG_GRAPHICS_ERROR=y`).
- Pre-change backup snapshots exist under `nuttx/build_backups/` (including `stage_20261127_143254/`, `clean_*/`, and `worker_fix_*/`).

## How we got here & root causes fixed (do not regress!)

### Timeline of the Home Screen bug
1. After getting 4-point calibration and LCD output working, tapping any Home Screen icon (`CHexCalculator`, `CNxTerm`, toolbar buttons) produced no application launch even though `nx_mousein()` returned `0` and `NX callback` delivered press/move/release coordinates inside the icon bounds.
2. Staged `NXDBG` instrumentation proved blank-area taps completed `pollEvents()` normally, whereas any tap hitting a `CImage` widget stalled inside `CImage::onClick()` -> `CNxWidget::redraw()` -> `CImage::drawContents()` -> `CGraphicsPort::drawBitmap()` -> `nxtk_bitmapwindow()` -> `nx_bitmap()`, leaving `lpwork` blocked `Waiting Semaphore` while `NX_Server` sat idle `Waiting MQ empty`.
3. A clean-up intermediate build disabled `CONFIG_NX_BLOCKING` and stripped diagnostics without fixing the worker task-group bug; that caused `CNxServer::listener` (`prio 100`) to busy-spin `Ready` and left the Home Screen still unresponsive.

### Root Cause #1 — Cross-task-group `mqd_t` mismatch (`CWindowMessenger` on `lpwork` `PID 0` vs `nxwm` `PID 7`)
- `nuttx/libs/libnx/` compiles with `-D__KERNEL__`, so `_MQ_OPEN` maps to `nxmq_open` and `_MQ_SEND` maps to `nxmq_send`.
- `nxmq_send()` (`nuttx/sched/mqueue/mq_send.c:690`) resolves `mqdes` via `file_get(mqdes, &filep)` against **`this_task()->group->tg_filelist`**.
- `conn->cwrmq` (the client-to-server mqueue fd) is opened by **`nxwm` (`PID 7`)** in `nx_connect.c:149`.
- Stock `CWindowMessenger` queued `inputWorkCallback()` and `destroyWorkCallback()` onto `USRWORK` (`lpwork`), which in `CONFIG_BUILD_FLAT=y` is a kernel thread in **`PID 0`**, NOT `PID 7`.
- Every `nx_*` / `nxtk_*` call executed from `lpwork` (`nx_bitmap()`, `nxtk_bitmapwindow()`, `nxtk_openwindow()`, etc.) looked up `nxwm`'s `conn->cwrmq` fd inside `PID 0`'s `tg_filelist` instead of `PID 7`'s (failing with `-EBADF` or aliasing `PID 0` descriptors; note `nxtk_bitmapwindow.c` also ignores `nx_bitmap()`'s return value).
- **Fix applied:** Replaced `work_queue(USRWORK, ...)` in `apps/include/graphics/nxwm/cwindowmessenger.hxx` and `apps/graphics/nxwm/src/cwindowmessenger.cxx` with a single detached worker pthread (`CWindowMessenger`, priority `110`, stack `4096`, `PTHREAD_EXPLICIT_SCHED`) created inside **`nxwm`'s own task group (`PID 7`)**, fed by a mutex-protected FIFO (`g_queueLock` + `g_queueSem`) with destructor cleanup invalidating any queued items matching `this`.

### Root Cause #2 — `CONFIG_NX_BLOCKING` must stay `y`
- With `CONFIG_NX_BLOCKING` disabled, `nx_connect.c:132` opens `conn->crdmq` with `O_NONBLOCK`, and `nx_eventhandler()` returns `OK` on `-EAGAIN`. `CNxServer::listener` (`cnxserver.cxx`) loops tightly calling `nx_eventhandler()`, spinning `Ready` at priority `100` and starving lower-priority threads.
- **Fix applied:** Restored `CONFIG_NX_BLOCKING=y` in `nuttx/.config` (keep `CONFIG_NX_BLOCKING=y` enabled whenever using `CNxServer::listener`).

### Preserved functional touchscreen & NXWM fixes in active build
- **Touchscreen hardware/driver (`nuttx`):** `drivers/input/ads7843e.c`, `drivers/input/ads7843e.h`, `include/nuttx/input/ads7843e.h`, `boards/arm/lpc17xx_40xx/open1788/src/lpc17_40_touchscreen.c`, `open1788.h` — 7-sample filtering (X/Y/Z1/Z2), corrected ADC shift, TI-style pressure calculation (`pressure-valid`), 630-ohm X-plate, SSP0 mode 0 at 100 kHz, 30 ms polling, threshold 3, `CONFIG_NXWM_CALIBRATION_MARGIN=25`, `CONFIG_NXWM_CALIBRATION_ANISOTROPIC=y`.
- **NX `nx_bitmap.c`:** `nuttx/libs/libnx/nxmu/nx_bitmap.c` retains a 1-second `nxsem_timedwait` guard on `sem_done` so a lost server reply cannot lock the worker forever.
- **`apps/graphics/nxwm/src/ctouchscreen.cxx` & `include/graphics/nxwm/ctouchscreen.hxx`:** Clamps `ANISOTROPIC` mapped coordinates to `[0 .. m_windowSize - 1]`, falls back to `m_lastX/Y` on `TOUCH_UP` samples lacking coordinates, and adds `wakeListener()`.
- **`apps/graphics/nxwidgets/src/cwidgetcontrol.cxx` & `include/graphics/nxwidgets/cwidgetcontrol.hxx`:** Preserves pending `leftPressed` / `leftReleased` flags with separate `pressX/Y` and `releaseX/Y` coordinates until `pollMouseEvents()` consumes them, executing press -> drag -> release in order.
- **`apps/graphics/nxwm/src/cstartwindow.cxx`:** Fixes `maxSize.w` and `windowSize.h` typos, enforces `STARTWINDOW_MIN_SPACING=12`, retains `StartWindow:` action/create/start console logs.
- **`apps/graphics/nxwm/src/capplicationwindow.cxx`:** Uses `40` px wide Minimize/Stop hit areas (`buttonGap=4`, `buttonRightInset=12`), centered borderless icons.
- **`apps/graphics/nxwm/src/nxwm_main.cxx`:** Duplicate-launch guard `g_nxwmStarted`, starts `CStartWindow` non-minimized, calls `stopApplication(calibrationApp)` after calibration finishes, and sleeps persistently in `main()`.

## Observed behavior notes & active next tasks

- **Green Start arrow in top-left Taskbar:** That button is the Start Window's taskbar switcher icon. Tapping it switches back to the Start Window from another running app; when the Start Window is already on top, it intentionally does nothing.
- **Pending UI updates requested by user:**
  1. Remove the Calibration (scales) shortcut icon from the Start Window (keep startup 4-point calibration, because launching calibration a second time from the Start Window freezes the UI).
  2. Add shortcuts for the remaining available applications on the Start Window.
- **Remaining hardware validation after UI updates:**
  - Verify SD/FAT operations, Ethernet, CAN1 (only after confirming jumpers/termination), UART2 (with JP6, RS-232 levels), joystick pin mapping (`open1788.h` vs EA sample P2.22/23/25/26/27), and USB host wiring/power.

## Board-level constraints (Rev C schematic verified)

- **CAN2 must remain disabled** while the LCD is active (`U14` "not mounted"; default `P0.4/P0.5` overlap LCD `VD0/VD1`, alternate `P2.7/P2.8` overlap `VD5/VD6`).
- **USB1 OTG conflicts with LCD** (`P1.27–P1.29`); do not enable USB1 OTG while LCD is active.
- **UART0** is the FTDI console (`115200` baud). **UART1** Port 3 mapping conflicts with the 32-bit EMC/SDRAM bus and remains disabled.
- **UART2** uses `P0.10/P0.11`; `JP6` is required for the DB9 route (RS-232 voltage levels, not TTL).

## Works cited / local references

- Embedded Artists, **OEM Base Board Rev C schematic**: `ealpc1788/public/assets/aistudio/OEM_Base_Board_rev_C_954e1d3b.pdf`
- Embedded Artists, **EA LPC1788 Linux BSP Guide** (reference only): `ealpc1788/public/assets/aistudio/linux-EA-LPC1788-bspg-1.12.0_4d1d3944.pdf`
- Embedded Artists, **UART2 RS-232 sample README**: `/home/bg9211014/ss4/eanatty/lpc1788_samples/s_ob_uart_rs232/readme.txt`
- Embedded Artists, **USB host sample README**: `/home/bg9211014/ss4/eanatty/lpc1788_samples/s_usbhostlite/ReadMe.txt`
- Embedded Artists, **joystick sample source**: `/home/bg9211014/ss4/eanatty/lpc1788_samples/s_ob_joystick/src/main.c`
