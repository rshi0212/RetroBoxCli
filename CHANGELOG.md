# Changelog

All notable changes to retrobox are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/). Since 2026-10-01 releases
are versioned `YYYY.MM.DD-<commit>` (the built commit's date and short
hash, as `retrobox --version` and the release tag show it); sections are
headed by the release date. `0.1.0` is the earlier development milestone.

## [Unreleased]

## [2026.10.01]

Fixes from the 2026-09-29 code audits (data safety, credential handling,
correctness of both scrape modes, CI), followed by the remaining feature
gaps (RA hashes for every rcheevos console, `--only-new`, zip64),
performance work and hardening found by fuzzing.

### Added
- **Self-update**: `retrobox -U` / `--update` installs the latest GitHub
  Release in place; `--update-to <TAG>` installs a specific release (may
  downgrade). The binary is verified against the release's `SHA256SUMS`
  before it atomically replaces the executable; `cargo install` and
  source-tree builds are left alone with instructions instead.
  `RETROBOX_UPDATE_BASE_URL` overrides the GitHub API root.
- Release job builds the release assets (`retrobox`, `SHA256SUMS`,
  `retrobox.cdx.json`) and fails when the tag does not match the version
  the binary reports.
- `docs/installation.md`: installing the release binary with curl/wget into
  `~/.local/bin` or `/usr/local/bin`, verification, updating, building
  from source, uninstalling.
- **RetroAchievements hashes for disc systems**, ported from rcheevos:
  PlayStation, PlayStation 2, PSP, Sega CD, Saturn, PC Engine CD, PC-FX,
  Neo Geo CD, Dreamcast and 3DO, from `.cue`/`.bin`, `.gdi`, `.iso`,
  **`.chd`** (CD, DVD and GD-ROM CHDs) and `.m3u` playlists. Verified
  against the compiled rcheevos reference on synthetic discs of every
  format (regression fixtures in `tests/fixtures/disc/`).
- Complete rcheevos cartridge rules: PC Engine header, SNES 8 KiB rule,
  Nintendo 64 byte orders, Nintendo DS layout, FBNeo subsystem folders,
  `.neo` carts, the 64 MiB hashing cap, and the list of consoles rcheevos
  can hash (others report "unsupported" instead of a wrong hash).
- `retrobox hash --system <dir> <files>` prints crc32/md5/sha1 and the RA
  hash of any file (debugging "RA does not match").
- ROM mode scans sub-directories (depth 4, media and hidden folders
  skipped, no symlinked dirs); sheets and playlists resolve references
  relative to themselves, so multi-disc `.m3u` sets are one game.
- Release job: CycloneDX SBOM and Sigstore-backed build-provenance/SBOM
  attestations (`gh attestation verify`).
- RetroAchievements hashes for **every console rcheevos hashes**:
  - GameCube from `.iso`/`.gcm`/`.gcz`/`.ciso`/`.wbfs`/`.rvz`/`.wia`, Wii
    discs from the same containers, WiiWare `.wad`. Wii `.rvz`/`.wia`
    store partitions decrypted without hashes; the reader rebuilds the
    H0/H1/H2 hash tree per 2 MiB group, applies the stored hash exceptions
    and re-encrypts with the title key the file stores — no console key is
    needed. WIA/RVZ compression: none, PURGE, bzip2, LZMA, LZMA2,
    Zstandard, RVZ packing;
  - Nintendo 3DS (NCSD, NCCH, CIA, 3DSX, ELF). Encrypted dumps use the
    user's own Citra/Azahar `aes_keys.txt` (+ `seeddb.bin`); no keys ship;
  - Atari Jaguar CD (second-session boot track, byte-swapped discs,
    homebrew), PS3 (`.iso`/`.chd` and `Game.ps3/` folder games), PSP `.cso`,
    MS-DOS (`.zip`/`.dosz` with parent archives and `.dosc`), Arduboy;
  - DiscJuggler `.cdi` images for Dreamcast and the other CD platforms
    (v2/v3/v3.5 descriptors, 2048/2336/2352/2448-byte sectors,
    multi-session; track positions as flycast computes them).
  All rules are verified against the compiled rcheevos reference (119
  comparisons on synthetic images); containers were produced with Dolphin `dolphin-tool`,
  Wiimms ISO Tools and MAME `chdman`, CDI layouts cross-checked with
  libmirage.
- `--only-new`: skip games already scraped (gamelist entry with an
  `<ssid>`); in ROM mode such files are not even hashed.
- ROM hash cache (`<cache>/hashes/`, validated by size + mtime, invalidated
  when hashing rules change or 3DS keys appear).
- `--rename` writes zip64 archives for members/archives of 4 GiB and more
  (previously refused).
- Platform dirs `ps3`, `dos`, `jaguar`/`jaguarcd`, `arduboy` (RA).
- `--report <FILE>` (both modes): JSON list of unmatched and failed
  entries, title-search matches to review, and games lacking requested
  media; written on quota stops and Ctrl-C too.
- `scrape roms --hash-only`: exact hash matches only, no title search
  (spares the unrecognized-ROM quota on hack/translation sets). The ROM
  mode summary line now splits matches into "by hash" and "by title
  search".
- `retrobox hash --explain`: step-by-step trace of how each RA hash was
  computed (rule, track and sector layout, ISO files, boot executable,
  container, 3DS keys); also in `--json`.
- NKit GameCube/Wii images are detected (as Dolphin does) and get a
  warning instead of a silently wrong RA hash.
- `tools/fixtures/`: the fixture generators and rcheevos reference-tool
  sources; every committed fixture is reproducible byte for byte
  (`check_committed.py`), 119 comparisons against rcheevos.
- Fuzz targets (`fuzz/`, cargo-fuzz) for the DAT parsers, API JSON,
  gamelist merge and disc image readers.
- Size limits on API/RA/media responses, DATs, existing gamelists, sheets
  and CHD headers.
- **Platforms follow Batocera**: 155 scrapable platforms (was 85), with
  Batocera's roms dir names, extensions, and the ScreenScraper and
  RetroAchievements ids its own EmulationStation uses — new among them
  Xbox, Xbox 360, CD-i, FM Towns, PC-88/98, Nintendo 64DD, PS Vita,
  Videopac+, MSX2+, Sharp X1, Mega Duck, Supervision, Game.com, V.Smile and
  many computers. ScreenScraper child systems are kept where Batocera uses
  the parent (Naomi 2, Model 2, Cave, Amiga AGA, Jaguar CD).
  `tools/batocera/` extracts Batocera's data from its sources and checks
  the registry against it and against ScreenScraper's system list.
- `systems list --json` also prints each dir's canonical dir, RA console,
  file kind and extensions.
- Folder games for every platform: a folder named with one of the
  platform's extensions (`Game.daphne/`, `Game.ps4/`, `Game.pc/`) is one
  game, as EmulationStation shows it (was PS3 only).
- `tests/cli.rs`: the real binary's output formats and exit codes.
- A `[note]` when a platform's gamelist sits in another name of its dir
  (`segacd/` for `megacd`) or a ROM-mode scrape uses such a name: Batocera
  reads only the canonical dir.
- `rust-version = "1.88"` (verified with 1.88.0); CI checks it (`msrv`
  job, gating releases) and checks that the fuzz targets compile.

### Security
- ScreenScraper responses echo the request URL (`header.commandRequested`)
  and embed the developer and member credentials in every media URL; these
  responses were stored as-is in the response cache (`api/*.json`), so the
  cache held `devpassword`/`sspassword` in plaintext. Credential parameters
  (`devid`, `devpassword`, `ssid`, `sspassword`) are now removed from every
  URL in a response before it is cached or used, and added back only when a
  media file is downloaded from a trusted ScreenScraper host. Cache entries
  written by earlier versions are rewritten without credentials the next
  time they are read; `retrobox cache clear` removes them at once.
- Network errors no longer include request URLs, so `devpassword`,
  `sspassword` and the RA API key never reach logs or stderr.
- Member credentials are appended to media URLs only for
  `https://*.screenscraper.fr` or the configured API origin (parsed host,
  not a substring match); redirects are never followed across hosts.
- `--system` rejects absolute paths, `..` and path separators (it is joined
  onto `--roms-root`).
- Zip member names are never used as filesystem paths (a crafted
  `../../x` member could make `--rename` move files outside the ROM dir).
- `auth show` no longer panics on multi-byte (e.g. Chinese) credentials and
  never reveals password characters.
- Cache files are written atomically with `0600` permissions.
- quick-xml 0.36 → 0.42 (RUSTSEC-2026-0194/0195); indicatif 0.18 drops the
  unmaintained `number_prefix`.

### Fixed — data safety
- `--rename` produced zips with a zlib-wrapped (undecompressable) member and
  then deleted the original. Members are now raw deflate, streamed, and the
  new archive is verified (size + CRC per member) before the original is
  removed; temp files are unique; targets are never overwritten.
- `gamelist.xml` is merged instead of overwritten: resumed entries, entries
  filtered by `--ra-only`, platforms after a quota stop and Batocera user
  fields (favorite, playcount, …) are preserved. Writes are atomic; a
  `.bak` is kept on the first rewrite of a foreign file.
- Resume state is recorded on every online run, checkpointed periodically,
  saved on quota stops and Ctrl-C (exit 130), and keyed by the options that
  shape the output. A fatal stop ends the whole run.
- Cross-platform media dedup referenced files in another platform's dir;
  dedup is now per platform dir. Media files are named after the ROM's disk
  name (no more regional variants sharing art); one failed kind no longer
  drops all media of a game; downloads go through `.part` files.

### Fixed — correctness
- Pure-DAT mode recognizes the source of each DAT from its header
  (No-Intro, Redump, FinalBurn Neo, MAME; `dats scan` has a SOURCE column)
  and places games by the source's conventions:
  - every FinalBurn Neo DAT belongs to `fbneo` (`arcade` at its root,
    consoles in sub-folders: `fbneo/nes/10yardfight.zip`); before, the
    console DATs went to `nes`, `megadrive`, … and `tg16` into `pcengine`;
  - FinalBurn Neo `hdd`/`samples`, MAME CHD DATs and software lists,
    Redump arcade-board discs and No-Intro flux/waveform captures are
    skipped with their reason (CHD-only DATs used to abort the run);
  - No-Intro dump formats of one platform (`(FDS)`/`(QD)`, `(Headered)`/
    `(Headerless)`, N64 byte orders, …) and romset storage methods of MAME
    and FinalBurn Neo (`(merged)`/`(split)`/`(non-merged)`) are equal
    alternatives: all are used, which one a file is follows from its
    checksum;
  - neighbouring platforms stay apart: SuperGrafx (now `supergrafx`),
    Videopac+, Amiga CD and GameShark update discs no longer land in
    `pcengine`, `o2em`, `amiga` and `psx`; `tg16` is a platform dir of its
    own;
  - `[BIOS]` entries of No-Intro/Redump DATs are left out of gamelists.
- Pure-DAT mode: several DATs for one platform dir are told apart by their
  identity — source plus header name without `(Parent-Clone)` and the
  romset storage method. Of one DAT only the newest version is used (MAME
  and FinalBurn Neo by version number, No-Intro and Redump by DAT date:
  header, file name, file time), with every DAT of that version — the
  split / merged / non-merged storage methods, named in MAME headers but
  not in FinalBurn Neo ones. Different DATs — No-Intro dump formats, each
  dated on its own, and platforms sharing a Batocera dir (No-Intro DS and
  DSi in `nds`, 3DS and New 3DS in `3ds`, FinalBurn Neo subsystems in
  `fbneo`) — are all used. A path several DATs list is one entry completed
  from all of them; arcade dirs merge by set name only (a merged parent
  holds its clones' ROMs); the result does not depend on the order DATs are
  read. Before, the largest DAT won silently.
- Pure-DAT mode with `--dat-dir` and a single `--system` assigned every DAT
  whose header names no known platform to that platform. Only explicit
  `--dat-file`s are forced now; unrecognized DATs are summarized in one
  line.
- Pure-DAT mode paths come from the DAT alone; a header declaring
  `forcepacking="zip"` (FinalBurn Neo) makes them `<game name>.zip`.
- FinalBurn Neo subsystem games (`fbneo/<subsystem>/x.zip`), in both modes,
  are fbneo sets: looked up on ScreenScraper under system 75 (where
  ScreenScraper files fbneo) by set file name — ROM mode no longer sends
  the sub-folder in `romnom` — and given rcheevos' RA hash
  (`md5("nes_x")`, `md5("x")` in folders rcheevos does not prefix). RA
  decides which game and console a hash is: the match is checked against
  the lists of arcade and of every subsystem's console.
- Channel F (`channelf`, ScreenScraper 80, RA 57) and SuperGrafx
  (`supergrafx`, ScreenScraper 105, RA files it under PC Engine) are
  platform dirs; No-Intro's DATs for them are no longer skipped.
- Games ScreenScraper does not find (or files as non-games) were left out
  of gamelists in both online modes, and their RetroAchievements data with
  them. They are recorded now: named after the RA title, DAT name or file,
  with RA tags (and `--ra-fill` data) when RA has them, without an `ssid`;
  into an existing entry they only add missing fields (RA tags follow RA).
  The run summary counts them as missing, not matched.
- `--ra-only` in pure-DAT mode skipped arcade dirs although their RA hash
  (set name) comes from the DAT.
- ScreenScraper's non-game entries (`notgame`, "ZZZ(notgame):#NONGAME")
  are reported as not found instead of becoming a game's metadata.
- Pure-DAT mode used the DAT md5 as RetroAchievements hash for headered
  dumps (No-Intro "Headered" NES DAT: wrong for every game). Entries whose
  size shows a header rcheevos strips get no RA hash.
- DAT platform names: No-Intro's "NeoGeo Pocket [Color]", "Wii", "Wii U",
  "Game & Watch", "Satellaview" and "Sufami Turbo" are mapped; DATs map to
  Batocera's dir names (`gb`, `ngp`/`ngpc`, `wswan`/`wswanc`, `lynx`,
  `jaguar`, `o2em`, `msx1`), which ROM mode recognizes too. `--system`
  selects DATs by exact dir name.
- ROM mode: archives named after a machine (`famista93e.zip`) search
  ScreenScraper by the member's title, and trailing GoodTools tags
  (`T-Eng`, `v2.2`) are dropped from search titles.
- Disk-only (CHD) DATs report that they list no ROM files instead of
  asking whether the file is a Logiqx DAT.
- RetroAchievements requests retry dropped connections, 429 and 502-504;
  the per-platform game lists are cached for 24 h.
- RetroAchievements: the hash library also lists registered games without
  achievements (SNES: 2831 registered, 1184 with achievements), and the
  public channel tagged them with a `cheevosId`; `--ra-only` kept them, and
  `--ra` and `--ra-v1` gave different results. A hash now matches only a
  game in its console's list of games with achievements
  (`r=officialgameslist`, or the Web API with `f=1`), which also rejects
  hashes of another console's games. The lists load before `--ra-only`
  filters DAT entries.
- ROM mode: zipped ROMs got `<path>` = the inner member name; the path is now
  the zip on disk. Arcade zips are treated as sets (not unpacked). Tracks
  referenced by `.cue`/`.gdi`/`.m3u` are no longer scraped separately.
- `--rename` uses ScreenScraper's canonical ROM name (or the scraped title +
  original region/version tags), only for exact hash matches, never for
  arcade sets, disc images or multi-file zip internals.
- ROM mode now uses the response cache; `--cache-dir` works in both modes
  and for `cache stats/clear`; `--threads` caps real concurrency (queries
  and media now run concurrently up to the account's `maxthreads`; the
  concurrency permit is held for the whole request).
- Token bucket no longer blocks forever at < 30 requests/min; media and
  resized downloads are paced and retried like API calls; transient
  network/5xx errors are retried; 401/423 stop the run like 430.
- ScreenScraper `jeuInfos` responses in the live array shape (`noms`,
  `synopsis`, `medias`, `dates` as arrays) are parsed; names/dates follow
  `--region-priority`.
- DAT parsing: paired `<rom></rom>` elements, sha1-only entries, entity
  decoding (`&amp;`), MAME `build`, multi-ROM games (largest ROM's hashes,
  `.cue` path), FBNeo/MAME Logiqx set names, ClrMamePro `romof` lines and
  quoted parentheses (tokenizer rewrite). DAT summary cache keys include
  the full path.
- DAT platform detection handles No-Intro/Redump "Vendor - Platform" names;
  an unrecognized DAT can be forced with a single `--system`.
- System IDs: Intellivision (115), MAME family (75), SG-1000 (109),
  VIC-20 (73), MSX (113), Neo Geo Pocket (25); removed the misspelled
  `gameboricolor`; added `segacd`/`megacd` and `fds`. Every ID now belongs
  to one platform (unit-tested); `systems list` derives from the table.
- RetroAchievements: `--ra-v1` indexes the platforms actually scraped (it
  indexed nothing without `--system`), falls back to the public library per
  failed console, and `--ra-only` errors out when no index could be loaded.
  `--ra-fill` works in ROM mode and never blanks DAT values. The RA media
  fallback maps box art to `box-2D` and title screens to `ss-title`. NES/
  FDS/Lynx/7800/SNES headers are stripped for RA hashes; platforms without
  an rcheevos rule are reported as unsupported instead of hashed wrongly. The public hash library is cached for 24h.
- marquee/manual/fanart/box back/title screen/wheel media are written to
  their Batocera tags; unknown `--media` kinds are rejected.
- `--dry-run` creates no directories and writes no cache.
- Exit codes follow the docs: 1 when any entry is unmatched, 3 for
  `auth check` credential rejections, 130 on Ctrl-C.
- `-v`/`-vv` map to debug/trace, `RUST_LOG` is honoured, `-q` silences
  status output; explicit `--env-file` overrides the environment.
- `releasedate` uses the real month/day and skips invalid years; invalid
  ratings and `ssid 0` are no longer written.
- WIA PURGE images: the raw-data and group tables are PURGE-encoded too
  and were read as-is, so every PURGE `.wia` was reported as not
  recognized.
- Dir aliases now share every attribute of their platform: `col` (alias of
  `colecovision`) had no RetroAchievements console, so its games got no RA
  tags; `neo-geo` (alias of `neogeo`) had no RA console either, and ROM
  mode scanned it for `zip`/`bin`/`rom` instead of `zip`/`7z`/`neo`.
- An archive retrobox cannot open (`.7z`, `.rar`, …) on a whole-file
  platform got the archive's md5 as its RA hash; no RA hash is written for
  such files now (arcade sets, hashed by name, are unaffected).
- CloneCD (`.ccd` + `.img`/`.sub`), Alcohol (`.mds` + `.mdf`) and cdrdao
  (`.toc`) dumps were scraped once per file; the files a sheet names are
  part of its game.
- The DAT keyword for Atari 8-bit never matched (`8-bit` normalizes to
  `8 bit`); the combined SG-1000/SC-3000 DAT and the V.Smile Pro disc DAT
  are kept apart from the new platforms.
- `tests/self_update.rs` ran `-U` against the build itself when
  `CARGO_TARGET_DIR` lies outside the source tree, replacing it with test
  bytes.

### Fixed — found by fuzzing
- A `gamelist.xml` starting with a UTF-8 BOM (common for files edited on
  Windows) made the merge panic.
- Malformed markup that the XML reader tolerates could be copied verbatim
  into the output; such files are now treated as corrupt and moved aside.
- Corrupt CHD headers could trigger multi-GB allocations; a panic in the
  `chd` crate's Huffman decoder on corrupt maps is contained (the image is
  reported as not recognized, the scrape continues).
- Container headers (WIA/RVZ tables and chunk sizes, 3DS ExeFS) are
  bounded before allocating; LZMA output is capped; corrupt GameCube DOL
  tables and WAD content lists that point past the end of the file are
  rejected instead of hashing gigabytes of padding.
- A corrupt Wii RVZ/WIA hash group was rebuilt for every cluster read
  (15× slower than a valid image); failures are cached now.

### Performance
- Scrapes identify DATs from their headers only; each DAT is fully parsed
  at most once, and only when its platform is scraped.
- ROM hashing runs on the blocking pool, in parallel across CPU cores.
- RA header checks reuse the bytes captured while hashing (no second read
  of files or zip members); one RA HTTP client per run (connection reuse).
- The pre-scrape account check fails fast instead of retrying; retry
  backoff is injectable, so the whole test suite runs in a few seconds.
- ROM mode re-runs reuse the hash cache: unchanged files are not read
  again.

### Changed
- RetroAchievements options: `--ra` tags games that have achievements,
  `--ra-only` additionally scrapes only those games, and `--ra-fill`
  additionally fills metadata and artwork from RA (needs `RA_API_KEY`). The
  hash index source is chosen automatically (public hash library, the Web
  API per console when the library is unavailable and a key is set) since
  both carry the same hashes; `--ra-v1` is hidden and treated as `--ra`.
  The RetroAchievements options have their own help section.
- `#![warn(missing_docs)]` on the library crate (CI builds with
  `-D warnings`), and every public item is documented.
- Regression tests for each bound found by fuzzing (CHD header, WIA
  chunk/table sizes, WAD, DOL, 3DS ExeFS, CDI descriptor).
- zlib is compiled in statically (`libz-sys` "static"): no runtime
  `libz.so` dependency.
- CI: pinned toolchain and action SHAs, `RUSTDOCFLAGS=-D warnings`,
  `cargo audit` job, release job actually triggers on `v*` tags, SHA-256
  checksum for the release binary, Dependabot for cargo and actions.
- Removed unused dependencies (`rayon`, `toml`, `once_cell`).
- Internal restructuring for modularity and extensibility; scrape results
  are unchanged (verified on 309 real DATs: identical `dats scan` output and
  byte-identical offline gamelists for 59 platforms):
  - **Platform registry** (`systems/`): one row per platform holds its dir
    names, ScreenScraper and RetroAchievements ids, file kind, ROM
    extensions and DAT name keywords, replacing seven separate tables in
    `systems`, `ra`, `rom::scanner`, `scrape::dat_mode` and `rom_mode`.
    Adding a platform is one row; unit tests check the invariants.
  - DAT selection (BIOS filtering, FinalBurn Neo placement, newest version
    per DAT identity, union) moved from `scrape::dat_mode` to `dat::select`;
    the DAT name guesser to `systems::dir_for_dat_name` (the data source
    no longer calls into the orchestration layer).
  - Both modes share one per-platform flow (`scrape::platform`: resume
    state, `--only-new`, query loop, gamelist merge, exit status) instead of
    two copies.
  - Scrape options are structs (`ScrapeOptions`, `DatJob`, `RomsJob`)
    instead of 15–17 positional parameters; the resume signature is built
    in one place and pinned by a test, so existing `--resume` state stays
    valid.
  - Exit codes are an `ExitStatus` enum; command handlers moved from
    `main.rs` into `commands/` (one module per command group).
- `systems map <id>` prints the platform's canonical dir (`3 -> nes`,
  `57 -> psx`) instead of the alphabetically first alias (`famicom`, `ps1`).
- Canonical platform dirs are Batocera's: Sega CD `megacd` (was
  `segacd`), Odyssey 2 `odyssey2` (was `o2em`), VIC-20 `c20` (was `vic20`),
  Amiga `amiga500` (was `amiga`). DAT mode writes there, so move gamelists
  written to the old dirs; ROM mode keeps working with either name.
  `tg16` is the same platform as `pcengine` (as in Batocera), `fba` the
  same as `fbneo`; `lcdgames` (MAME LCD handhelds) is no longer an alias of
  Game & Watch (it shares ScreenScraper's Game & Watch system, where
  ScreenScraper files MAME LCD sets). A DAT named exactly like an alias, and
  `scrape dat --system <alias>`, map to the canonical dir.
- ROM mode scans exactly Batocera's extensions per platform (e.g. `psx`:
  no loose `.bin`/`.zip`; `nes`: no `.fds`, which belongs to `fds`).
- Arcade boards Batocera classifies as arcade (Naomi, Naomi 2,
  Atomiswave, Model 2/3, Cave, ST-V) are matched as sets by name, like MAME.
- The ROM hash cache is invalidated once (RA consoles changed for several
  platforms).
- Releases (binary, `SHA256SUMS`, SBOM) and the user documentation are
  published to the public repository `rshi0212/RetroBoxCli` by
  `tools/release/publish.sh` (release notes from this file); the source
  repository is private. GitHub build attestations, which private
  repositories cannot produce on this plan, are no longer attached.
- Docs: how to get DATs (downloaded manually; retrobox does not download or
  update them), No-Intro's P/C XML format, and how to report feedback.
- Versions are `YYYY.MM.DD-<commit>` (build.rs) instead of SemVer:
  `retrobox --version`, release tags (`v2026.10.01-7a20a85`) and
  `retrobox -U` use them; `-U` never installs an earlier date, and another
  build of the same day counts as the latest release.

## [0.1.0] — 2026-09-27

### Added
- **Pure-DAT mode** (`scrape dat`): scrape from No-Intro/Redump (Logiqx),
  ClrMamePro and MAME listxml DATs; ROM files are never read. Includes
  `--offline` (zero-API gamelist generation) and `--no-clones`.
- **Pure-ROM mode** (`scrape roms`): hash + filename-search matching for
  collections without DATs. 40+ platform extension tables.
- **RetroAchievements integration** for both modes:
  - `--ra` — match and export `cheevosId`/`cheevosHash` into gamelist.xml
    (Batocera-native tags; ES shows the achievements icon immediately).
  - `--ra-only` — scrape only RA-supported entries/ROMs.
  - `--ra-v1` — official v1 per-console hash index with achievement counts.
  - `--ra-fill` — fill missing publisher/developer/genre/year from RA and
    use RA title/boxart artwork as a media fallback.
- **`--rename`** — canonical ROM renaming; zip containers are renamed inside
  and re-packed to the **TorrentZip** standard (deterministic layout).
- **Quota engine**: three-dimensional adaptive limiting from the live
  `ssuser` block (threads / per-minute / per-day), 429 backoff, 430 daily
  breaker, 431 unrecognized-ROM classification, 403 credentials detection
  in a 200 body.
- **Two-level caching**: API responses (hit 7d / miss 24h) and DAT parse
  summaries (mtime/size keyed).
- **`quota`** command with `--watch N` live refresh; does not consume quota.
- **`doctor`** environment self-check; **`dats scan/info`**; **`systems
  list/map`** (99 Batocera platform mappings); **`cache stats/clear`**;
  **`completion bash|zsh|fish`**.
- Full multi-progress-bar UX (DAT scan, hashing, queries with live quota,
  media downloads).
- Resume sidecar (`.retrobox-state.json`) with run-signature invalidation.

### Changed
- ScreenScraper client links real zlib (`libz-sys`) for TorrentZip
  byte-exactness; flate2 default backends are not zlib-identical.
