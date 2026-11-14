# Box art, screenshots and game lists (scraping)

## Where images go
- Roms/<FOLDER>/Imgs/<rom file name without extension>.png. The menu shows it next to the game. Ports: Roms/PORTS/Imgs/
  <shortcut name>.png. Small PNGs load faster and take less space; very large images slow the lists.
- Favorites and recents keep their own thumbnails; if they go missing or duplicate: Tweaks > Tools > Favorites: Fix
  thumbnails and duplicates.

## Built-in scraper (Wi-Fi models: Mini Plus, Flip)
- In a game list press Y (game list options) > Scraper. It can scrape one game or the whole list, and can run in the
  background ("Toggle background scraping").
- Scripts and settings: .tmp_update/script/scraper/ (menu.sh, scrap_screenscraper.sh, scrap_launchbox.sh,
  scrap_retroarch.sh), settings in .tmp_update/config/scraper.json.
- Sources: ScreenScraper (SS, best coverage, matches by checksum/name; a free screenscraper.fr account in
  "Screenscraper: account settings" gives more threads/quota and fewer failures), LaunchBox (LB), RetroArch
  thumbnails (RA, libretro-thumbnails, matched by the No-Intro name: works best with No-Intro/Redump file names).
  Order/enable them in "Scraping sources".
- Media types: Box Art (SS, LB, RA), Title screenshot (SS, LB, RA), In-game screenshot (SS, LB, RA), 3D box (SS, LB),
  Wheel (SS, LB), Marquee (SS, LB), ScreenScraper Mix V1/V2 (SS). Defaults: SS box-2D, LB "Box - Front",
  RA Named_Boxarts. Region preference (us, eu, jp, br...) in "Region selection". scraper.json keys:
  screenscraper_username/password, Retroarch/Screenscraper/Launchbox_enabled, ScrapeInBackground,
  *MediaType, LaunchboxRegion, ScreenscraperRegion. Never write the person's password anywhere yourself.
- Needs Wi-Fi and the correct date (TLS): if it fails everywhere, check the clock and the connection.

## Doing it on a PC (Mini without Wi-Fi, or big collections)
- Skraper (Windows/Linux, uses ScreenScraper) or any other scraper works if it writes PNGs into Roms/<FOLDER>/Imgs
  named exactly like the ROM (without its extension); configure the output folder and size in the tool.
- The agent can also do it on the device: fetch images with download_file (e.g. libretro-thumbnails raw URLs, the
  name must match exactly, spaces escaped) and save them as Roms/<FOLDER>/Imgs/<name>.png; check one with view_image.

## Names and lists
- miyoogamelist.xml in a ROM folder sets display names (generator: .tmp_update/script/miyoogamelist_gen.sh). Then
  Refresh all roms. If the list stops loading, check "extlist" in Emu/<SYS>/config.json.
- Hide things: files/folders starting with "." are not listed.
