# Making and testing ports for the Miyoo Mini

## Building
- Cross-compile for ARMv7 hard float: arm-linux-gnueabihf-gcc (the miyoomini-toolchain Docker image), flags
  -marm -mtune=cortex-a7 -mfpu=neon-vfpv4 -mfloat-abi=hard. Link against the firmware's libraries (SDL2 from
  the steward-fu/miyoo SDL2 port, or the framebuffer + MI_GFX directly).
- Ship needed .so files next to the port and set LD_LIBRARY_PATH in its launcher.

## Port layout and conventions (Roms/PORTS)
- A launcher Game.sh in Roms/PORTS (cd to its folder, set LD_LIBRARY_PATH, cpuclock, start the binary) and the game
  folder. Box art in Roms/PORTS/Imgs.
- Clock from the model: 1800 MHz only on the Mini Plus (354), 1600 on the others; read /tmp/deviceModel and
  /tmp/screen_resolution (752x560 on Mini v4 and Flip).
- MENU quits with one click (on release, only if no other key was pressed with it: MENU+POWER is OnionOS's
  screenshot). Support OnionOS's GameSwitcher / resume where possible.
- Screen: 640x480 or 752x560, the framebuffer is upside down; MI_GFX can scale and rotate for free.
- Audio: stop the audioserver or go through libpadsp/MI_AO; mind latency.
- Memory: ~100 MB for everything; check with perf_monitor (process) for leaks.

## Testing with the agent
1. Upload the build: download_file from a GitHub release or raw URL (sha256), or write_file for small files.
2. snapshot the port's save folder before risky tests.
3. run_app {launch: Roms/PORTS/Game.sh, timeout_s, watch_for, shots_every_ms, cpu_mhz} and read the crash report:
   program, signal, last output, last_frame.png, core dump (core.<prog>.<pid> in the run folder; open with gdb and
   the unstripped binary on the build machine).
4. Repeatable input: input_record / input_play (+ start_on a line the port prints when ready). For frame-exact
   replays, add a frame counter and record/replay inside the port.
5. perf_monitor (wait false) during the run for CPU, clock and memory; screen_capture for visual checks.
