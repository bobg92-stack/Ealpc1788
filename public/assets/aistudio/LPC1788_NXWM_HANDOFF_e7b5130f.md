# LPC1788 / NuttX NXWM Troubleshooting Handoff

Updated: 2026-11-27

## Objective and current state

Get touchscreen taps on the NXWM start-window icons to launch their applications on the EA LPC1788 hardware, then validate the rest of the active build against the attached board/peripherals.

Current symptom: touchscreen calibration completes and the calibration touches feel responsive, and the LCD draws the NXWM start screen. However, taps on the start-window icons have not produced a visible application launch. Board serial logs have shown `Touch map ... screen=(x,y) buttons=1` and `Touch NX: nx_mousein returned 0`, including press/release samples. One- and two-tap attempts did not show a `StartWindow: icon ... clicked` message. The board has previously appeared frozen after tapping an icon, but it is not yet established whether it is a whole-system lockup or a UI/application hang.

The previously tested diagnostic image established that NX receives touch callbacks, but no calculator app launches. A follow-up image with queue/worker and widget-hit diagnostics has now been built successfully and is ready for ISP flashing. **That follow-up image has not yet been flashed or tested.** Therefore the root cause remains unconfirmed; current evidence points to a failure after NX callback delivery and before the start-window action launches the application.

### Latest one-tap test (2026-11-27)

The user has now tested one tap on the calculator icon with the diagnostic build. The output establishes:

- `nx_mousein()` returns 0 for press, held/move, and release samples.
- NX callback delivery is working: press callback at local `(98,26)`, subsequent held callbacks near `(100,22..31)`, and release callback at `(102,31)`.
- The user-workqueue poll callback ran for the initial no-button event, then for the press. The press poll saw `press=1 release=0 held=1`, and `Widget hit test: (98,26) widgets=3` ran.
- No later `NX poll` line appeared for the held samples or release, and no `StartWindow: icon N clicked` line appeared.
- The user clarified that the system is **not frozen**: further touchscreen input continues to print in the serial terminal. The NSH prompt does not return while the foreground `nxwm` command is active; that alone is expected and is not evidence of a lockup.
- A follow-up test launched `nxwm &`, returned to NSH, tapped the calculator area, then successfully ran `ps`. The tap again produced one `NX poll` for the press and a widget hit test at local `(84,17)`, followed by move/release `NX callback` messages but no further `NX poll` or icon-click message. `ps` showed `nxwm` waiting for a signal, its NX listener waiting on the message queue, the touchscreen listener waiting on a semaphore, `NX_Server` waiting on its queue, and `lpwork` waiting on its semaphore. No calculator task appeared. This confirms the shell, NX server, touchscreen listener, and work-queue thread were alive/idle at the time of `ps`; it does not indicate a hard freeze.

This narrows the remaining failure to after the first press hit-test: the release state is not being processed by `pollEvents()` in the captured transcript, so the image's press/release action is never observed. Since `nx_mousein()` succeeds, the NX callback logs the release, further touch input continues, and `ps` shows the service threads alive, panel mapping, NX server routing, and overall system liveness are demonstrated. The workqueue thread is idle at the later `ps` snapshot, so it is not obviously stuck running; the missing later `NX poll` messages point instead to checking whether per-event work submissions are accepted and whether every callback schedules the worker. Do not treat this as proof of a particular workqueue bug until queue submission results and worker-entry traces confirm it.

### Direct serial diagnostics (2026-11-27)

The agent connected to `/dev/ttyUSB0` with picocom at 115200 after the user's session released the port. No reset or flash was performed. NSH responded normally. At the time of inspection, `nxwm` was running in the background and `ps` showed:

- `nxwm` PID 7 waiting for a signal.
- `NX_Server` TID 10 waiting on its message queue.
- NX listener TID 11 waiting on its message queue.
- Touchscreen listener TID 12 waiting on a semaphore.
- `lpwork` TID 2 waiting on its semaphore.
- No calculator app task.

`free` reported 32,551,852 bytes total, 35,228 used, and 32,516,624 free. `/dev/input0` exists as a readable character device. `dmesg` returned no entries. These results show healthy idle threads and ample free memory; they do not explain the missing click. No tap arrived during this direct session, so no new live touch callback was captured.

### Follow-up trace build (2026-11-27)

Added targeted diagnostics to the existing NXWM/NXWidgets code:

- `CWindowMessenger::handleMouseEvent()` now logs each `work_queue(USRWORK, ...)` return code, the work-state pointer, event coordinates, and button state.
- `inputWorkCallback()` logs worker entry with pending press/release state and coordinates, then logs poll result and worker exit.
- `CWidgetControl::handleLeftClick()` reports each candidate widget's position/size/enabled state and whether it accepted the point, including the selected widget pointer.
- `pollMouseEvents()` reports whether a release found a selected widget and the release coordinates.
- A failed mouse work-queue submission now frees the allocated state object after logging the error.

These diagnostics distinguish work submission/execution from widget hit testing and release/action behavior. They are additional tracing, not a confirmed functional fix.

## System and hardware identity

- MCU/board: EA LPC1788 OEM Board Rev E on EA Base Board Rev C.
- Display: EA LCD Board Rev A, 480 × 272 resistive touchscreen (TSC2046-compatible controller).
- Board photos supplied in chat show the LPC1788 module seated on the base board and the LCD connected by the flat ribbon cable. No obvious cable disconnect is visible. The photos do not show jumper positions clearly enough to confirm them.
- Relevant supplied manuals/schematics are under `ealpc1788/public/assets/aistudio/`:
  - `linux-EA-LPC1788-bspg-1.12.0_4d1d3944.pdf`
  - `OEM_Base_Board_rev_C_954e1d3b.pdf`
  - `LPC1788_OEM_Board_revE_9206cdfc.pdf`
  - `LPC1788_OEM_Board_Users_Guide_5b48ef17.pdf`
  - `LPC1788_OEM_Board_Datasheet_Rev_E_62b7c942.pdf`
  - `LCD_Board_Users_Guide_55b8b8db.pdf`
  - `LCD_Board-measurements_1f9ed01b.pdf`
  - `LCD-Board-rev-A_c235a4f8.pdf`
- Touchscreen wiring found in the EA base-board schematic: P0.15 SSP0_SCK, P0.17 SSP0_MISO, P0.18 SSP0_MOSI, P0.20 chip select, P2.15 PENIRQ. The configured SSP0 path and calibration operation have worked on this hardware.
- LCD config is 480 × 272, RGB565/16 bpp, 50 Hz; H pulse/front/back = 2/5/40, V pulse/front/back = 2/8/8, framebuffer base `0xa0010000`. These settings match the LCD board documentation reviewed so far.
- External SDRAM is configured as 32 MiB. External heap offset is 1 MiB; framebuffer allocation is separate from that heap region in the reviewed configuration.
- Other peripherals enabled/configured include LAN8720 Ethernet, CAN1 at 250 kbit/s, SD card, UART0 console, UART2, ADC, DAC, I2C0, RTC, watchdog, and joystick. Actual peripherals connected during the current test have not been confirmed.

## Software source and active build

- NuttX source tree: `/home/bg9211014/lpc1788_workspace/nuttx`
- Apps source tree: `/home/bg9211014/lpc1788_workspace/apps` (NuttX `.config` sets `CONFIG_APPS_DIR="../apps"`)
- NuttX runtime identifies as `NuttX-13.1.1-RC0`.
- Board/config: `open1788:ea_gui`; active configuration is in `nuttx/.config`, with template at `nuttx/boards/arm/lpc17xx_40xx/open1788/configs/ea_gui/defconfig`.
- Build is flat (`CONFIG_BUILD_FLAT=y`) and boots `nsh_main` (`CONFIG_INIT_ENTRYPOINT="nsh_main"`). Architecture-document guidance specific to protected builds/syscalls does not directly apply to this active flat build.
- Important active config: `CONFIG_NX=y`, `CONFIG_NXWM=y`, `CONFIG_NXWIDGETS=y`, `CONFIG_NX_XYINPUT=y`, `CONFIG_NX_XYINPUT_TOUCHSCREEN=y`, `CONFIG_NXWM_TOUCHSCREEN=y`, `CONFIG_NXWM_TOUCHSCREEN_DEVPATH="/dev/input0"`, `CONFIG_INPUT_ADS7843E=y`, `CONFIG_ADS7843E_SPIDEV=0`, `CONFIG_ADS7843E_SPIMODE=0`, `CONFIG_ADS7843E_FREQUENCY=100000`, `CONFIG_NXWM_CALIBRATION_MARGIN=25`, `CONFIG_LPC17_40_SSP0=y`, `CONFIG_LPC17_40_LCD=y`, `CONFIG_LPC17_40_LCD_BPP16_565=y`.
- `ea_gui/defconfig` was aligned to the active/proven touchscreen settings: SSP mode 0 and calibration margin 25. That changes future config regeneration, not the already active `.config` unless the config is regenerated.
- The board pinout comments in `nuttx/boards/arm/lpc17xx_40xx/open1788/src/open1788.h` were corrected to match the EA schematic. This file had existing local modifications; only comments were intended to be adjusted.

## Diagnosis so far

The intended input path is:

1. ADS7843E driver reads `/dev/input0`.
2. NXWM `CTouchscreen` reads events, calibrates/maps raw coordinates to screen coordinates.
3. `nx_mousein()` injects coordinates/buttons into the NX server.
4. NX routes mouse callbacks to `CWindowMessenger::handleMouseEvent()`.
5. The callback schedules the input work callback, which calls `pollEvents()`.
6. `CWidgetControl` hit-tests widgets and generates press/release behavior.
7. The start-window image action calls `CStartWindow::handleActionEvent()`, which should print the icon index and start the application factory.

Evidence currently localizes the issue only to after `nx_mousein()` returns success. That return value means the injection API accepted the sample; it does **not** prove a window callback ran or an icon widget was hit.

There is an existing local change in `CWidgetControl::newMouseEvent()` / `pollMouseEvents()` to retain press/release edge flags and their coordinates until asynchronous polling consumes them. The rationale is that an NX callback may be followed by another sample before the worker polls, and the old logic could clear/overwrite a quick edge. This is plausible but has not yet been demonstrated as the root cause. The diagnostic build was prepared to distinguish it from callback delivery, work-queue scheduling, coordinate transforms, widget hit-testing, or app launch.

Diagnostics included in the current apps sources:

- `CWindowMessenger::handleMouseEvent()` prints `NX callback: local=(x,y) buttons=...`.
- `CWindowMessenger::inputWorkCallback()` prints `NX poll: pending press=... release=... held=...` before polling.
- `CWidgetControl::handleLeftClick()` prints `Widget hit test: (x,y) widgets=n`.
- `CStartWindow::handleActionEvent()` already prints `StartWindow: icon N clicked`.

Interpret the next serial log as follows:

| Last observed diagnostic | Likely area to inspect next |
| --- | --- |
| Touch-map and `nx_mousein returned 0`, but no `NX callback` | NX server/window callback registration, window geometry/ownership, or server event delivery |
| `NX callback`, but no `NX poll` | work-queue submission, callback lifetime/state, or queue execution |
| `NX poll`, but no `Widget hit test` | XY event state/edge processing between callback and widget controller |
| `Widget hit test` with coordinates outside icon bounds | screen-to-window coordinate origin, window geometry, or hit-test bounds |
| Hit-test coordinates inside an icon but no icon click | widget press/release sequence or widget/image action wiring |
| `StartWindow: icon N clicked` then freeze/no app window | application factory/startup, memory/stack, or NX window lifecycle |

Do not use `hexdump /dev/input0` as an interactive diagnostic; it previously blocked the shell until reset. Use the NXWM diagnostic output and safe shell commands such as `ps`, `free`, and `dmesg` if available.

## Local changes and preservation

Both repositories contain substantial local modifications. They are user work and must be preserved. Do not reset, stash, rebase, or discard unrelated modifications.

At the time of this handoff, the NuttX tree has modified driver/board files including ADC, CAN, DAC, Ethernet, I2C, LCD, SD card, board bring-up/buttons/joystick/LCD/SDRAM/SSP/touchscreen, ADS7843E driver/header, plus `open1788.h`; it also has untracked build/helper scripts and the `ea_gui` config directory. The apps tree has local changes across NXWM and NXWidgets, including calibration, touchscreen, start window, application window, window messenger, widget control, and related headers. Inspect `git status --short` in each tree before editing, and keep changes narrowly scoped.

The diagnostic files are:

- `apps/graphics/nxwm/src/cwindowmessenger.cxx`
- `apps/graphics/nxwidgets/src/cwidgetcontrol.cxx`
- `apps/graphics/nxwm/src/cstartwindow.cxx`
- related existing modifications in `apps/graphics/nxwm/src/` and `apps/include/graphics/`
- calibration sample/coefficient logging in `apps/graphics/nxwm/src/ccalibration.cxx` and `ctouchscreen.cxx`
- per-icon creation/start results in `apps/graphics/nxwm/src/cstartwindow.cxx`

## Reproducing the active build

The working build used the active `.config` in the NuttX tree and the sibling apps tree. The toolchain installed on the host provides `arm-none-eabi-*`; the NuttX configuration expects `arm-nuttx-elf-*`. A temporary shim directory was created under `/tmp/vscode-agent-codex-sandbox-BnhALL/lpc1788-toolshim` with symlinks from the expected NuttX tool names to the installed `arm-none-eabi-*` tools.

To reproduce the same active build without replacing the config, from the workspace shell:

```sh
cd /home/bg9211014/lpc1788_workspace/nuttx
PATH=/tmp/vscode-agent-codex-sandbox-BnhALL/lpc1788-toolshim:$PATH make -j4
```

If that temporary directory has been removed, recreate the tool aliases (assuming `arm-none-eabi-*` tools are installed):

```sh
mkdir -p /tmp/lpc1788-toolshim
for tool in gcc g++ ld ar objcopy objdump nm size readelf strip ranlib; do
  target="$(command -v "arm-none-eabi-${tool}")"
  test -n "$target" && ln -sf "$target" "/tmp/lpc1788-toolshim/arm-nuttx-elf-${tool}"
done
```

Then build with:

```sh
cd /home/bg9211014/lpc1788_workspace/nuttx
PATH=/tmp/lpc1788-toolshim:$PATH make -j4
```

For a fresh configuration in the future, first preserve the current `.config` and avoid cleaning the tree unless needed. The template is `boards/arm/lpc17xx_40xx/open1788/configs/ea_gui/defconfig`; config regeneration should be an explicit, deliberate step because it can replace active settings. The apps directory is expected at `../apps`.

Latest recorded full diagnostic build (2026-11-27, after the user asked to use the flash budget for better diagnosis):

- `make olddefconfig` and `PATH=/tmp/vscode-agent-codex-sandbox-BnhALL/lpc1788-toolshim:$PATH make -j4` completed successfully.
- Linker report: FLASH 467,848 bytes of 512 KiB (89.23%); SRAM 35,072 bytes of 64 KiB (53.52%). Flash reserve is 56,440 bytes.
- Enabled global NuttX error/warning output and debug assertions. `CONFIG_DEBUG_INFO` remains off because it enables very chatty output that can distort touchscreen timing and overwhelm the serial link.
- Added explicit calibration target/raw-point/coefficient logs, touch raw/pressure/mapped-coordinate logs, NX queue and worker traces, widget candidate/hit/release traces, and icon app-create/app-start result logs.
- Output raw image: `/home/bg9211014/lpc1788_workspace/nuttx/nuttx.bin`, 467,848 bytes; SHA-256 `ef85d2b707ea004bf6a6dc416abeaa6b4f8e416937ce7ff9330e41912f8dee74`.
- Intel HEX image: `/home/bg9211014/lpc1788_workspace/nuttx/nuttx.hex`, 1,315,995 bytes; SHA-256 `0abef2a3bf051552c2d388c2a4eac13f37e949997d3ba619ccdbedc9f09ea24c`.
- The raw `.bin` was generated from the successfully linked ELF using `arm-none-eabi-objcopy -O binary`. lpc21isp 1.97 knows the LPC vector-checksum location and writes that checksum during ISP programming.
- The prior diagnostic image, ELF, map, and `.config` were preserved under `build_backups/nxwm-full-diagnostic-20261127/pre-change/` before rebuilding.
- `git diff --check` passed in both NuttX and apps repositories.
- At the time of this build record the image had not yet been flashed. The later sections below record the successful flash and hardware traces.

## Flashing

Use 57600 baud or lower for LPC1788 UART ISP. Although the user manual's generic Set Baud Rate table includes 115200, the LPC177x/8x errata (ISP.1) specifically limits UART ISP to 57600 for affected revisions, including the LPC1788 revisions listed in the errata. The erratum overrides the generic command table.

Recommended command for the current full diagnostic image:

```sh
lpc21isp -control -try100 -verify -bin /home/bg9211014/lpc1788_workspace/nuttx/nuttx.bin /dev/ttyUSB0 57600 12000
```

Close picocom first so it releases `/dev/ttyUSB0`. This uses the raw 467,848-byte image and omits `-writedelay`; if synchronization is unreliable, retry with `-writedelay` or reduce to 38400. Keep `-verify` for this diagnostic image; omitting it saves time but removes the post-write check. Do not try to switch the ROM ISP session above 57600: the ROM supports a Set Baud Rate command, but the silicon erratum makes higher-rate UART ISP unsupported. No image was flashed during the latest build.

## Next steps

1. Flash the current `nuttx.bin` with the command above, then boot and launch `nxwm &` so NSH remains available. Confirm that `NX queue` and `NX worker begin/end` appear on a touch before interpreting the next test.
2. Complete calibration once and save its raw four-point readings and coefficients. Tap the calculator icon once, then the green arrow/edge target once. Capture `Touch map`, `NX callback`, queue/worker, widget candidate/hit/release, and start/create result lines. Run `ps` immediately afterward and confirm whether an app task/window appears.
3. Use the first missing trace boundary to make the fix: failed queue submission, missing worker execution, no widget accepts the local coordinates, or release/action handling after a widget is selected.
4. Rebuild the same active configuration after the fix, record linker memory and image hash, then flash/test. Keep diagnostics until the full press-to-action path is verified; remove or gate the traces in a later production build.
5. Once icon launch works, validate other enabled hardware one subsystem at a time against the EA schematics and actual connected peripherals. Prioritize SD card, Ethernet, CAN, UART2, joystick/buttons, ADC/DAC, RTC/watchdog, and SDRAM. Calibration proves part of the touch path; it does not validate these independent peripherals.

## Useful board commands and test capture

At NSH, the prior `help` output showed `ps`, `free`, and `dmesg` are present. `nxwm` is a builtin app and can be started manually. The expected debug transcript should start with:

```text
nsh> nxwm
```

After one tap, capture the entire serial transcript. Do not type into a blocked NSH while the touchscreen input device is being read by another diagnostic command; specifically, the direct `hexdump /dev/input0` attempt previously appeared to freeze the shell.

## Remaining unknowns

- Runtime output from the newly built queue/worker and widget-hit diagnostic image after it is flashed.
- Whether the board truly hard-locks on the first app icon or only the UI becomes unresponsive.
- Exact jumper positions and all peripherals connected during tests.
- Whether each enabled non-display peripheral works on this individual board.
- The complete identity/hash of the exact image being flashed; record a hash alongside future builds if comparing images is needed.

## 2026-11-27 jumper and physical-connection photo review

New close-ups are stored under `jumpers/` in the workspace. They show the EA OEM base board marked Rev C and the red LPC1788 OEM board marked Rev D. The LCD ribbon is visibly attached at both ends in the overview photos; the LCD is illuminated and has displayed calibration and the NXWM start screen. The photos therefore support that the panel path is powered and producing graphics, while not proving every signal or jumper position electrically.

The base-board user's guide (section 4.25) says JP22–JP27 select the I2S codec signal group and must be set to the upper position for the LPC1788 OEM board. The close-ups include that jumper region, but orientation/position is not clear enough to certify every shunt against the guide diagram. These are audio routing jumpers, not the touchscreen SSP0/PENIRQ path, so they are low priority for the current icon-activation fault. Do not move them as an experiment while diagnosing NXWM.

The base-board Rev C schematic and user's guide warn that some jumpers are board-revision/use dependent and that the default diagram is not a guarantee of the actual board setup. In particular, USB1 OTG and the LCD interface share P1.27–P1.29 and cannot be used simultaneously; the active setup needs the LCD. Do not add a USB1 OTG test into the LCD configuration without resolving this conflict. The observed working LCD output implies the current display wiring/pin setup is at least sufficient for scanout. No photo establishes a hardware cause for the NXWM failure; the serial trace shows that touch samples reach NX and the app/widget callback path remains the active software lead.

No jumper was changed. The picture review is informational only; a definitive audit of any specific shunt requires a straight-on, sharp photo that includes its silkscreen reference and the board's orientation, compared with the correct Rev C diagram.

## 2026-11-27 live board session (before full diagnostic image flash)

Connected to `/dev/ttyUSB0` with picocom at 115200 baud after the user's reset. NSH responded to Enter. Before launching NXWM, `ps` showed only Idle, hpwork, lpwork, `nsh_main`, and `telnetd`; `free` reported 32,551,852 total bytes and 32,534,312 free; `dmesg` printed no entries. This is a healthy-looking idle baseline, not evidence of an early boot hang.

Started `nxwm &` successfully (PID 7). NXWM initialized the taskbar, start window, touchscreen listener, calibration, NxTerm, and calculator, then reached `nxwm_main: Waiting for touchscreen calibration`. After the user completed calibration, a reconnect showed NXWM PID 7 and its NX listener/touchscreen threads active, with no application task. `free` then reported 32,551,852 total bytes and 32,516,544 free.

During a live tap, the serial output showed the complete touch sample sequence and `nx_mousein returned 0`, followed by `NX callback: local=(81,30) buttons=01`, move callbacks, and `NX callback: local=(83,20) buttons=00` on release. It did **not** show the follow-up source diagnostics `NX queue`, `NX worker begin/end`, or `Widget hit test` that are present in the latest source/build. A subsequent `ps` still showed only NXWM, its two threads, NX_Server, NSH, and telnetd; no launched calculator/terminal task appeared. `free` reported 32,551,852 total bytes and 32,516,304 free. Thus the currently running board image appears older than the latest queue/worker diagnostic build (or the console was not running that built image); this touch test confirms the touch-to-NX callback boundary but cannot yet diagnose work-queue execution or widget activation. The current build hash remains `8f8e8a1609c50657ecb37bb6bfa3f3d9bed43d957c804acabba9fb4d6f43c9e6`; the handoff previously recorded it as not flashed. Do not infer that the latest diagnostic code was tested on hardware until the image identity is confirmed.

The user's latest serial capture identifies the first target as the calculator and the next targets as the green arrow and scale/target graphic. Calculator tap: raw `(901,2720)` -> screen `(90,77)`, callback local `(86,19)` on press, moving to local `(88,34)` before release. Arrow/edge tap: raw `(156,3688)` -> screen `(-8,7)`, callback local `(0,7)`; the negative screen X means this calibration maps that touch just beyond the display edge and NX receives a clipped/local edge coordinate. Third target: raw `(350,2748)` -> screen `(19,76)`, callback local `(15,18)` on press, with subsequent movement/release. All these targets produced touchscreen samples, `nx_mousein returned 0`, and NX callbacks, but none produced `NX queue`, `NX poll`/widget-hit diagnostics, or an app-launch line. The user's following `ps` again shows NXWM and its listeners active with no app task. This supports working touch hardware and dispatch through the NX callback; it does not yet prove the widget click/release action or app launch. The edge-coordinate offset deserves calibration review after the up-to-date diagnostic image is confirmed.

### Likely regression boundary from source comparison

The strongest candidate for the homescreen-wide failure is the asynchronous widget event path in `apps/graphics/nxwidgets/src/cwidgetcontrol.cxx`, rather than the ADC calibration arithmetic alone. In the original version, each `newMouseEvent()` clears pending mouse-edge flags before recording the new sample. `CWindowMessenger` schedules `pollEvents()` asynchronously on a work queue, so a following MOVE/UP can clear a pending press before the worker consumes it. The original poller also skipped release handling when another event (such as the press) had already been handled in that same poll, then cleared the flags. This can leave an icon with no complete press/release click. The current local patch changes this exact boundary: preserve pending edge flags, store separate press/release coordinates, handle both edges in order, and log widget selection/release. Calibration can still work because it consumes raw touchscreen samples in its calibration application, while launcher icons depend on this additional NXWidgets callback/queue/poll/action path.

The calculator coordinates specifically argue against a gross calibration offset as the cause of that tap: `CWindowMessenger`'s local `(86,19)` for global mapped `(90,77)` implies a window origin near `(4,58)`. With the active small icon configuration, calibration/NxTerm/calculator slots are laid out from x≈12, with 12-pixel minimum spacing; the calculator slot is approximately local x=84..108, y=12..37. Both observed calculator press `(86,19)` and release `(88,34)` fall inside that slot. This is an inference from the active layout code and configured small glyphs, not a widget hit trace; the running board image still needs the new instrumentation to confirm it.

There is a separate, confirmed edge-calibration issue: `CONFIG_NXWM_CALIBRATION_ANISOTROPIC=y`, and that branch in `ctouchscreen.cxx` computes scaled positions without clipping to display bounds. The logged arrow-area result `screen=(-8,7)` is therefore allowed through and can make a control at the left edge difficult or impossible to hit consistently. This matches the reported leftward touch difficulty, but by itself does not explain the calculator tap that maps inside its widget. The touch flags also show `0x39` DOWN and `0x3a` MOVE with `TOUCH_PRESSURE_VALID`, and an UP with a valid position, so the pressure gate accepted these particular samples; it is not the immediate explanation for these missed clicks.

Confidence: moderate that the NXWidgets press/release edge handling explains the general launcher behavior; high that the current anisotropic mapper can produce out-of-range edge coordinates; not yet proven which fix restores icon launching because the board's trace did not include the newest queue/worker/widget logs. Keep the latest image flash and callback-to-action trace as the next decisive test.

## 2026-11-27 full diagnostic image flashed and tested

The user flashed `nuttx.bin` with `lpc21isp -control -try100 -verify -bin ... 57600 12000`. The tool reported the expected 467,848-byte image, recognized LPC1788 / 512 KiB flash, and ended with `Download Finished and Verified correct... taking 306 seconds` followed by launching the image. This confirms the full diagnostic image is running; its `NX queue`/`NX worker` logs appeared.

Safe NSH checks after reset: `ps` showed idle, hpwork, lpwork, nsh_main and telnetd; `free` showed 32,551,676 bytes total with 32,534,136 free; `dmesg` was empty. `/dev` listed `adc0`, `can0`, `djoy0`, `fb0`, `i2c0`, `input0`, `mmcsd0`, `ttyS0`, and `ttyS1`. `i2c bus` reports bus 0 exists; no I2C address scan was run because the built-in command warns it can have side effects. `ifconfig` reports eth0 DOWN with address `00:00:00:00:00:00`; `df` shows only procfs mounted. No CAN/ADC/SD read-write tests were attempted without known loopback devices or card contents.

NXWM started as PID 9 and completed calibration. It logged all four raw calibration points: UL `(360,3459)`, UR `(3757,3407)`, LR `(3733,551)`, LL `(291,622)`, with a 480x272 target. Pressure was valid at all four down samples. Following calibration, a calculator-area press mapped to screen `(92,80)` and window-local `(88,22)`; subsequent moves and release mapped around local `(89,25)`. This lies within the calculated calculator image bounds. The edge/arrow tap mapped to screen `(3,13)` and local `(3,13)`, so this recalibration did not repeat the previous negative-X result.

The crucial event trace for the calculator-area press was `NX callback` -> `NX queue result=0` -> `NX worker begin` with `press=1` -> `Widget hit test: (88,22) widgets=3`. There was no `Widget candidate`, `Widget selected`, `NX worker end`, `Widget release`, `StartWindow: action event`, or app-start line after that hit test. Later touch samples still reached NX callbacks and `NX queue result=0`, but no further worker-start logs appeared. `ps` still showed no calculator/terminal application task; memory remained plentiful and `dmesg` empty.

Interpretation: touch hardware, pressure acceptance, coordinate mapping, NX callback delivery, and work submission all pass for this tap. Processing stops at or inside the first `candidate->click()` call in `CWidgetControl::handleLeftClick`; that call can invoke `CImage::onClick()` and redraw synchronously. The work trace does not distinguish a stuck widget click/redraw from a worker scheduling/state issue after the first hit test. The next narrow diagnostic should add a log immediately before `candidate->click()` and immediately after it, plus entry/exit markers around `CImage::onClick()`/redraw. Keep the 57600 ISP limit from the errata. The board currently has NXWM running and the serial console session active.

An initial repeat attempt still showed the prior NXWM PID 9 and a duplicate launch was correctly refused. A later user transcript showed fresh task numbering (NSH PID 4, NXWM PID 7) and is recorded in the next subsection; that later run confirms a physical reset did occur. No blank-area-vs-icon comparison has yet been collected.

### Second fresh-boot input attempt (user's pasted trace)

The next transcript shows a fresh NuttX task numbering (NSH PID 4; NXWM PID 7), full calibration, then a touch at mapped screen `(95,76)`, local `(91,18)`, followed by release at local `(95,24)`. This is again in the calculator-icon region, rather than the requested blank-area control touch. The same stop repeats: queue result is 0, an `NX worker begin` sees `press=1`, then `Widget hit test: (91,18) widgets=3` is the last worker-side line; there is no candidate/result, worker-end, release, action, or app task. Further move/release samples continue to get NX callbacks and successful queue results. This fresh-boot reproduction confirms the stall is deterministic on the first icon-area press and is not stale calibration state. The candidate result log is emitted after `candidate->click()` returns, so more detailed click/redraw markers were added in the next image build.

## 2026-11-27 detailed click/redraw trace image

The preceding fresh-boot trace repeatedly stopped after `Widget hit test` for a tap whose mapped local coordinates fell inside the calculator icon. `Widget candidate[...]` is printed after `candidate->click()` returns, so the evidence puts the stall inside that call (or just before it), but did not distinguish recursive hit testing, focus/click state changes, synchronous image redraw, or click-handler dispatch.

Added `NXDBG` stage markers to the existing local apps changes:

- `cwidgetcontrol.cxx`: print immediately before invoking each top-level candidate.
- `cnxwidget.cxx`: trace click entry/hit/reject, child recursion, focus and clicked-widget handling, `onClick`, and each stage of a clicked widget's redraw.
- `cimage.cxx`: trace clicked-image redraw, bitmap row generation, and entry/exit of each bitmap draw request.

This does not yet implement a fix; it is a targeted diagnostic to identify the exact blocking call from one hardware tap. The flat active build uses `USRWORK == LPWORK`, so a synchronous wait inside the click callback would stall later queued input work while the touchscreen listener can continue accepting and queueing samples. The previously built `nuttx`/`apps` source had no compile errors after adding these markers.

Rebuilt the active `open1788/ea_gui` configuration successfully with `make -j4`; then regenerated the raw binary using `arm-nuttx-elf-objcopy -O binary nuttx nuttx.bin`. Linker usage is FLASH 469,312 / 524,288 bytes (89.51%), SRAM 35,072 / 65,536 bytes (53.52%). `nuttx.bin` SHA-256 is `2f96cffd73092487eb50a713b81b051a3315e00645ff3f7cbaaab42c2e39c47b`; `nuttx.hex` SHA-256 is `146a5b8998ffa0c7c47a9ced160f2cc31fc02b011f1d82d0b63114289c3d0d19`. The prior image and configuration were saved in `build_backups/nxwm-click-trace-prebuild-20261127/`.

The new `.bin` was flashed with verification using `lpc21isp -control -try100 -verify -bin ... 57600 12000`; ISP reported `Download Finished and Verified correct... taking 307 seconds`, then launched the image. Connected to NSH at 115200, confirmed the post-boot baseline tasks, and started `nxwm &` as PID 7. The board completed calibration and the following live touch trace was captured.

The blank-area control tap mapped to screen `(86,68)` and local `(82,10)`. All three start-window candidates rejected it, and `NX worker end ... handled=0` appeared. A left-edge tap at screen `(0,3)` similarly missed the start button at `(2,2 24x24)` and the worker returned. Thus an unhandled touch and the queue/hit-test loop complete normally.

A later tap mapped to screen `(438,51)` and local `(434,19)`, hitting an image widget with rect `(420,0 40x25)` (not the calculator icon). The trace reached `NXDBG click onClick begin`, `NXDBG image onClick begin`, `NXDBG redraw begin`, `NXDBG redraw border done`, and `NXDBG image draw begin ... rect=(420,0 40x25)`. No later image-row, bitmap, image-draw-end, click-end, or worker-end marker appeared. Subsequent touch callbacks and `NX queue result=0` lines continued, but no work callback began. This proves the failure is in a generic `CImage` activation/redraw path, not specific to the calculator's hitbox. It does not yet identify the exact instruction after image-draw entry because current markers omit allocation/setup and the top/bottom padding bitmap loops.

After the stall, NSH still accepted commands. `ps` showed `nsh_main` running, NXWM and its listeners alive, NX_Server waiting for its message queue, and LPWORK waiting on a semaphore. `free` reported 32,551,676 total and 32,516,128 free. This is not a complete kernel lockup or heap exhaustion. The semaphore state is consistent with the sole LPWORK thread blocked inside a synchronous draw request, though the exact waiting operation still needs a marker around the omitted CImage paths and NX server completion.

### Calculator-icon trace on the second detailed build

The follow-up image was rebuilt, flashed, and verified in 307 seconds. The user completed calibration (raw UL `(301,3343)`, UR `(3701,3484)`, LL `(697,615)`, LR `(3705,582)`), then tapped the calculator icon. The mapped screen point was `(88,72)`, local `(84,14)`, and candidate index 2 had rect `(84,12 24x25)`, so the click hit the calculator widget.

This time the trace continued through `CImage::onClick`, `CNxWidget::redraw`, row-buffer allocation, image geometry `(origin=0,0; bitmap=24x25)`, and `getRun(row 0)`. It stopped immediately after `NXDBG image bitmap begin ... y=12`; the corresponding end marker never appeared. This pinpoints the block to the first `CGraphicsPort::drawBitmap()` call, which synchronously calls `nx_bitmap()` and waits for its completion semaphore. The NX bitmap/server trace was filtered to a different 40-pixel toolbar control, so it did not match this 24-pixel calculator icon; absence of those server logs is not evidence that the server failed to receive the request. A further build now filters the calculator's first row `(x=84..107, y=12)` and traces both `nx_bitmap()` send/wait and NX-server draw/post stages.

### Next hardware test

1. Add markers after the CImage row-buffer allocation and setup, plus around top-padding and bottom-padding `drawBitmap()` calls. Add NX server markers at bitmap-message receipt, after `nxbe_bitmap()`, and after `sem_done` post.
2. Rebuild the active configuration and flash the resulting `.bin` at 57600 baud with `-verify`; the current NSH connection must be closed first.
3. Repeat calibration, then tap a blank area and one launcher icon. Capture the first missing marker. If the server receives and posts the bitmap but the work queue does not resume, inspect the semaphore/message lifetime; if the server does not receive it, inspect the client send path; if it returns from bitmap drawing, inspect image setup/padding and click dispatch.
4. Make the smallest functional fix at that exact boundary, rebuild the active config, and verify press/release/action and launch on the board. Keep the previous binary in the backup directory for comparison.

