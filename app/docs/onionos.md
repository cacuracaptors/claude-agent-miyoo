# OnionOS and RedOnion: how the software works

RedOnion is a fork of OnionOS with the same layout; device_info.os says which one (RedOnion has
.tmp_update/onionVersion/website.txt). Check the real files before changing anything: versions differ.

## Boot and the main loop
- The stock firmware runs /mnt/SDCARD/.tmp_update/runtime.sh (OnionOS takes over from there). It detects the
  model (writes /tmp/deviceModel: 283 Mini, 354 Plus, 285 Flip; /tmp/screen_resolution), starts keymon, then loops:
  1. MainUI (the menu) runs. When the person picks a game or app, MainUI writes /tmp/cmd_to_run.sh and exits.
  2. runtime.sh moves it to .tmp_update/cmd_to_run.sh and runs it (the game owns the screen and buttons).
  3. When it ends, back to 1. (The agent's run_app uses the same path: it writes cmd_to_run.sh and closes MainUI.)
- keymon (always running): global keys. MENU in a game opens the GameSwitcher (recent games, resume) or quits;
  MENU+POWER takes a screenshot (Screenshots/); volume and brightness keys; sleep and power off; low battery.
  It checks /tmp/stay_awake (the Claude Agent app touches it so the device does not sleep while open).
- The GameSwitcher is its own program (gameSwitcher). B goes back to the menu.
- Logs: .tmp_update/logs/ (MainUI.log, keymon.log, gameSwitcher.log...). Config flags: .tmp_update/config/
  (small files; .tz is the time zone).

## Where things are (SD card root /mnt/SDCARD)
- Roms/<SYSTEM>/ (NDS, GBA, SFC, PS, PORTS...), Roms/<SYSTEM>/Imgs/ box art.
- Emu/<SYSTEM>/: config.json (label, launch script, rom folder) + launch.sh. Most systems launch RetroArch with a
  core; NDS uses DraStic (Emu/NDS, topic drastic).
- RetroArch/: retroarch binary (retroarch_miyoo354 on the Plus), .retroarch/cores, config, cheats.
- App/<name>/: config.json ("label", "icon", "launch") + launch.sh. Apps: Tweaks, PackageManager, Overclock,
  Terminal, ThemeSwitcher, PlayActivity, ClaudeAgent...
- Roms/PORTS/: ports. Each is a .sh launcher (shown in the menu) and usually a folder with the game; ports from the
  Miyoo Mini Ports catalog follow that layout.
- Saves/CurrentProfile/saves (battery saves) and states (save states), per emulator subfolder.
- BIOS/, Themes/, Screenshots/, Media/. miyoo/app and miyoo/lib: stock firmware programs and libraries.
- .tmp_update/bin: OnionOS tools (cpuclock, fbmode, keymon, detectKey, sendkeys...); .tmp_update/script: helper
  scripts (stop_audioserver.sh, change_resolution.sh, network/...).

## Things that matter when changing or testing
- FAT filesystem: no symlinks, no permissions; run scripts with sh. Sync (sync) before cutting power.
- Programs started outside MainUI need the same environment: LD_LIBRARY_PATH with /mnt/SDCARD/miyoo/lib and
  /mnt/SDCARD/.tmp_update/lib; audio users stop the audioserver; LD_PRELOAD libpadsp.so for OSS audio.
- CPU clock: OnionOS sets it per emulator/port (cpuclock). The Overclock app and launch scripts may change it.
- Save-state-on-power-off: OnionOS can save and resume the running game at power off/on.
- Before editing system files: say what will change, keep backups (write_file does), and prefer the OnionOS apps
  (Tweaks) for settings the person can change themselves.
