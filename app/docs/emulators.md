# Emulators on OnionOS / RedOnion (Miyoo Mini family)

Check the device before answering: /mnt/SDCARD/Emu/<SYS>/config.json and launch.sh say what is installed and which
core runs. This page is the general picture (OnionOS 4.3 docs + what was found on a RedOnion v4.5 Mini Plus).

## How a system is wired
- Emu/<SYS>/config.json: "label" (name in the menu; leading/trailing spaces or invisible characters change the
  sort order), "launch" (launch.sh), "rompath" (../../Roms/<FOLDER>), "extlist" (accepted extensions, "|" separated;
  ROMs whose extension is not listed do not show). launch.sh picks the RetroArch core or a standalone emulator.
- ROM folders are case sensitive and differ from the stock Miyoo card (PS1 is Roms/PS, NES is Roms/FC, SNES is
  Roms/SFC, Genesis Roms/MD, Master System Roms/MS, TurboGrafx Roms/PCE, Arcade Roms/ARCADE, Ports Roms/PORTS).
  Subfolders work one level deep (Roms/GB/Hacks) and cannot contain an apostrophe.
- After adding ROMs: in the Games tab, highlight the system, SELECT > Refresh all roms (game lists are cached;
  each list also has "~Refresh roms" and "~Filter" entries, .miyoocmd files). Per-game settings such as the chosen
  core live in Roms/<FOLDER>/.game_config/.
- Systems are installed from Apps > Package Manager: "Verified" (Emu, shows in Games), "Expert" (RApp: experimental,
  slower or alternative cores; shows in the Expert tab, hidden until Tweaks > User interface > Show expert mode or the
  Expert shortcut is installed). Catalog on the card: App/PackageManager/data/{Emu,RApp,App}.
- Per-game core: in a game list press Y (GLO, game list options) > Game core, to change the RetroArch core for that
  game (the default is marked).

## Systems found installed on this Mini Plus (RedOnion v4.5) and their core
Arcade (ARCADE, mame2003_plus: MAME 2003-Plus romset), MS-DOS (DOS, dosbox_pure), NES (FC, fceumm), GB / GBC
(gambatte), GBA (GBA, gpsp by default; launch.sh switches a game to mgba automatically once if needed and marks it
with <rom>.auto_core_switched), Genesis and Master System (MD / MS, picodrive), NDS (NDS, DraStic standalone, see
docs drastic), Neo Geo (NEOGEO, fbalpha2012_neogeo, needs BIOS/neogeo.zip), Odyssey2 (o2em), TurboGrafx-16 (PCE,
mednafen_pce_fast), PICO-8 (PICO, fake08), Ports (PORTS: .port shortcuts in Roms/PORTS/Shortcuts), PlayStation (PS,
pcsx_rearmed), Saturn (SATURN, yabadabadoo, slow), ScummVM (SCUMMVM: .scummvm/.target shortcuts), SNES (SFC,
mednafen_supafaust), TIC-80 (TIC).
Available in Package Manager besides: Atari 2600/5200/7800/Lynx, WonderSwan, CPS1/2/3, ColecoVision, Amiga, C64,
VIC-20, Channel F, Vectrex, Intellivision, Mega Duck, MSX, SuperGrafx, PCE-CD, FDS, Game & Watch, Pokemon Mini,
Satellaview, Super Game Boy, Virtual Boy, Neo Geo CD / Pocket, 32X, Sega CD, Game Gear, SG-1000, ZX Spectrum,
Supervision, and many expert cores (Snes9x 2002/2005/2010, Genesis Plus GX, SwanStation, PCSX standalone,
FB Neo, MAME 2000/2003-xtreme, Nestopia, mGBA, VBA-M, PICO-8 standalone, EasyRPG, OpenBOR, PC-98, X68000, ...).

## Which core for speed (Mini / Mini Plus)
- 8/16-bit systems run at full speed with the defaults. SNES: supafaust is the default; Snes9x 2005 is lighter if
  a game is slow. GBA: gpsp is the fast default, mGBA more accurate. PS1: PCSX ReARMed (fast, NEON). Arcade:
  MAME 2003-Plus needs the 0.78-era MAME 2003-Plus romset; FBA 2012 / FB Neo need their own romset versions.
- Saturn, Dreamcast-class and 3D-heavy systems are slow or experimental. NDS through DraStic runs many games at
  full speed with frameskip (heavy 3D games drop frames).
- Overclock RetroArch: RetroArch/cpuclock.txt (4 digits, e.g. 1600) for all cores, or
  Saves/CurrentProfile/config/<core>/cpuclock.txt for one core. Recommended max 1600 on Mini, 1800 on Mini Plus
  (docs say 1700/1900 are the absolute limits, with stability risk). This device: RetroArch/cpuclock.txt = 1200.

## BIOS
- In /mnt/SDCARD/BIOS (root of the card) unless a system says otherwise; names and extensions are case sensitive.
- Found on this card: gba_bios.bin, gb_bios.bin, gbc_bios.bin (optional boot logos), neogeo.zip (Neo Geo),
  32X_G/M/S_BIOS.BIN (32X), Sega CD (bios: eu_mcd1_9210.bin, jp_mcd1_9112.bin, mpr-17933.bin, us/... per region),
  Saturn BIOS (saturn_bios.bin and named variants), PS1 (PSXONPSP660.BIN; scph1001.bin is the other usual one),
  Neo Geo Pocket (neo-po.bin / neo-epo.bin), Odyssey2 o2rom.bin, mame2003-plus/ (samples, artwork), scummvm/,
  prboom.wad, ECWOLF.PK3. Never send or download BIOS files for the person: they must dump/own them.

## Saves, states and settings
- Battery saves (.srm): Saves/CurrentProfile/saves/<CORENAME>/<rom name>.srm (rename .sav to .srm; same name as the
  ROM). States: Saves/CurrentProfile/states/<CORENAME>/. Old stock saves were in RetroArch/.retroarch/saves.
- Onion auto-saves a state when leaving through MENU/GameSwitcher and loads it at the next start (slot -1 is the
  auto slot). To start a game fresh: MENU+SELECT > Settings > Saving > Load State Automatically off, saved as an
  override; or delete its states.
- "Game too slow/fast after an update" or "won't start": delete that game's states in states/<CORENAME>/ (old
  states can be incompatible).
- RetroArch settings: change in the quick menu (MENU+SELECT) and save as overrides (Quick Menu > Overrides: Game,
  Core or Content directory). Never "reset to defaults" inside RetroArch; use Tweaks > Advanced > Reset settings >
  Reset RetroArch main configuration.
- Scaling: Settings > Video > Scaling (Integer scale, Keep aspect ratio), saved as override.
- Cheats: .cht files from libretro-database in RetroArch/.retroarch/cheats/<system>; quick_menu_show_cheats = "true"
  in retroarch.cfg; Quick menu > Cheats > Load Cheat File (Replace).
- Remap: Controls > Port 1 Controls (saved on exit; per game/core in Manage Remap Files). A held fast-forward key
  can be mapped there.
- RetroArch log: Settings > Logging > Log to file, save config, then .tmp_update/logs/retroarch.log.

## In-game shortcuts (RetroArch, OnionOS defaults)
MENU = save and exit to GameSwitcher; hold MENU = save and exit to the menu; MENU MENU = launch the previous game;
MENU+SELECT = RetroArch quick menu; MENU+START = aspect ratio; MENU+L = rewind (if available); MENU+R = fast
forward; MENU+L2 / R2 = load / save state; MENU+LEFT/RIGHT = state slot; MENU+X = FPS display. GB: L2/R2 change the
palette. (MENU actions can be changed in Tweaks > Button shortcuts; RedOnion stores them in .tmp_update/config/
keymap.json.)

## Disc and multi-file games
- PS1: .chd (smallest), .pbp, .cue/.bin; multi-disc games use .m3u playlists (one save per game). Tweaks > Tools
  generates the .m3u files (PS, SEGACD, NEOCD, PCE, PCFX, AMIGA); hide the disc files in .Game_Name or .multi-disc
  folders. .tmp_update/script/cue_gen.sh and m3u_gen.sh exist on the card.
- Manuals: Roms/<FOLDER>/Manuals/<rom name>.pdf (same name, case sensitive) shows "Game manual" in GLO (Y).
- Game names: miyoogamelist.xml in the ROM folder renames entries (.tmp_update/script/miyoogamelist_gen.sh makes
  one, removing extensions, bracketed tags and rankings); then Refresh all roms.

## Standalone emulators and apps that use the screen directly
- DraStic (NDS), PICO-8 standalone, PCSX standalone, ScummVM standalone, OpenBOR... launch their own binary from
  Emu/<SYS>/launch.sh. Broken sound in some standalone apps was fixed by wrapping their launch.sh (FAQ).

## Troubleshooting a game that will not run
1. Right folder (case), extension in extlist, Refresh all roms.
2. BIOS present with the exact name (BIOS folder).
3. Delete its states (incompatible) and try another dump; arcade: the romset version must match the core.
4. Check .tmp_update/logs (retroarch.log with logging on) or use the agent's run_app to see how it ends.
5. Bad SD cards (the stock Miyoo card) corrupt files: suggest a good card.
