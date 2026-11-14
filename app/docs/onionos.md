# OnionOS and RedOnion: how the software works

OnionOS (github.com/OnionUI/Onion) is not a firmware: the stock firmware boots and runs Onion's scripts from the SD
card, so removing the card's .tmp_update returns to stock. RedOnion is a fork (github.com/Amiga500/Onion; its
repository also calls the line OnionPlus) with the same layout, extra performance work, Mini Flip support (device
285), the Open MainUI menu, fbmode (framebuffer modes on 752x560 devices), AdvanceMENU as an alternative frontend and
its own OTA updater (app "RedOnion OTA update"). device_info.os says which one runs; version in
.tmp_update/onionVersion/version.txt, project in website.txt. Check the real files before changing anything.
Other topics: emulators (systems, cores, BIOS, saves, RetroArch), scraping (box art), ports, performance.

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

## Shortcuts (OnionOS defaults; changeable in Tweaks > Button shortcuts)
- Main menu: MENU = GameSwitcher, MENU MENU = resume the last game, SELECT or hold MENU = context menu (refresh,
  search, Tweaks, shut down). X / Y in the menu launch an app or GLO (RedOnion: .tmp_update/config/keymap.json,
  e.g. "mainui_button_x": "app:Search", "mainui_button_y": "glo").
- Everywhere: MENU+POWER = screenshot; hold MENU+B 2 s = blue light filter; hold MENU+A 2 s = screen recorder;
  brightness: Mini SELECT+L2/R2, Mini Plus MENU+VOL; START+L2/R2 = volume boost (START+L2+R2 resets); Mini Plus
  VOL+ and VOL- together = mute; POWER = sleep; hold POWER = save and turn off; hold POWER 5 s = force shutdown;
  Mini Plus hold POWER 10 s = cut the power (only for a frozen device).
- In games: see docs emulators. GameSwitcher: LEFT/RIGHT browse recent games with a screenshot of where they were,
  A resumes, Y toggles the minimal view (hold for full-screen image), B back.

## Apps (Apps tab; more in Package Manager)
Tweaks (system settings, network, button shortcuts, user interface, tools, advanced resets), Package Manager
(install/remove emulators and apps), Search (find games), Play Activity / Activity Tracker (time played), Clock
(set the emulated date/time; the RTC stops when off on models without a clock battery), ThemeSwitcher, Battery
Monitor, Gallery (screenshots), File Explorer (DinguxCommander), Music Player (GMU), Video Player (FFplay), Ebook and
PDF readers, Terminal, Guest Mode (a separate profile), Random Game, List shortcuts, Quick Guide / Onion Manual,
OTA update. Startup: Tweaks > System > Startup (auto-resume last game, start tab/app, add hours to the RTC: Pokemon
day/night).

## Wi-Fi features (Mini Plus, Flip)
- Tweaks > Network: SSH, Telnet, FTP (bftpd), HTTP file server (filebrowser: transfer files from a browser), Samba
  (Windows share), NTP time sync, hotspot, WPS (for networks with special characters in the name/password, which the
  menu cannot type), Wi-Fi power saving. "Disable services in game" stops them while playing (battery, speed).
- RetroArch netplay (GLO > Netplay) for supported cores.
- Configs in .tmp_update/config/ (smb.conf, bftpd.conf, hostapd.conf, filebrowser/...).

## Battery and charging
- Percentage comes from voltage (Plus: PMIC) and fluctuates. Save power: brightness under ~70%, Wi-Fi off,
  headphones. Mini v2 charges from USB-A to USB-C (5 V 1 A); Plus and v4 accept limited USB-C to USB-C.

## Recovering a device
- Black screen boot loop: hold POWER 10 s (until the second rumble); if it keeps happening, delete
  .tmp_update/cmd_to_run.sh from a PC (it relaunches the last game at boot).
- Lost play activity or favorites: Tweaks > Tools fixes; Saves/CurrentProfile/saves/playActivity.db (backups in
  PlayActivityBackup/).
- Settings: Tweaks > Advanced > Reset settings (RetroArch main config, MainUI, ...). Never reset inside RetroArch.

## Things that matter when changing or testing
- FAT filesystem: no symlinks, no permissions; run scripts with sh. Sync (sync) before cutting power.
- Programs started outside MainUI need the same environment: LD_LIBRARY_PATH with /mnt/SDCARD/miyoo/lib and
  /mnt/SDCARD/.tmp_update/lib; audio users stop the audioserver; LD_PRELOAD libpadsp.so for OSS audio.
- CPU clock: OnionOS sets it per emulator/port (cpuclock). The Overclock app and launch scripts may change it.
- Save-state-on-power-off: OnionOS can save and resume the running game at power off/on.
- Before editing system files: say what will change, keep backups (write_file does), and prefer the OnionOS apps
  (Tweaks) for settings the person can change themselves.
