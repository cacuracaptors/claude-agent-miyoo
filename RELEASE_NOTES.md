## v1.0.4 - Claude's notes: performance, emulators, box art

### New
- A new topic in Claude's built-in notes, `performance`: what made ports faster and smoother on the Miyoo Mini, measured in real ports (Tomb Raider, Mario Kart 64, Ocarina of Time, Hexen II, Fallout, Pinball, DraStic): how to measure, frame pacing and vsync, using both cores, the 2D engine, dynamic resolution, rasterizer techniques with their gains, what did not work, loading times, battery, and a checklist.
- New topics `emulators` (systems and their folders, cores, BIOS, saves and states, RetroArch settings, shortcuts, multi-disc games, troubleshooting) and `scraping` (box art with the built-in scraper or a PC, game names), and the `onionos` notes now cover RedOnion, the shortcuts, the apps, the Wi-Fi features, battery and how to recover a device. So Claude can help players, not only developers.
- The `ports` notes now also explain how a port should fit into OnionOS and RedOnion: the screen mode at start, saving at power off, the GameSwitcher, the system overlays and the clock for each model.

### Improvements
- Claude is told about its notes as soon as it connects to the Miyoo, and to read the performance notes before any speed, smoothness or battery work.

### Updating
- Use *Settings > Check for updates* in the app, or download `ClaudeAgent-v1.0.4.zip` below and extract it to the root of the SD card.
- Your settings, connection and backups are kept.
- If you run your own relay, publish it again (`bash server/deploy.sh`).
