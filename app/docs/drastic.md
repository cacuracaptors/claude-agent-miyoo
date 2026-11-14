# DraStic (Nintendo DS) on the Miyoo

## Setup
- Emu/NDS/: drastic (closed-source DraStic r2.5.0.4 by Exophase, with full DWARF debug info and symbols), the
  steward-fu SDL2 layer (libs/), launch.sh, config/drastic.cfg (frameskip, threaded_3d...), game_database.xml,
  usrcheat.dat, input_record/, input_capture_crash.ir (written by DraStic when it crashes).
- launch.sh stops the audioserver, sets the CPU clock, on 752x560 models sets the framebuffer mode before start
  (DraStic picks its layout once, at start), then runs `./drastic "$1"`.
- Speed (Mini Plus, 1800 MHz, Mario Kart DS): emulation at full speed with auto frameskip (about 20-45 drawn
  fps); one core does almost everything (3D rasterization ~22%, emulated CPU ~18%, waiting ~30%).
- SIGTERM makes DraStic close cleanly ("Oops sigterm"): it saves the auto state and closes its files. But if the
  signal arrives while it runs emulated code, it crashes while saving (SIGSEGV, "Saving state" then a register
  dump). Send it while the main thread sleeps between frames (/proc/<pid>/task/<pid>/stat state S): run_app's stop
  does that. Give it a few seconds; SIGKILL loses the state and the input recording.

## Save states
- Saves/CurrentProfile/states/Drastic/<rom name>_<slot>.dss. <rom name> is the zip/nds file name without its
  extension: "New Super Mario Bros.zip" -> "New Super Mario Bros_10.dss" (an older name with an extra dot, like
  "New Super Mario Bros._10.dss", is not found).
- Slot 10 is loaded automatically at start ("Loading state ..." in the output) and WRITTEN at exit. For repeatable
  tests keep a copy of the starting state and put it back before every run: run_app's drastic_state does that.
- Battery saves: <rom name>.dsv in the same folder.

## Frame-exact input record and playback (built into DraStic)
- `drastic --input-record <file> <rom>` writes every button change with the emulated frame number;
  `drastic --input-playback <file> <rom>` replays it on exactly the same frames: the game repeats identically
  (tested with New Super Mario Bros.: 50 s of play replayed exactly as it was played). The file is written when DraStic exits
  cleanly (SIGTERM or its menu), so stop it with SIGTERM, not SIGKILL.
- File format: 10-byte records, little endian: uint32 frame, uint32 buttons (bit mask), uint16 (touch/other).
  Frame numbers continue from the save state's frame, so record and replay must start from the same state.
- With run_app on Emu/NDS/launch.sh: drastic_state (a .dss to put in slot 10 first), drastic_input_record or
  drastic_input_playback (a .ir path). run_app runs a copy of launch.sh with the option added. Typical test:
  1. run_app {launch, rom, drastic_state: base.dss, drastic_input_record: x.ir, wait false}; the person plays;
     then job_status {id, cancel true} (SIGTERM: the recording is saved).
  2. run_app {launch, rom, drastic_state: base.dss, drastic_input_playback: x.ir, timeout_s, shots...} as many
     times as needed (perf comparisons, bug repros).
- Whether DraStic still reads the real buttons during playback is not known: ask the person not to touch them.

## Measuring
- The test copy Emu/NDS/launch_teste.sh preloads /mnt/SDCARD/drastic_teste/libdshook3.so, which logs emulated /
  drawn fps and idle time every second to /mnt/SDCARD/drastic_teste/fps.log (patches update_screen /
  update_frame of this exact drastic binary). Notes of that work: drastic_teste/progresso.md.
- LD_PRELOAD libraries that use SIGPROF crash DraStic (it uses SIGSEGV handlers and sigaltstack): sample from
  another process instead.
