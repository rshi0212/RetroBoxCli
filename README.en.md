# retrobox

A ScreenScraper + RetroAchievements scraper CLI for Batocera: it writes
`roms/<platform>/gamelist.xml` plus media, and tags every game with its
RetroAchievements id and hash so EmulationStation shows achievements right
away.

One static binary (zlib is compiled in; the release build is a fully static
musl executable). The prebuilt binaries are free to use; the source code is
not public at this time. The detailed guides under `docs/` are in Chinese;
this page covers what you need to get going.

## Install

The Linux x86_64 release is a single static binary; download it into a
directory on your `PATH`:

```bash
# current user (~/.local/bin)
mkdir -p ~/.local/bin
curl -L https://github.com/rshi0212/RetroBoxCli/releases/latest/download/retrobox -o ~/.local/bin/retrobox
chmod a+rx ~/.local/bin/retrobox

# or all users (/usr/local/bin)
sudo curl -L https://github.com/rshi0212/RetroBoxCli/releases/latest/download/retrobox -o /usr/local/bin/retrobox
sudo chmod a+rx /usr/local/bin/retrobox
```

Update in place:

```bash
retrobox -U                    # sudo retrobox -U when installed in /usr/local/bin
retrobox --update-to v2026.10.01-7a20a85   # install a specific release (may be older)
```

`-U` downloads the release's `SHA256SUMS`, verifies the new binary against
it and atomically replaces the running executable; it never downgrades.
Binaries installed by `cargo install` or built inside a source tree are
not replaced — the command prints how to update them instead. Only Linux
x86_64 binaries are available at this time. Details (wget, verification,
uninstalling) are in [docs/installation.md](docs/installation.md) (Chinese).

## Two scrape modes

| Mode | Source | Use it when |
|---|---|---|
| `retrobox scrape dat` | DAT files (No-Intro P/C XML, Redump XML, ClrMamePro, MAME listxml) | ROMs are not local yet, or you pre-build gamelists for a large set |
| `retrobox scrape roms` | ROM files on disk | loose collections without DATs; hash match, then title search |

Both modes **merge** into an existing `gamelist.xml` by `<path>`: only the
fields retrobox writes are replaced; favorites, play counts, hand-edited
fields, unknown tags and entries it did not process are kept. Writes are
atomic; a foreign gamelist is backed up once as `gamelist.xml.bak`, and an
unparsable one is moved aside as `gamelist.xml.corrupt`.

Games ScreenScraper does not find are recorded too: named after the RA
title, the DAT name or the file, with RetroAchievements data when RA has
some, and without an `ssid` (so `--only-new` asks again next time). Into an
existing entry they only add missing fields, never overwrite what an
earlier run or the user wrote. `--report` lists them as `missing`.

## Quick start

```bash
cat > .env << 'EOF'
SS_DEVID=your_dev_id          # required (ScreenScraper developer account)
SS_DEVPASSWORD=your_dev_password
SS_SSID=your_member_name      # optional, sets your quota level
SS_SSPASSWORD=your_member_password
RA_API_KEY=your_ra_web_api_key  # optional, needed for --ra-fill only
EOF

retrobox doctor                                   # environment self-check
# DATs: download them yourself and put them into ~/.config/retrobox/dats/
retrobox scrape dat --roms-root /userdata/roms --media box-2D,ss --resume
retrobox scrape roms --roms-root ./roms --ra --report report.json
```

Credentials are never taken from the command line (shell history, `ps`).

## Commands

| Command | Purpose |
|---|---|
| `auth check` / `auth show` | validate credentials / show their source (masked) |
| `quota [--json] [--watch N]` | live account quotas (does not use daily quota) |
| `systems list` / `map <dir\|id>` | Batocera dir ↔ ScreenScraper system id (an id maps to the platform's canonical dir) |
| `dats scan` / `info` | inspect DAT files |
| `scrape dat` / `scrape roms` | the two scrape modes |
| `hash --system <dir> [--explain] <files…>` | crc32/md5/sha1 and the RetroAchievements hash of any file; `--explain` shows how the RA hash was computed |
| `doctor`, `cache stats/clear`, `completion bash\|zsh\|fish` | utilities |
| `-U` / `--update-to <TAG>` | update to the latest / a specific release |

Every command has `-h` (short) and `--help` (full, with examples).

## Main scrape options

```
--media <LIST>        box-2D,box-2D-back,ss,ss-title,wheel,wheel-hd,marquee,
                      screenmarquee,video,manuel,fanart | none | all
--fields <LIST>       name,desc,year,publisher,developer,players,rating,genre
--region-priority     default us,eu,jp,wor
--ra                  tag games that have achievements (cheevosId/cheevosHash)
--ra-only             like --ra, and scrape only games that have achievements
--ra-fill             like --ra, and fill metadata and art from RA (needs RA_API_KEY)
--resume              skip entries done in the previous run
--only-new            skip games already in gamelist.xml with a ScreenScraper id
--report <FILE>       JSON report: unmatched, failed, title-search matches to
                      review, games lacking requested media
--threads <N>         cap concurrency (default: your account's maxthreads)
--dry-run             no API calls, no files written
scrape dat:  --dat-dir/--dat-file, --no-clones, --offline
scrape roms: --rename (canonical names, TorrentZip repack), --hash-only (no title search)
```

ROM mode scans sub-directories, hashes files in parallel and caches the
hashes (path + size + mtime), so re-runs do not read unchanged files. It
scans each platform's Batocera extensions; folders named with one of them
(`Game.ps3/`, `Game.daphne/`) are folder games, and files a sheet names
(cue/toc tracks, gdi, m3u discs, CloneCD `.img`/`.sub`, `.mdf`) are not
separate games. Archives retrobox cannot open (`.7z`, `.rar`, …) are
scraped by name but get no `cheevosHash` — never the archive's own, wrong
hash.

## ScreenScraper quotas

Quotas come from each response's `ssuser` block and are applied live:
threads, requests per minute, requests per day (reset at midnight CET), and
a separate daily quota for *unrecognized* ROMs (431 — hack/translation sets
hit it quickly; use `--hash-only`). Every media download costs one request.
On 430/431/401/403/423/426 retrobox stops, merges what it has, saves resume
state and exits with code 3; `--resume` continues later.

## RetroAchievements

| Option | Effect | RA account |
|---|---|---|
| `--ra` | writes `cheevosId` (RA game id) and `cheevosHash` for games that have achievements, so the frontend shows them | not needed |
| `--ra-only` | like `--ra`, and scrapes only games that have achievements | not needed |
| `--ra-fill` | like `--ra`, and fills publisher/developer/genre/year and box art / title screens from RA where ScreenScraper has none | `RA_API_KEY` |

A file matches when its RA hash is in RA's hash library **and** the game
has achievements: RA also registers many games without any (SNES: 2831
registered, 1184 with achievements), which get no `cheevosId`. The data
sources are chosen automatically and give the same result (public hash
library, cached for 24 h; the RA Web API when the library is unavailable
and a key is set). Only `--ra-fill` requires `RA_API_KEY` (RA control
panel → Web API Key); requests use only the key, `RA_USERNAME` is optional
and the RA password is never needed. `--ra-v1` from earlier versions still
works and means `--ra`.

### Getting DATs

retrobox does not download or update DATs at this time. The DATs used by
pure-DAT mode are downloaded manually from each project's website and put
into the DAT dir (default `~/.config/retrobox/dats/`, or `--dat-dir`), or
passed with `--dat-file`; newer versions are replaced or added the same way
(of several versions of one DAT, the newest is used). No-Intro DATs come
from DAT-o-MATIC in the P/C XML (Parent/Clone XML) format, Redump DATs from
each platform's Datfile download, FinalBurn Neo DATs from its source
repository (`dats/`), MAME data from `mame -listxml` (ROM DATs by storage
method are published by third parties). Archives must be extracted first:
retrobox reads `.dat` and `.xml` files; ClrMamePro text DATs work too.

### DAT sources (pure-DAT mode)

Pure-DAT mode reads only DATs. The file format is detected from the
content: Logiqx XML (`<datafile>`), ClrMamePro text or MAME `-listxml`.
No-Intro's DATs are P/C XML (Parent/Clone XML): Logiqx XML whose header
name ends in `(Parent-Clone)` and whose clones name their parent game in
`cloneof` (what `--no-clones` uses); Redump and FinalBurn Neo also use
Logiqx XML. The source is recognized from the header (`retrobox dats scan`
shows it): No-Intro and Redump DATs map by platform name; every
FinalBurn Neo DAT goes to `fbneo` — `arcade` at its root, consoles in
sub-folders (`fbneo/nes/10yardfight.zip`). retrobox passes them on without
deciding for ScreenScraper or RetroAchievements: ScreenScraper files fbneo
under system 75, so every fbneo game is looked up there by set file name;
the RA hash is rcheevos' (`md5("nes_10yardfight")`) and RA decides which
game and console it is. MAME ROM DATs go to `mame` (CHD DATs and software lists
are skipped). Several DATs for one platform dir are told apart by identity
(source + header name without `(Parent-Clone)` and without the romset
storage method). Of one DAT only the newest version is used (MAME and
FinalBurn Neo by version, No-Intro and Redump by date), with every DAT of
that version: the split / merged / non-merged storage methods of MAME and
FinalBurn Neo are equal alternatives (any one of them, or all, work).
Different DATs are all used, with no preference: No-Intro dump formats
(`(FDS)`/`(QD)`, `(Headered)`/`(Headerless)`, N64 byte orders, …), each
dated on its own, and platforms sharing a Batocera dir (DS and DSi in
`nds`, 3DS and New 3DS in `3ds`, FinalBurn Neo subsystems in `fbneo`) —
which one a file is follows from its checksum. A path listed by several
DATs is one entry completed from all of them (e.g. the RA hash of a
headerless DAT); arcade dirs merge by set name only, so a merged parent
never hides the clones of a split DAT. No-Intro/Redump platform DATs already list their
Aftermarket/Unlicensed entries; there are no separate Aftermarket DATs.
`[BIOS]` entries are left out.

### Hashes

The RA hash code is a port of **rcheevos** (the implementation RetroArch and
Batocera use) and covers every console rcheevos hashes. Each rule was
verified against a compiled rcheevos reference (119 comparisons on synthetic
images; see `tools/fixtures/`).

Supported images and containers: `.cue`/`.bin`, `.gdi`, `.iso`, `.chd` (CD,
DVD, GD-ROM), DiscJuggler `.cdi`, `.m3u` (first disc), PSP `.cso`,
GameCube/Wii `.iso`/`.gcm`/`.gcz`/`.ciso`/`.wbfs`/`.rvz`/`.wia` (all WIA/RVZ
compressions), WiiWare `.wad`, 3DS `.3ds`/`.cci`/`.cia`/`.3dsx`, PS3 `.iso`,
`.chd` and `Game.ps3/` folders, MS-DOS `.zip`/`.dosz`, Arduboy.

- **Wii RVZ/WIA** need no console key: the files store each partition's
  title key, and retrobox rebuilds the encrypted data (hash tree +
  encryption). Convert losslessly — Dolphin without "Remove Junk Data", or
  `wit --raw`; scrubbed or NKit images differ from the disc (NKit images are
  detected and skipped with a warning).
- **Encrypted 3DS dumps** need your own Citra/Azahar `aes_keys.txt`
  (`RETROBOX_3DS_KEYS`, `~/.config/retrobox/aes_keys.txt`, or the
  emulators' `sysdata` folders). No keys ship with retrobox; without keys no
  hash is written (never a wrong one).
- DAT mode can use DAT md5s only where the RA hash is the ROM md5; disc
  platforms and NDS need the actual files (`scrape roms`).

When a hash does not match RetroAchievements, run
`retrobox hash --system <dir> --explain <file>` and compare with the hashes
listed under the game's "Supported Game Files" on retroachievements.org.

## Output

```
roms/<platform>/
├── gamelist.xml          ./<rom>, media paths; ssid/cheevosId/cheevosHash tags
├── .retrobox-state.json  resume state
├── images/  marquees/  videos/  manuals/
```

Media are named after the ROM on disk and deduplicated per platform dir.

## Platforms

retrobox's platforms are Batocera's (about 150 scrapable consoles,
handhelds, computers, arcade boards and pinball; engine ports and tools are
not platforms). `retrobox systems list` shows every platform dir (aliases
such as `ps1` for `psx` or `segacd` for `megacd` included) with its
ScreenScraper system id; `--json` adds the whole registry row. All platform
knowledge — dir names (the canonical one is Batocera's roms dir), the
ScreenScraper and RetroAchievements ids Batocera uses, file kind
(cartridge, disc, arcade set), Batocera's extensions and the keywords that
identify a DAT — lives in one registry row per platform
(`src/systems/table.rs`), checked against Batocera's sources with
`tools/batocera/`. `retrobox hash --explain --system <dir> <file>` shows
which RA console and hash rule a dir uses.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | some entries unmatched or failed |
| 2 | usage/environment error |
| 3 | stopped early: quota, credentials, API closed/offline (progress saved) |
| 130 | interrupted (Ctrl-C; progress saved) |

## Trying it out and feedback

retrobox is under active development, and trying it out, testing it and
reporting back are welcome. Please open an issue in this repository
(<https://github.com/rshi0212/RetroBoxCli/issues>) for problems, wrong or
missing matches, platforms that are not recognized, or suggestions. Useful
details: the retrobox version (`retrobox --version`), the command line, the
platform dir, and the relevant output (`-v` for more detail; credentials
never appear in it, but check before posting). For a hash question, the
output of `retrobox hash --explain --system <dir> <file>` helps.

## Credits

Data from [ScreenScraper](https://www.screenscraper.fr) (CC BY-NC-SA 4.0)
and [RetroAchievements](https://retroachievements.org). The ScreenScraper
API is for free software only; please follow both services' terms, and
contribute or donate to raise your quota.

The prebuilt binaries are free to use; the source code is not public at
this time.
