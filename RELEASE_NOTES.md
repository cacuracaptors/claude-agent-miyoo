## v1.0.3 - Exact replays in DraStic + built-in notes for Claude

### New
- Frame-exact replays in DraStic (Nintendo DS). `run_app` can put a save state in DraStic's auto-load slot before each run (`drastic_state`) and use DraStic's own input recording (`drastic_input_record`, `drastic_input_playback`), which applies every button change on the same emulated frame: a recorded play repeats exactly, for benchmarks and bug reports.
- Built-in notes for Claude (`docs`): how to use the app's tools well, the Miyoo hardware, how OnionOS and RedOnion work and where things are, DraStic, and making and testing ports. Claude reads them before technical work instead of rediscovering the device in every chat.

### Improvements
- Stopping a test (time limit or cancel) now lets the game save and quit: the game gets the signal first and has up to 10 s, and its launch script finishes its own cleanup.
- `job_status` no longer shows finished jobs as still running.

### Fixes
- The Mini Flip's screen was described as 640x480; it is 752x560.

### Updating
- From v1.0.0 to v1.0.2: use *Settings > Check for updates* in the app.
- Or download `ClaudeAgent-v1.0.3.zip` below and extract it to the root of the SD card.
- Your settings, connection and backups are kept.
- The new `docs` tool appears in new Claude chats.
- If you run your own relay, publish it again (`bash server/deploy.sh`).
