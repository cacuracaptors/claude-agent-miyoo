# Miyoo Mini family: hardware

## Models (device_info.model / model_id from /tmp/deviceModel)
| model | id | screen | Wi-Fi | battery | safe max CPU |
|---|---|---|---|---|---|
| Mini v2 / v3 | 283 | 2.8" 640x480 | no | ADC (no %) | 1600 MHz |
| Mini v4 | 283 + /tmp/screen_resolution 752x560 | 2.8" 752x560 | no | ADC | 1600 MHz |
| Mini Plus | 354 | 3.5" 640x480 | yes (RTL8188FU, 8188fu.ko on the SD) | AXP223 PMIC (% and charging) | 1800 MHz |
| Mini Flip | 285 | 752x560, clamshell lid | yes | PMIC | 1600 MHz |
- Never set more than 1800 MHz on the Plus or 1600 MHz on the others. Pick the clock from the model, not a menu.
- The agent needs Wi-Fi, so in practice it runs on the Plus and the Flip.

## SoC: SigmaStar SSD202D
- 2x ARM Cortex-A7 (ARMv7-A, NEON, VFPv4, hard float), ~1.2 GHz stock, raised by OnionOS's cpuclock
  (/mnt/SDCARD/.tmp_update/bin/cpuclock <MHz>) which programs the PLL directly: cpufreq's scaling_cur_freq does
  not show it (device_info and perf_monitor read the PLL).
- 128 MB DDR inside the SoC; Linux sees ~100 MB (the rest is reserved for the media heap and the framebuffer,
  see /proc/cmdline). The game, OnionOS and this agent share it: watch memory with perf_monitor.
- Build flags: -marm -mtune=cortex-a7 -mfpu=neon-vfpv4 -mfloat-abi=hard (arm-linux-gnueabihf toolchain).
- Kernel 4.9.84. Root filesystem: read-only squashfs on internal flash. SD card at /mnt/SDCARD (FAT32/exFAT: no
  permissions, no symlinks, case-insensitive names).
- Vendor SDK (libmi_*.so in /config/lib): MI_SYS, MI_GFX (2D blitter: copy, scale, rotate,
  color-convert), MI_DISP, MI_AO (audio out). Ports use MI_GFX to scale and rotate frames for free.

## Screen
- /dev/fb0, 32 bpp ARGB, 2 or 3 pages (yres_virtual); the visible page changes with FBIOPAN_DISPLAY.
- The panel is mounted upside down: the framebuffer is rotated 180 degrees. screenshot / screen_capture already
  turn the picture upright.
- 752x560 models: some programs need the framebuffer mode set to 752x560 before they start (DraStic does).
  OnionOS can show 640x480 programs scaled or centered on them.
- Refresh about 59.7 Hz on the Mini Plus: measure it, do not assume 60.

## Buttons
- One input device, /dev/input/event0, driver gpio-keys-polled: it re-reads every button every 10 ms
  (/sys/class/input/event0/device/poll). Key codes: UP/DOWN/LEFT/RIGHT = KEY_UP..., A = KEY_SPACE,
  B = KEY_LEFTCTRL, X = KEY_LEFTSHIFT, Y = KEY_LEFTALT, L1 = KEY_E, R1 = KEY_T, L2 = KEY_TAB, R2 = KEY_BACKSPACE,
  SELECT = KEY_RIGHTCTRL, START = KEY_ENTER, MENU = KEY_ESC, POWER = KEY_POWER, volume = KEY_VOLUMEUP/DOWN.
- keymon (OnionOS) and running programs all read event0 at the same time.

## Audio
- MI_AO (vendor) under an OnionOS audioserver; programs that open the audio device directly stop it first
  (/mnt/SDCARD/.tmp_update/script/stop_audioserver.sh). libpadsp.so (LD_PRELOAD) gives an OSS /dev/dsp emulation.

## Power and clock
- Plus/Flip: AXP PMIC (battery %, charging). /dev/rtc0 keeps the time while off on the Plus.
- OnionOS can save the game and resume it after power off (on supported emulators and ports).
