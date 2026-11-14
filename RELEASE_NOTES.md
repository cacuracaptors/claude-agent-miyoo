Buttons that games see, steadier connection, and a status line.

- **Simulated buttons now work in games and emulators.** The Miyoo's button driver re-reads the real buttons every 10 ms, so a button pressed by Claude was let go almost at once: menus noticed it, but games (DraStic, the GameSwitcher) did not, and a held button never stayed held. While Claude presses buttons (`press_buttons`, `input_play`) that re-reading now pauses, and it comes back right after, even if the app is closed in the middle. Your own buttons do nothing during that time.
- **`input_play` and `input_record` can start counting from a line of the game's output** (`start_on`), for example the line an emulator writes when a save state has loaded. Loading times vary, so this keeps every button press at the same moment of the game, run after run.
- **`run_app` reports crashes of the game itself.** Launch scripts often end normally even when the game they started crashed; the crash (which program, which signal) and out-of-memory kills are now reported.
- **A short line next to *Activity* shows what Claude is doing** when it works for a while without touching the Miyoo (`set_status`), so the app does not look stuck.
- **Short Wi-Fi drops no longer break Claude's work.** The relay waits up to 20 s for the Miyoo to reconnect instead of failing at once.
- **No more "Bad Gateway" on long tests.** Long tools answer within 50 s and go on as jobs; an answer that takes longer is kept by the relay until Claude asks for it (`wait_result`).
- If a game was left in the GameSwitcher (MENU pressed during a test), `run_app` now says so, instead of only saying the app could not reopen.

**Install:** download `ClaudeAgent-v1.0.2.zip` below and extract it to the root of the SD card, or, if you already have Claude Agent, update from the app: *Settings > Check for updates*.

The relay changed too: if you run your own, publish it again (`bash server/deploy.sh`).
