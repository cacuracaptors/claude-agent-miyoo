# Using the Claude Agent tools well

Read this before any technical work on the Miyoo (docs {"topic":"agent"}). Other topics: hardware (the
devices), onionos (the system and its files), drastic (Nintendo DS emulator), ports (making and testing ports,
living with OnionOS), performance (what made ports faster or smoother on this hardware, with measured numbers,
what failed, and a checklist: read it before any speed, smoothness or battery work), emulators (systems, folders,
cores, BIOS, saves, RetroArch settings and troubleshooting), scraping (box art and game names).
These notes are for you, not for the person: explain things to them in plain words.

## How the connection works
- Claude -> Cloudflare relay (MCP) -> WebSocket -> `claude-agent connect` on the Miyoo. Each tool call runs in a
  worker thread on the device; the answer goes back the same way.
- Claude's side gives up on a request after about 100 s. The relay answers by 75 s: if the Miyoo is still working,
  the answer says so and gives an id. Do not repeat the call: use `wait_result {"id": N}` (waits up to 70 s, call
  it again if still running). Answers are kept 30 min and given once.
- If the Miyoo's Wi-Fi drops, the relay waits up to 20 s for it to come back. "Not connected for the last 20 s":
  try again in a few seconds before asking the person.
- New tools only appear in new chats (the tool list is read when a chat starts).

## The person and the screen
- The person may read the Miyoo instead of the chat: after each answer, `show_message` with a short plain-text
  version, in the Miyoo's language (`device_info.language`).
- Working for more than ~1 minute without touching the device: `set_status` ("Writing the fix...", same language),
  clear it with empty text when done. It shows next to ACTIVITY and expires by itself (default 5 min).
- Big or risky changes: `show_plan` first and wait for A. Small changes and looking around do not need it.
- Permissions are per category (screen, read, write, control = buttons, command) and asked on the Miyoo. A refusal
  is final for that request: explain and do not insist.
- Everything `write_file` replaces is backed up; `delete_path` moves to backups. The person can undo from the app.

## Leaving the app: background_mode
- While the app is on screen, the agent runs next to it. To launch games or apps (run_app), the app must close:
  turn on `background_mode` (the person confirms with A). It lasts until restart or until turned off.
- After run_app ends, the app is reopened. If the GameSwitcher is on screen (MENU pressed in a game), press B.

## Running and testing programs: run_app
- `launch` (a .sh like /mnt/SDCARD/Roms/PORTS/Game.sh or /mnt/SDCARD/Emu/<SYS>/launch.sh) + `rom` for emulators,
  or `command`. It starts it like OnionOS does (MainUI closes, runtime.sh runs it), keeps all output in
  App/ClaudeAgent/runs/<time>-<name>/output.log, and reports exit code or crash: which program, which signal,
  out-of-memory kills, last output lines, last_frame.png, core dump.
- Options: timeout_s, watch_for + stop_on_match, shots_every_ms/shots_start_ms/shots_count, cpu_mhz (never above
  the model's cpu_max_safe_mhz), env, core_dump. wait false returns at once; follow with job_status.
- Stopping (timeout, cancel, stop_on_match) sends SIGTERM to the programs first (not to the shell scripts that run
  them), waits up to 10 s so games can save and close files and the launch script can finish its cleanup, then
  stops what is left (SIGTERM, then SIGKILL).
- DraStic (NDS) has extra options, see topic drastic: drastic_state, drastic_input_record, drastic_input_playback.
- Only one run at a time. It needs MainUI (main menu) on screen to start.

## Jobs
- Long tools (download_file, run_app, input_play, input_record, perf_monitor, screen_capture, watch_file) wait up
  to 50 s, then go on as background jobs: `job_status` (no id lists them; id + wait_s up to 50 s; cancel true).
- Jobs are kept in memory: they are lost if the agent restarts (device restart, app update).

## Buttons
- `press_buttons`: tokens separated by spaces: A, MENU+POWER (together), DOWN*3, A@1500 (hold ms), WAIT:500.
  Works in games: while simulated buttons are used, the button driver's polling is paused
  (/sys/class/input/event0/device/poll 10 -> 0) and restored after, so presses are not released after 10 ms.
  The person's real buttons do nothing during that time.
- MENU in a game opens OnionOS's GameSwitcher (or quits). POWER can sleep or turn off the device: avoid it.

## Repeating the same input: three ways, choose by how exact it must be
1. Generic (any program): `input_record` (the person plays) and `input_play` (recording or timeline). Times are
   wall-clock milliseconds. Good for menus and tests where a few frames do not matter.
2. Aligned start: add `start_on` (text that appears in the run's output.log, or in start_on_file) to both. Time 0
   becomes the moment that line appears (DraStic: "Loading state"). Removes the loading-time difference.
3. Frame-exact (DraStic only): DraStic's own input record/playback (topic drastic). Replays a game identically,
   because the emulator applies every change on the same emulated frame. Use this for benchmarks and bug repros.
- Why 1 and 2 are not frame-exact: emulators do not run at a constant speed (DraStic emulates ~47 fps in the first
  second after loading and ~60 later), so the same millisecond lands on different frames, and games with physics
  or randomness drift apart. For your own ports, a frame counter in the program is the exact way.
- Keep MENU out of replays. Start every replay from the same save state.

## Measuring
- `perf_monitor`: CPU %, real clock (PLL), free memory, battery, and one process (process: name). CSV saved.
- `screen_capture`: screenshots every N ms without slowing the game; `view_image` to look at them.
- `watch_file`: follow a log for words. `snapshot`: save a folder (saves, a port) and restore it exactly.
- Measure before and after any change; do not assume. The screen refresh is ~59.7 Hz on the Mini Plus.

## Files and commands
- `run_command` runs BusyBox sh as root (default cwd /mnt/SDCARD; timeout_s up to 600, longer goes to wait_result).
  BusyBox has no nm/readelf/gdb; `strings`, `hexdump`, `md5sum`, `dd` exist.
- `read_file` up to 512 KB per call (offset for more; base64 for binary). `write_file` for text and small files,
  `download_file` (https, sha256) for anything big, for example a build from a GitHub release.
- The internal flash (root filesystem) is read-only squashfs; never write to /dev/mtd*. Work on /mnt/SDCARD.

## Plugins
- With developer tools on, `plugin_create` makes a new tool (sh script + description) the person approves; read
  `plugins {"action":"docs"}` first. Use it for repeated, specific jobs.
