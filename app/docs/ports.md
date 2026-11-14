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

## Living with OnionOS / RedOnion (do these in every port)
- Screen mode at start: the previous program may leave /dev/fb0 at 752x560 (Mini v4/Flip paths) or with a single
  page (yres_virtual 480, seen on RedOnion), which silently disables page-flip vsync (tearing) or makes a
  640x480-only renderer show nothing. At start, if it is not 640x480, 32 bpp, at least 2 pages, set 640x480x32
  with virtual 640x960 (FBIOPUT_VSCREENINFO, like OnionOS's fbset -g 640 480 640 960 32), then re-read VSCREENINFO
  and FSCREENINFO before mmap, accept 3 pages (640x1440, the menu leaves that), and tell batmon and keymon
  (killall -SIGUSR1 batmon keymon) because they draw overlays on the screen. Do not restore the old mode at exit.
  Log the mode found. Test by forcing the bad modes in a test shortcut: fbset -g 752 560 752 560 32 and
  fbset -g 640 480 640 480 32 before the launch, fbset -g 640 480 640 960 32 after.
- Page 0 at exit and in the crash handler: OnionOS draws its menu on page 0.
- On 752x560 devices 640x480 is scaled by the hardware: no need to render 752x560 (DraStic is the exception: it
  needs the 752x560 mode set before it starts).
- Power off: OnionOS's keymon sends SIGTERM to non-RetroArch programs, deletes .tmp_update/cmd_to_run.sh and waits
  up to ~30 s. A port can save then: handle SIGTERM (it does not always arrive: also watch /tmp/.offOrder a few
  times per second), create a script in .tmp_update/checkoff/ so OnionOS waits, save to a hidden slot, put the
  cmd_to_run.sh back and leave a "resume" flag so the next boot reopens the game and loads it (holding MENU at
  boot cancels). Close open windows first (send ESC), skip saving in places the game cannot save (dialogs, world
  map, videos). Long-press power-off (10 s) waits for nobody. Fallout 1/2 do all of this.
- GameSwitcher: MENU can open it like in emulators (single/long/double press timing as keymon), save a screenshot
  to Saves/CurrentProfile/romScreens/<hash>.png, and resume through the same save path as power-off. Otherwise the
  port's MENU quits with one click on release (MENU+POWER is the screenshot combo).
- Overlays: volume/brightness bars and the low-battery icon are drawn by OnionOS on the screen; pausing frame
  presentation while they show keeps them visible. Low battery: a port can save and quit by itself.
- "Disable services in game" (Tweaks) stops SSH while a game runs: for testing, keep services alive
  (.keepServicesAlive) or use the agent.
- Clock by model at start (1800 only on the Plus); a release without it froze the other models.

## Testing with the agent
1. Upload the build: download_file from a GitHub release or raw URL (sha256), or write_file for small files.
2. snapshot the port's save folder before risky tests.
3. run_app {launch: Roms/PORTS/Game.sh, timeout_s, watch_for, shots_every_ms, cpu_mhz} and read the crash report:
   program, signal, last output, last_frame.png, core dump (core.<prog>.<pid> in the run folder; open with gdb and
   the unstripped binary on the build machine).
4. Repeatable input: input_record / input_play (+ start_on a line the port prints when ready). For frame-exact
   replays, add a frame counter and record/replay inside the port.
5. perf_monitor (wait false) during the run for CPU, clock and memory; screen_capture for visual checks.
6. Performance work: read docs topic performance first (measured techniques, what failed, checklist).
7. For a port's own frame-exact replays, record input per game frame inside the port (like DraStic's .ir files).
