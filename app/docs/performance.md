# Performance on the Miyoo Mini: what works, what does not, with numbers

Lessons measured on a Mini Plus in real ports (Tomb Raider 1/2 on OpenLara, Mario Kart 64, Zelda Ocarina of Time,
Hexen II, Fallout 1/2, 3D Pinball, DraStic) and in mr-finster's miyoo64 N64 ports. Use it as the starting point
for any performance, smoothness or battery work; then measure on the device, because every game is different.

## The rules that paid off most
1. Measure before changing anything, and after every change. Several "obvious" ideas gained nothing or lost
   (see "Tried and not worth it"); most gains came from places only measuring showed.
2. Keep a per-frame performance line in the log every 1-2 s, in ms: game logic, building the draw list,
   rasterizing on each core, waiting between cores, idle time (the headroom), plus CPU clock and governor.
   Use clock_gettime (microseconds); SDL_GetTicks (1 ms) is too coarse.
3. Check the image bit for bit when an optimization should not change it (dump frames or capture a frame and
   replay it on the PC; "0 pixels different"). Any change that alters the image needs the developer's approval.
4. Find the worst case (heaviest scene) and optimize for it; averages hide the stutters players feel.
5. Remove every measurement and diagnostic from the release build: counters inside pixel loops cost real time
   (one cost a read and two writes per pixel). Mark temporary code with a unique comment to find it later.
6. The profile report must come from the same binary that produced the profile.

## How to measure
- Per thread CPU: /proc/self/task/<tid>/stat fields 14+15 (utime+stime) every 2 s; name threads with
  pthread_setname_np. This found audio reverb at 8-10% of a core and a screen copy at 5-8%.
- Whole system from outside: the agent's perf_monitor (CPU %, real PLL clock, memory, one process).
- Sampling profiler from another process (ptrace, ~250 Hz) when the program cannot take signals: DraStic crashes
  with LD_PRELOAD SIGPROF profilers (it uses SIGSEGV handlers and sigaltstack).
- Off-device: qemu-arm with a per-instruction counting plugin gives per-function / per-line cost of an ARM build;
  measure with the Miyoo GCC 8.3 build, not a newer GCC (GCC 13 scalarized NEON compares and misled).
- Cost model by regression over several scenes (time = a*pixels + b*rows + c*triangles). OpenLara: 44 ns per
  pixel, 310 ns per triangle row, 0.8 us per triangle.
- Read the assembly of the hottest loop (-S with the same flags): look for registers spilled to the stack each
  pixel and forgotten counters.
- Crashes: keep the unstripped binary of the exact build; addr2line -f -C -i -e <binary> <addresses>.
- Visual glitches that happen rarely: keep a ring of the last ~8 frames' summaries (average brightness of 64
  points, count of each command type) and print it when the event happens.
- Repeatable runs: same save state + same input (see docs agent: input replays; DraStic has frame-exact ones).

## The hardware facts that drive everything
- 2x Cortex-A7, in-order, NEON. IPC around 0.5 in rasterizers; L2 misses ~10% of the time; NEON latency
  dominates. float->int conversions and moving values from the FPU to core registers cost several cycles: reuse
  results, prefer fixed point in inner loops, keep few live variables (spills).
- Clock: stock 1200 MHz. Overclock with OnionOS's cpuclock: up to 1800 on the Mini Plus, never above 1600 on
  Mini v2-v4 and Flip (they become unstable). Choose it from /tmp/deviceModel (354 = Plus), not from a menu.
  Many ports reach their goal at 1200-1600; overclock is the last lever, after real optimizations.
- Sleep granularity is 10 ms: usleep/SDL_Delay wake on the next 10 ms tick. A naive 60 fps limiter
  (SDL_Delay(16 - spent)) locks the game at 50 fps (every frame ends at 20 ms).
- Screen: the panel refresh is ~59.7 Hz on the Mini Plus (measured 59.70; 16.75 ms between vsyncs). Measure it.
- Framebuffer memory is write-combining: writing it is almost 2x faster than normal RAM (4.55 vs 8.24 ms for 4
  full screens), reading it is ~30x slower (7.73 ms vs 0.25 ms for 256 KB). Never read it back in a loop.
- About 100 MB of RAM for everything; the media (MMA) heap for MI_GFX is small (mma_heap in /proc/cmdline, ~21 MB;
  MK64 using ~8 MB left ~9 MB free) and can be taken by another program (seen once on RedOnion until reboot).
- SD card (FAT): seeks and many small reads are slow, especially cold; file listing is slow.

## Frame pacing and vsync
- Page flipping is the biggest felt improvement: the fps number did not change but removing tearing felt ~30%
  smoother. Draw into the hidden page, then FBIOPAN_DISPLAY, which returns at vsync. Needs yres_virtual >= 2
  pages; measure FBIO_WAITFORVSYNC / FBIOPAN_DISPLAY first (~16.7 ms each).
- Do the copy/flip in its own thread, so the game already draws the next frame while it waits; a late frame then
  shows at the next vsync instead of dropping to 30.
- With vsync, the panel is the clock: no fps limiter, no busy waiting. The main thread waits only when a frame
  is already queued.
- Without vsync, the limiter that works: wait from the end of the previous frame; usleep(1000) only while more than
  ~11 ms remain, then sched_yield until the deadline; a late frame does not wait at all. Never follow a fixed 60 Hz
  grid ("slots"): a 22 ms frame misses its slot and waits for the next (33 ms), dropping heavy scenes to 30-43.
- Hexen II: its own throttle (sys_throttle) on top of page-flip vsync cost frames: sys_throttle 0 gave a locked 60.
- Steady pacing beats a higher average: pinning threads to cores (pthread_setaffinity_np) removed frame-to-frame
  variation in OoT and the developer noticed more fluidity (OoT also runs with a real-time priority option).
- Fixed-step games at 20 or 30 fps logic (N64): interpolate extra frames instead of running logic faster.
  OoT and miyoo64: logic at 20 fps, picture at 30 (59.7/2) by drawing the 1/3 and 2/3 points between game frames
  with interpolated matrices (Ship of Harkinian method: record the matrix tree per actor/camera/effect each game
  frame, replay the display list with interpolated Mtx). Fall back to 20 by itself when a draw gets too slow
  (average > ~23-30 ms). Watch for things written after the frame was recorded (the camera matrix rewritten at
  the end of Play_Draw caused visible back-and-forth) and for CPU-built vertices (skinned models) that matrices
  do not reach.
- Game speed must follow the vblanks a frame really took (MK64: the race steps by the vblanks elapsed), or the
  game runs slow when frames are late.

## Using both cores
- Pipelining works: core 0 runs logic and records draw commands for frame N+1 while core 1 rasterizes frame N
  (MK64, OpenLara). Then core 0 rasterizes its share of N.
- Split the screen in many interleaved row bands (OpenLara: 64 groups, rebalanced every frame from the previous
  frame's times) rather than top/bottom halves: detail is uneven across the screen.
- Pin threads to cores. Name them. Dividing work makes frames faster but does not save battery (same total work).
- When logic is tiny (OoT ~1 ms of 20+), running drawing in parallel with logic does not pay; split the drawing
  itself. A core waiting for the other (spin) showed as 6-20% in profiles: balance the split.
- Holding back triangles to sort them can stall the second core: sorting opaques near-to-far helped OpenLara and
  MK64 but made miyoo64 slower (+0.8 ms), because its streaming second core sat idle. Measure per renderer.

## The 2D engine (MI_GFX) and the screen
- MI_GFX (libmi_gfx + libmi_sys, headers in the Miyoo toolchain) copies, color-converts, scales and rotates 180
  degrees for free from MMA memory (MI_SYS_MMA_Alloc + MI_SYS_Mmap, MI_SYS_FlushInvCache before the blit,
  MI_GFX_BitBlit, MI_GFX_WaitAllDone). MK64: RGB565 frame -> 32-bit, rotated and scaled to the panel, saved
  ~2.9 ms per frame of a core. Load it with dlopen, run a small self-test blit at start (orientation and colors),
  and fall back to the CPU copy if anything fails; free the MMA at exit.
- It reads RGBA5551 with the N64 bit layout (R 15-11 ... A 0) in CPU byte order: N64-style renderers can hand it
  their frames directly, without a conversion pass (OoT).
- Good uses: the final frame (scale/rotate/convert), a HUD or text layer drawn once at 320x240 and composed over a
  640x480 3D frame, pre-rendered still backgrounds. Text boxes as one layer rebuilt only when they change (OoT).
- It cannot antialias a 1:1 frame, and it did not help where the profile showed the screen copy was already
  negligible (miyoo64 at 1600 MHz): check the profile first.
- MMA is scarce: try smaller sizes before giving up, log how much was free (grab 1 MB chunks until refused, give
  them back), and keep a CPU path.
- Drawing straight into the framebuffer (3-page rotation: one shown, one waiting for vsync, one being drawn)
  removed the copy and sped up painting (OpenLara: 46.7 -> 52-56 fps in its worst scene). Decide at start by
  timing writes. Effects that read the screen (shadows darkening pixels, menu backgrounds) must read a copy in
  normal RAM instead.
- Present only when something changed: Pinball and Fallout redraw/present only on change (Fallout forces one
  present every 250 ms and always during videos); saves battery and CPU in menus and turn-based games. Mind the
  regions that change: Pinball's "only changed areas" left a ball trail until the previous position was included.
  SDL partial texture updates are ignored by the Miyoo SDL driver: present the whole texture.
- Put the screen in a known mode at start (docs ports: framebuffer states): with a single page, page flipping
  silently turns off and tearing returns.

## Resolution
- Native 640x480 is reachable for software 3D: OpenLara TR1 and Hexen II at 60 fps, MK64 at 60 with dynamic
  resolution, OoT at 30 with dynamic resolution, all on the Plus.
- Dynamic resolution in steps keeps the frame rate and is barely visible: MK64 drops to 448x336 at most for an
  instant; OoT steps of 32 lines, target 28 ms for a 33 ms frame, goes up only when the step above fits with
  margin (90% of the target) and waits a few frames between changes to avoid oscillating.
- Keep the UI/HUD at 320x240 scaled by MI_GFX while the 3D is 640x480 (Hexen II -ui320, OoT HUD/text layer):
  readable UI, cheaper, and it does not force the 3D down.
- Lowering the whole render resolution is the last resort, after the real optimizations.

## Software rasterizers: what gave results (image identical unless noted)
- Cache what does not change: menu/title backgrounds cached per line with a key (image + version + params):
  6 -> 2.4 ms per frame. Drop repeated full-screen copies in the same frame (a video asked for 4): 21 -> 5 ms.
- Draw each room/portal once on the union of portal rectangles, skip portals beyond the fog distance, sort rooms
  and opaque faces near to far (OpenLara TR2 Offshore Rig: 72 -> 17 rooms, 25 -> 38 fps).
- Skip spans already fully covered: test the depth of the span ends before perspective (1/w is linear, the
  nearest point is an end): 130-160k pixels skipped per frame.
- Draw the sky last, with infinite depth, only where nothing was drawn (OpenLara TR2; miyoo64 skydefer).
- Inner pixel loop: walking pointers instead of indices, no counters, light constant per 16-pixel span,
  reuse the span start (it is the previous end): -11 to -17% on triangles.
- Adaptive perspective: 32-pixel spans when 1/w varies little (error < 0.13 texel): -6 to -9% per pixel.
- Depth: 16 bits with an exponent (4+12) computed only at span ends: precise far away for almost no cost;
  32-bit depth fixed z-fighting too but cost ~10% per pixel.
- Texture filters: N64 3-point filtering is cheaper than bilinear; precompute it with a quarter-texel table and
  sample exactly only for strong magnification (MK64). A pre-expanded 2x2-cells-per-texel texture gives a filtered
  look with one read per pixel (OoT "filtro_rapido"). Filter only where it shows (close surfaces, the road).
- Padded textures (1-texel border, mirrors unfolded) remove wrap/clamp work from the sampler; NEON bilinear in asm;
  NEON texture hashing; process spans in blocks of 64 pixels (OoT: worst frame 49 -> 37 ms).
- Expensive translucent things: half resolution only for heavy translucent objects (water, waterfall) or a
  pre-rendered filtered version; opaque item boxes were acceptable to the developer for speed. A big translucent
  halo with depth test (Navi near the camera) cost ~17% of a frame and could not go to MI_GFX.
- Check what the engine already disables in software mode before "fixing" GPU features (OpenLara forced water,
  light and shadows to minimum already).

## Tried and not worth it (do not repeat without a new reason)
- Unrolling the pixel loop x2: the pixel got ~15% slower (32 template variants overflowed the instruction cache).
- Textures in 8x8 tiles: only 2-3%, and the new addressing caused a rare crash.
- Joining room copies "only when it pays": slower than always joining.
- Reading the texture once per pixel pair when magnified ("halfmag", miyoo64 at 320x240): no gain.
- Nearest filtering instead of bilinear in miyoo64: not faster (the cost was elsewhere), so a precomputed filter
  was not worth writing there.
- Sorting opaques near to far in miyoo64: slower (see Using both cores).
- MI_GFX for the final copy when the profile showed it at ~0%.
- Overclock before optimizing: hides problems and costs battery; OpenLara gained a lot at 1500 MHz but after its
  optimizations did not need it.

## Audio
- Effects can cost more than expected: OpenLara's reverb (not in the original game) ran per sample even in
  silence, 4-5% of a core; removing it gave ~2 fps. N64 software synthesis is affordable when mixed in C with the
  audio microcode commands (MK64), output through MI_AO.
- Mix 32 samples at a time, never read past the buffer (Fallout mixer).

## Loading times and files (Fallout 2)
- Slow boot and loads came from many small reads and FAT seeks: cache each archive's directory table in a sidecar
  index file, read text files in one bulk read instead of per character, fix O(n) "binary searches", reuse file
  handles, cache directory listings, add a per-archive miss filter (bloom filter) for files that are not there.
- Raise the CPU governor to performance only while loading/saving, then restore it.

## Battery
- Do less work: caches, skipping repeated work, presenting only on change. Using two cores does not save energy.
- Vsync removes busy waiting for free. A 30 fps mode or lower clocks in light scenes are options; ask the
  developer, many prefer frame rate over battery.
- PerformanceMode=1 in an OnionOS .port shortcut sets the governor to performance (fixed max clock).

## Linux/ARM pitfalls seen in ports
- rand() goes to 2147483647 in glibc, not 32767 as on Windows: rand() < N written for Windows almost never hits
  (Tomb Raider enemies always fled). Use (rand() & 0x7FFF) < N. Search for "rand() <", "rand() >", RAND_MAX.
- x >> 4 is not x / 16 for negatives (floor vs toward zero): (d + ((d >> 31) & 15)) >> 4.
- Little-endian: N64/big-endian structs read with different field widths than the macro that wrote them break
  (OoT cutscene music commands asked for song 0xFFFF).
- Reading past a buffer at the end of an ELF segment hits unmapped holes between PT_LOAD segments (OoT texture
  loads): find holes with dl_iterate_phdr and read zeros there.
- #include of standard headers inside a namespace breaks std (Fallout).
- Signals: some programs save from their SIGTERM handler and crash if it lands mid-work (DraStic): see docs drastic.
- The same effect may go through different code paths in different game states (menu background in game vs title
  screen): fix and measure each path.
- Common release flags: -O3 -ffast-math -marm -mtune=cortex-a7 -mfpu=neon-vfpv4 -march=armv7ve+simd
  -fno-unroll-loops (OpenLara). The Miyoo toolchain is GCC 8.3 (arm-linux-gnueabihf); never the distro's ARM GCC.

## DraStic (closed source) on the Mini Plus, Mario Kart DS, 1800 MHz
- Full speed with auto frameskip: 20-45 drawn fps. Main thread ~78% of samples: 33.5% sleeping in its frame sync
  (SDL_Delay), 21.5% 3D rasterization, 18% emulated CPU (11% JIT), 8.4% 3D geometry, 4.4% 2D, 1.6% sound.
  The steward-fu video thread ~19% (screen copies). threaded_3d=1 gained little and broke a camera scene.
- Ideas being explored: split the 3D rasterization (bins) across both cores using its DWARF info, move screen
  work to MI_GFX. Its emulation speed is not constant (~47 fps in the first second after loading).

## Checklist for a port or app that is slow
1. Per-frame log line (logic, draw, wait, idle, clock) and per-thread CPU. Find the worst scene.
2. Stuck at 50 fps with idle time left? The 10 ms sleep: use vsync or the limiter above.
3. Tearing or uneven pacing? Page-flip vsync in its own thread; pin threads; check the framebuffer mode at start.
4. Repeated work per frame: still backgrounds, copies that overwrite each other, effects nobody sees.
5. Split the heaviest stage across both cores; balance with measured times.
6. Hot loop: assembly, spills, float->int, counters; NEON where data is regular.
7. MI_GFX for scale/rotate/convert and HUD layers, if the profile shows the CPU doing it.
8. Dynamic resolution steps and a 320x240 HUD before lowering the whole resolution.
9. Audio and file loading: they show up in per-thread CPU and boot logs.
10. Only then overclock, per model (1800 Plus / 1600 others).
11. Remove all diagnostics and measure the release build again.
