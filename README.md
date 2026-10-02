# retrobox

ScreenScraper + RetroAchievements ROM 刮削 CLI，输出 Batocera 风格的 `roms/<platform>/gamelist.xml` + 媒体文件。

单个二进制，zlib 静态编译进程序（无需系统 libz）；发布的 Linux x86_64 版本为完全静态的 musl 构建，
没有运行时依赖。预编译版本免费使用；源代码暂未公开。

## 文档

| 文档 | 内容 |
|---|---|
| [README](README.md) | 快速开始、命令参考（本文件） |
| [README.en.md](README.en.md) | English overview and quick start |
| [docs/installation.md](docs/installation.md) | 安装发布版二进制、`retrobox -U` 自更新、校验、卸载 |
| [docs/configuration.md](docs/configuration.md) | .env 完整参考、配额等级表、目录布局、退出码 |
| [docs/troubleshooting.md](docs/troubleshooting.md) | 错误码逐条排查、匹配失败、RA 哈希、性能调优 |
| [CHANGELOG.md](CHANGELOG.md) | 版本历史 |

欢迎试用、测试并反馈，见[试用与反馈](#试用与反馈)。

## 两种独立刮削模式

| 模式 | 数据源 | 适用场景 |
|---|---|---|
| `retrobox scrape dat` | DAT 文件（No-Intro P/C XML、Redump XML、ClrMamePro、MAME listxml） | ROM 未下载完、在别的机器刮削、超大库预生成 |
| `retrobox scrape roms` | 磁盘上的 ROM 文件 | 没有 DAT 的散装合集；哈希+文件名混合匹配 |

两种模式互斥（单次运行只走一条路径），但共享输出格式、媒体管线、RA 匹配、缓存、断点续跑与并发。

**不会覆盖已有数据**：写 `gamelist.xml` 时与现有文件按 `<path>` 合并——只替换 retrobox
本次写出的字段，`favorite`/`playcount`/`lastplayed`/手工编辑的字段、未知标签以及本次未处理的
条目全部保留；原子写入（临时文件写入并落盘后 rename，写入中途断电也不会留下空文件或半个文件），首次改写非 retrobox 生成的文件时保留
`gamelist.xml.bak`，无法解析的旧文件改名为 `gamelist.xml.corrupt` 而不是被覆盖。

**如实记录**：ScreenScraper 没有找到的游戏（包括 SS 标为非游戏的条目）同样写入 `gamelist.xml`——
名称取 RetroAchievements 标题，没有时取 DAT 名称或文件名；RA 有结果时照常写入 `cheevosId`/
`cheevosHash`（`--ra-fill` 时也写入 RA 的资料与图片），不写 `ssid`。对已存在的条目，这类记录只补充
缺少的字段，不覆盖以前刮到的或用户编辑的内容（RA 标签按 RA 当前结果更新）。这些游戏在 `--report`
中列为 `missing`；没有 `ssid`，因此 `--only-new` 会在下次运行时重新查询，SS 收录后即可补全。

## 安装

Linux x86_64 发布版为单个静态二进制，下载到 `PATH` 中的目录即可：

```bash
# 当前用户（~/.local/bin）
mkdir -p ~/.local/bin
curl -L https://github.com/rshi0212/RetroBoxCli/releases/latest/download/retrobox -o ~/.local/bin/retrobox
chmod a+rx ~/.local/bin/retrobox

# 或所有用户（/usr/local/bin）
sudo curl -L https://github.com/rshi0212/RetroBoxCli/releases/latest/download/retrobox -o /usr/local/bin/retrobox
sudo chmod a+rx /usr/local/bin/retrobox
```

更新到最新版本：

```bash
retrobox -U                    # 安装在 /usr/local/bin 时用 sudo retrobox -U
retrobox --update-to v2026.10.01-7a20a85   # 安装指定版本（可安装旧版本）
```

自更新会校验 `SHA256SUMS` 后原子替换可执行文件。wget 安装、校验与卸载见
[docs/installation.md](docs/installation.md)。目前只提供 Linux x86_64 版本。

## 快速开始

```bash
# 1. 配置自己的账号（均为可选）
cat > .env << 'EOF'
SS_SSID=你的会员名             # ScreenScraper 会员账号，决定配额等级；不填为匿名访问
SS_SSPASSWORD=你的会员密码
RA_API_KEY=你的RA_Web_API密钥   # 可选，--ra-fill 需要（--ra/--ra-only 不需要）
EOF

# 2. 环境自检
retrobox doctor

# 3. DAT 文件放入 ~/.config/retrobox/dats/（或用 --dat-file / --dat-dir）
#    DAT 需从各项目网站手工下载（见下文「DAT 文件的获取」）

# 4. 刮削
retrobox scrape dat --roms-root /userdata/roms --media box-2D,ss --resume
retrobox scrape roms --roms-root ./roms --ra-fill
```

## 命令总览

| 命令 | 说明 |
|---|---|
| `auth check` | 校验凭据，显示账号等级/线程/配额 |
| `auth show` | 显示当前凭据来源（密码打码） |
| `quota [--json] [--watch N]` | 实时配额面板（不消耗每日配额）；`--watch` 至少 1 秒，与 `--json` 同用时每次刷新输出一行 JSON（含 `time` 字段） |
| `systems list` / `map <目录或ID>` | Batocera 目录名 ↔ ScreenScraper 系统 ID 对照（按 ID 查询时给出该平台的规范目录名） |
| `dats scan` / `info` | DAT 目录扫描（条目数/格式/系统识别） |
| `scrape dat` | 纯 DAT 模式刮削 |
| `scrape roms` | 纯 ROM 模式 |
| `hash --system <dir> <文件或文件夹>…` | 显示 retrobox 为文件计算的 crc32/md5/sha1 与 RetroAchievements 哈希（排查"RA 匹配不上"；`--explain` 逐步说明哈希是怎么算出来的，`--json` 输出） |
| `doctor` | 环境自检（.env/凭据/API/DAT 目录/缓存） |
| `cache stats` / `clear` | 响应缓存管理 |
| `completion bash\|zsh\|fish` | Shell 补全脚本 |
| `-U` / `--update-to <TAG>` | 更新到最新版本 / 指定版本（见 [docs/installation.md](docs/installation.md#更新)） |

每个命令 `-h` 精简 / `--help` 完整（含真实可复制的示例）。

## scrape 选项

两种模式共享的选项：

```
--media <LIST>            媒体类型：box-2D,box-2D-back,ss,ss-title,wheel,wheel-hd,
                          marquee,screenmarquee,video,manuel,fanart
                          none=纯信息 / all=box-2D,ss,wheel,marquee（未知类型直接报错）
--max-width/--max-height  服务端缩放图片（mediaJeu.php）
--fields <LIST>           XML 字段选择：name,desc,year,publisher,developer,
                          players,rating,genre 或 all/none
--region-priority         名称/日期/媒体的区域优先级（默认 us,eu,jp,wor）
--ra                      为有成就的游戏写入 RetroAchievements 游戏 ID（cheevosId）
--ra-only                 同 --ra，且只刮有成就的游戏，其余条目跳过
--ra-fill                 同 --ra，且用 RA 资料补全空缺字段与封面/标题画面（需要 RA_API_KEY）
--no-cache                不用响应缓存
--cache-dir <DIR>         缓存目录（默认 $RETROBOX_CACHE_DIR 或 ~/.cache/retrobox）
--resume                  断点续跑：跳过上次已记录的条目（进度每次都会记录）
--only-new                只刮 gamelist.xml 中还没有 ScreenScraper id 的游戏（给已刮削的库
                          补刮新增 ROM，省配额；ROM 模式下已刮削文件连哈希都不计算）
--report <FILE>           运行结束时写一份 JSON 报告：未匹配、查询失败、按标题搜索得到的
                          模糊匹配（需人工核对）、缺少所需媒体的游戏（见下）
--threads <N>             最大并发请求数（1-32，默认且不超过账号 maxthreads）
--dry-run                 预览：不请求 API、不创建目录、不写缓存
--offline                 （仅 dat 模式）零 ScreenScraper 调用，纯 DAT 元数据生成 gamelist
                          （如同时指定 --ra/--ra-only/--ra-fill，仍会访问 RetroAchievements）
```

`--system` 只接受平台目录名（如 `nes`），拒绝绝对路径、`..` 与路径分隔符。

`scrape dat` 独有：`--dat-dir`/`--dat-file`、`--no-clones`（跳过 clone）。纯 DAT 模式只读取 DAT，
gamelist 的路径、平台目录与 RA 哈希全部由 DAT 内容决定，不检查磁盘上的文件。

### DAT 文件的获取

retrobox 目前不能自动下载或更新任何 DAT。纯 DAT 模式使用的 DAT 需要从各项目的网站手工下载，放入
DAT 目录（默认 `~/.config/retrobox/dats/`，或用 `--dat-dir` 指定），也可以用 `--dat-file` 直接指定文件；
有新版本时同样手工替换或并列放入（同一 DAT 的多个版本只使用最新的，见下文）。

| 来源 | 获取方式 | 格式 |
|---|---|---|
| No-Intro | DAT-o-MATIC 网站下载，选择 P/C XML（Parent/Clone XML） | Logiqx XML，clone 条目以 `cloneof` 指向 parent |
| Redump | Redump 网站各平台的 Datfile 下载 | Logiqx XML |
| FinalBurn Neo | FinalBurn Neo 项目源码仓库中的 DAT（`dats/` 目录） | Logiqx XML |
| MAME | `mame -listxml` 的输出；按存储方式整理的 MAME ROM DAT 由第三方发布 | MAME listxml / Logiqx XML |

下载的压缩包需要先解压：retrobox 读取 `.dat` 与 `.xml` 文件。ClrMamePro 文本格式的 DAT 也可以使用。

### DAT 来源与平台目录

retrobox 从 DAT 头部识别来源（`retrobox dats scan` 的 SOURCE 列），按来源的规则决定平台目录。文件格式
（FORMAT 列）与来源无关，由内容判断：Logiqx XML（`<datafile>`）、ClrMamePro 文本、MAME `-listxml`。
No-Intro 的 DAT 为 P/C XML（Parent/Clone XML）：Logiqx XML 结构，头部名称以 `(Parent-Clone)` 结尾，
clone 条目用 `cloneof` 属性写明其 parent 游戏（`--no-clones` 据此跳过 clone）。Redump 与 FinalBurn Neo
同样使用 Logiqx XML。

| 来源 | 识别依据 | 平台目录 | 说明 |
|---|---|---|---|
| No-Intro | `<url>` no-intro.org | 按平台名，如 `Nintendo - Game Boy` → `gb` | 同一平台按转储格式分为多个 DAT（`(FDS)`/`(QD)`、`(Headered)`/`(Headerless)`、`(BigEndian)`/`(ByteSwapped)`/`(LittleEndian)`、`(Decrypted)`/`(Encrypted)`、`(A78)`/`(BIN)`、`(LNX)`/`(LYX)`、`(J64)`/`(ROM)` 等），各格式并列、没有优先级，全部使用；`(Flux)`、`(Waveform)` 为保存用的采样数据，跳过 |
| Redump | `<url>` redump.org | 按平台名，如 `Sony - PlayStation 2` → `ps2` | `Arcade - …` 为街机基板的光盘转储，不是 Batocera 街机模拟器使用的 romset，跳过 |
| FinalBurn Neo | 版本 `[FBNeo ]1.0.0.3 260723 GIT…` | 全部为 `fbneo` | `arcade` 位于 `fbneo/` 根目录；各子平台位于同名子目录，如 `fbneo/nes/10yardfight.zip`、`fbneo/megadrive/…`（FinalBurn Neo 与 rcheevos 按此识别）；`hdd`、`samples` 为辅助文件，跳过 |
| MAME | `MAME ROMs (…)`、`-listxml` | `mame` | `(merged)`、`(split)`、`(non-merged)` 是同一 romset 的不同存储方式，并列、没有优先级：先按版本号取最新版本，再使用该版本的全部存储方式（只放其中任意一种或全放都可以）；CHD DAT 只列光盘镜像，跳过；软件列表（software list）不是街机 romset，跳过 |

- 平台目录使用 Batocera 目录名（`gb`、`ngp`/`ngpc`、`wswan`/`wswanc`、`lynx`、`odyssey2`、`megacd`、`msx1`、
  `c20`、`amiga500`、`supergrafx`、`channelf` 等）；相近但不同的平台各自独立（如 SuperGrafx 与 PC Engine、
  Videopac+ 与 Odyssey 2、MSX2+ 与 MSX2），不会合并（TurboGrafx-16 在 Batocera 中就是 `pcengine`）；
  retrobox 没有对应平台的（如 V.Smile Pro 光盘）跳过并提示。
- 头部无法识别平台的 DAT 跳过，运行结束时汇总一行提示；`--dat-file <dat> --system <dir>` 可为单个
  DAT 指定平台（覆盖头部识别）。目录扫描不会把无法识别的 DAT 归入 `--system` 指定的平台。
- No-Intro/Redump 中以 `[BIOS]` 开头的条目是 BIOS 转储，不写入 gamelist。
- 头部声明 `forcepacking="zip"`（FinalBurn Neo）时，路径为 `<游戏名>.zip`；否则为 DAT 中的 ROM 文件名。
- DAT 中带文件头的条目（No-Intro "Headered" NES DAT 等）的 md5 不是 RA 哈希；同一路径在无头 DAT 中
  也有时取后者的 RA 哈希，否则不写 `cheevosHash`。
- N64 的 `(ByteSwapped)`/`(LittleEndian)` DAT 列出的是字节交换后的文件，md5 不是 RA 哈希（rcheevos 按
  `.z64` 字节序计算），这些条目不写 `cheevosHash`；`(BigEndian)` DAT 的 md5 就是 RA 哈希。
- 路径会离开平台目录的条目（含 `..` 或绝对路径）不写入 gamelist，并给出警告。
- Redump 多轨条目取最大数据轨的哈希、路径优先用 `.cue`。

**同一平台目录有多个 DAT 时**，按 DAT 身份区分。身份 = 来源 + 头部名称，名称中只去掉
`(Parent-Clone)` 与 romset 存储方式（`(merged)`/`(split)`/`(non-merged)`/`(fully non-merged)`）；
版本与日期不在名称中。No-Intro 的转储格式标记属于身份的一部分。

1. **身份相同 = 同一 DAT 的不同版本**：只用最新版本，该版本的所有 DAT 都使用。
   - 版本比较：MAME/FinalBurn Neo 按版本号（`0.288` > `0.287`，`1.0.0.3 260723` > `1.0.0.3 260411`），
     No-Intro/Redump 按 DAT 日期（头部 `<date>`，其次文件名中的日期，再次文件修改时间）。
   - 同一版本的多个 DAT 并列使用：MAME 与 FinalBurn Neo 的 `split`/`merged`/`non-merged` 存储方式。
     MAME 在头部名称中注明存储方式；FinalBurn Neo 的头部不注明（各存储方式都叫 `arcade`、`nes`…），
     同一版本的多个同名 DAT 同样全部使用。只放其中任意一种、或多种都放，结果都正确。
   - 旧版本整体跳过，不论它是哪种存储方式。
2. **身份不同 = 不同的 DAT**，全部使用，没有优先级：
   - 同一平台的不同转储格式（每种格式是单独发布、单独标注日期的 DAT）：No-Intro 的 `(FDS)` 与 `(QD)`、
     `(Headered)` 与 `(Headerless)`、N64 的 `(BigEndian)`/`(ByteSwapped)`/`(LittleEndian)`、
     `(Decrypted)`/`(Encrypted)`、Atari 7800 `(A78)`/`(BIN)`、Lynx `(LNX)`/`(LYX)`、Jaguar `(J64)`/`(ROM)` 等；
   - 共用一个 Batocera 目录的不同平台：No-Intro 的 Nintendo DS 与 Nintendo DSi → `nds`，Nintendo 3DS
     与 New Nintendo 3DS → `3ds`；FinalBurn Neo 的 `arcade` 与各子平台 → `fbneo`。
3. 一个文件属于哪种格式或存储方式，只能由它的校验值判断，因此 retrobox 不替用户选择。合并规则：
   - 路径不同的条目各自保留（如 `Game.fds` 与 `Game.qd`；merged 存储才有的父集文件，如 MAME 的
     `famicom.zip`）；
   - 同一路径出现在多个 DAT 中（如 `(Headered)`/`(Headerless)` 的 `Game (USA).nes`、各存储方式共有的
     `<set>.zip`）时只写一个条目，并用其他 DAT 补全缺失的信息（如只有无头 DAT 才提供的 RA 哈希）；
   - 非街机平台中，另一个 DAT 里路径不同但 ROM 校验值相同的条目（DS 与 DSi 都收录的游戏）只写一次；
     街机平台（`mame`/`fbneo`/`fba`/`arcade`）的游戏以集合名为准，只按路径合并——merged 父集包含
     clone 的 ROM，不能因此丢掉 split/non-merged 中的 clone；
   - 同一个 DAT 内的条目全部保留（街机 clone 可能共用 ROM）；
   - 结果与 DAT 的读取顺序无关。
4. 每次选择都会输出一行说明（使用哪个 DAT、跳过哪个、原因，或哪些 DAT 并列使用）。

No-Intro 与 Redump 的平台 DAT 本身已包含 Aftermarket、Unlicensed、Pirate 等条目（例如 Game Boy
DAT 2292 个条目中有 296 个 `(Aftermarket)`），没有单独的 Aftermarket DAT；合集中按目录分开存放的
`(Aftermarket)`、`(Private)` 文件只是存储方式不同，核对时以文件校验值与 DAT 比对为准（ROM 模式）。
`(Private)` 转储不在公开 DAT 中。

### FinalBurn Neo 子平台的刮削与 RA 匹配

FinalBurn Neo 的全部游戏，包括 `fbneo/nes/x.zip` 这类子平台游戏，都属于 `fbneo` 平台。retrobox 只负责
正确传导，不替用户、ScreenScraper 或 RetroAchievements 做判断：

- **ScreenScraper**：SS 把 fbneo 归在 75 号系统（Mame，其 recalbox 名称包含 `fbneo`），因此子平台游戏与
  `fbneo/` 根目录的 romset 一样，在 75 号系统下按集合文件名（`x.zip`）查询，SS 返回的结果原样记录
  （包括查不到）。
- **RetroAchievements**：RA 哈希按 rcheevos 规则计算——`fbneo/nes/x.zip` 为 `md5("nes_x")`，rcheevos 不加
  前缀的子目录（如 `snes`）为 `md5("x")`。这个哈希对应哪个游戏、该游戏属于哪个主机，由 RA 的数据决定；
  retrobox 只确认该游戏在 RA 归入的主机中有成就（加载街机及各子平台所模拟主机的"有成就游戏"列表）。
- 所有子平台都同等处理，包括 Channel F、SuperGrafx 等小众平台；SS 或 RA 以后补充数据时，无需任何
  改动即可匹配。
- 纯 DAT 模式与 ROM 模式对这些游戏得到相同的结果。

`--report` 的内容按平台分组，只包含本次运行处理过的条目（被 `--resume`/`--only-new`
跳过的不在其中）；配额中断或 Ctrl-C 时同样会写出，与已合并进 gamelist 的内容一致：

```json
{ "mode": "roms", "platforms": [ { "dir": "snes", "matched": 120, "by_hash": 112, "by_search": 8,
  "missing": ["Hack (Unl).sfc"], "failed": [],
  "fuzzy": [ { "file": "Mario (Beta).sfc", "title": "Super Mario World", "ssid": 1234 } ],
  "missing_media": [ { "file": "Game.sfc", "kinds": ["video"] } ] } ] }
```

`scrape roms` 独有：
- `--roms-root` 下按平台扩展名自动扫描（扩展名来自平台注册表，见[支持的平台](#支持的平台)），**包括子目录**（最多 4 层；
  跳过隐藏目录、`images`/`videos`/`manuals` 等媒体目录，不跟随符号链接目录）。子目录中的
  游戏在 gamelist 中写为 `./子目录/游戏.zip`。
- 哈希阶段按 CPU 核数并行（最多 8 路），不阻塞网络请求；文件哈希缓存在缓存目录
  （按相对路径 + 大小 + 修改时间校验），未改动的大库重跑不再重读文件（`--no-cache` 关闭）
- 文件夹游戏：以平台扩展名命名的文件夹（`游戏.ps3/`、`游戏.daphne/`、`游戏.pc/` 等）与含
  `PS3_GAME/USRDIR/EBOOT.BIN` 的文件夹各作为一个游戏刮削，与 EmulationStation 的显示一致
- `.7z`、`.rar` 等 retrobox 无法打开的压缩包照常刮削（ScreenScraper 可按文件名匹配），但不写
  `cheevosHash`——RA 哈希针对压缩包里的内容，retrobox 不会写出压缩包本身的错误哈希；需要成就标识时请解压
- 光盘描述文件引用的文件不单独刮削：`.cue`/`.toc` 的轨道、`.gdi`、`.m3u` 的各张碟、CloneCD `.ccd` 的
  `.img`/`.sub`、`.mds` 的 `.mdf`
- 匹配顺序：先按哈希查询（精确），查不到再按解析出的标题搜索（取服务器排名第一的结果，
  **不保证正确**，用 `--report` 的 `fuzzy` 列表复核）。`--hash-only` 关闭标题搜索：hack/
  汉化合集的标题搜索大多落空，却会消耗稀缺的"未识别 ROM"配额（431）
- zip 包：gamelist 路径始终是磁盘上的 `.zip` 文件；内部文件名只用于 API 查询
  （且只取 basename，不会被当作文件系统路径）。arcade 目录的 zip 视为 set，不解包。
  描述文件中的路径按描述文件所在目录解析（包括多碟 `.m3u` 放在 `.hidden/` 或子目录中的光盘），
  一张光盘/一组多碟只算一个游戏。
- `--rename` —— 只对**哈希精确匹配**的 ROM 生效（文件名搜索的模糊结果不改名），
  arcade set 与光盘镜像永不改名。目标名优先用 ScreenScraper 返回的规范 ROM 名
  （`rom.romfilename`），否则为刮削标题 + 原文件名中的 `(...)` 标签（保留区域/版本，
  避免不同区域版本撞名），如 `Super Mario Bros (J) [!].nes` → `Super Mario Bros. (J).nes`。
  **单文件 zip**：外部与内部文件同时改名，并按 **torrentzip 规范重打包**——固定 DOS
  时间戳（1996-12-24 23:32）、raw deflate level 9（静态链接 zlib）、EOCD 注释
  `TORRENTZIPPED-<中央目录CRC32大写hex>`；新包写入唯一临时文件并**逐成员解压校验
  CRC/大小通过后**才替换，原文件在此之前不会被删除。**多文件 zip**（cue/bin 等）
  只改容器名，内部文件名不动。目标已存在时跳过，永不覆盖；≥4 GiB 的成员或压缩包按 zip64
  写出（APPNOTE 4.5；中央目录只为溢出的字段写扩展字段）。

## 配额模型（重要）

ScreenScraper 的配额是三维的，全部由每次响应的 `ssuser` 块**动态下发**：

| 维度 | 说明 |
|---|---|
| `maxthreads` | 最大并发（基础 1，贡献者最高 8，捐赠 +1/+5） |
| `maxrequestspermin` | 每分钟上限（注意：官方 FAQ 的 threads×50 公式已过时，实际约为 1024×(threads+1)，以 API 返回为准） |
| `maxrequestsperday` | 每日总量（匿名 1 万 / 注册 2 万 / 赞助 5–10 万），**CET 午夜重置** |
| `maxrequestskoperday` | 每日"未识别 ROM"配额（独立计数，约为总量 1/10，hack/汉化库易触顶） |
| `maxdownloadspeed` | 下载限速（免费 128 KB/s） |

**关键事实**：
- 每下载一张图/视频**各消耗 1 次配额**（1 游戏 6 类媒体 = 7 次请求）
- `retrobox quota` 用 `ssuserInfos.php`，**不消耗每日配额**
- 错误码：429=退避重试（自动）；网络错误/5xx=有限次重试；430=当日耗尽（熔断，整次运行停止并保存进度）；431=未识别配额耗尽；401=服务器饱和；423=API 离线；403=凭据错误（401/403/423/426/430/431 均立即停止，退出码 3）
- 并发：按账号 `maxthreads`（可用 `--threads` 调低）同时处理多个条目，每个请求在整个请求期间占用一个并发槽；每分钟上限由令牌桶控制（低配额账号同样可用）

## 缓存

```
~/.cache/retrobox/
├── api/<sha1>.json     成功响应（TTL 7 天）
├── api/*.miss          未找到标记（TTL 24h，重跑不重复烧未识别配额）
├── datparse2/...       DAT 摘要（完整路径+mtime+size 键控），未改动的 DAT 再次扫描时直接取用
├── ra/hashlibrary.json RA 公开哈希库（TTL 24h）
└── hashes/...          ROM 模式的文件哈希缓存（每个平台目录一个文件）
```

两种模式都使用响应缓存。缓存目录权限 0700、文件 0600，写入为原子操作。ScreenScraper 响应会回显
请求 URL 并在媒体地址中附带凭据；这些凭据参数（如 `ssid`/`sspassword`）在写入缓存前
删除，下载媒体时才重新附加，且只附加到 `screenscraper.fr`（HTTPS）或所配置 API 源的地址上。
早期版本写入的缓存条目在下次读取时自动清除凭据，`retrobox cache clear` 可立即全部删除。
网络错误信息中不会出现请求 URL；重定向不会跨主机跟随。

## RetroAchievements 集成

> 详细错误码排查见 [docs/troubleshooting.md](docs/troubleshooting.md)

三个选项按需叠加，后两个都包含 `--ra` 的效果：

| 选项 | 作用 | 需要 RA 账号 |
|---|---|---|
| `--ra` | 为有成就的游戏写入 `cheevosId`（RA 游戏 ID），前端据此显示成就 | 否 |
| `--ra-only` | 同 `--ra`，且只刮有成就的游戏；没有成就的条目不查询 ScreenScraper、不写入 gamelist；平台中一个有成就的条目都没有时整个平台跳过，不生成 gamelist | 否 |
| `--ra-fill` | 同 `--ra`，且在 ScreenScraper 缺少时用 RA 资料补全发行商/开发商/类型/年份，并以 RA 封面/标题画面作为媒体回退 | `RA_API_KEY` |

匹配方式：retrobox 按 rcheevos 规则计算每个文件的 RA 哈希（见下表），在 RA 哈希库中查到对应
游戏后，再确认该游戏**有成就**——RA 也登记了大量没有成就的游戏（例如 SNES 登记 2831 个，其中
1184 个有成就），这些游戏不写 `cheevosId`，也会被 `--ra-only` 过滤。每个平台的"有成就游戏"
列表在运行开始时加载一次。

`cheevosHash`（RA 哈希）与这三个选项无关：只要 retrobox 能算出正确的 RA 哈希（ROM 模式由文件计算，
DAT 模式取可用的 DAT md5），每个条目都会写入，不论该游戏是否有成就；算不出时不写（不会写错误的哈希）。

数据来源自动选择，结果相同：

- 哈希库：RA 公开哈希库（一次请求覆盖全部平台，缓存 24 小时）；获取失败且配置了 `RA_API_KEY`
  时，改用 RA Web API 按平台获取。
- 有成就游戏列表：配置了 `RA_API_KEY` 时用 Web API，否则用公开接口。
- `--ra-fill` 的资料：Web API 按游戏获取（每个请求间隔 250 ms）。

只有 `--ra-fill` 必须配置 `RA_API_KEY`（[RA 控制面板](https://retroachievements.org/controlpanel.php)
→ Web API Key）；请求只使用该 key，`RA_USERNAME` 可选（仅 `auth show` 显示），不需要 RA 密码。
旧版本的 `--ra-v1` 仍可使用，等同于 `--ra`。

RA 哈希移植自 RetroAchievements 官方参考实现 **rcheevos**（RetroArch/Batocera 在设备上
使用的同一套算法），**覆盖 rcheevos 能计算哈希的全部平台**，并用编译自 rcheevos 源码的参考
工具逐格式核对（119 项对比全部一致；压缩容器样本由 Dolphin 的 `dolphin-tool`、Wiimms ISO
Tools 与 MAME `chdman` 生成，CDI 布局另用 libmirage 交叉核对）：

| 平台 | 规则 |
|---|---|
| NES / FDS | 去掉 16 字节 `NES\x1a` / `FDS\x1a` 头 |
| Atari 7800 / Lynx | 去掉 128 字节 `ATARI7800` / 64 字节 `LYNX` 头 |
| PC Engine / SNES | 分别在 `大小 & 512`、`大小 % 8KiB == 512` 时去掉 512 字节头 |
| Nintendo 64 | `.v64`/`.n64` 先转换为 `.z64` 字节序 |
| NDS / DSi | 头部 + arm9 + arm7 + 图标块 |
| arcade（console 27） | 集合名 md5，如 `md5("mslug")`；FBNeo 子系统目录（`fbneo/nes/x.zip`）为 `md5("nes_x")`；`.neo` 为 4KiB 头之后的内容 |
| PlayStation / PS2 | `SYSTEM.CNF` 引导文件名 + 可执行文件内容 |
| PSP | `PARAM.SFO` + `EBOOT.BIN`（`.pbp` 为整个文件） |
| Sega CD / Saturn | 第 0 扇区前 512 字节 |
| PC Engine CD / PC-FX / Neo Geo CD / Dreamcast / 3DO / Jaguar CD | 各自的引导区/可执行文件规则（Jaguar CD 读取第二个 session；Dreamcast 读 GD-ROM 第 3 轨，自制/MIL-CD 光盘读第一条数据轨） |
| GameCube | 光盘头 + apploader + main.dol 各段 |
| Wii / WiiWare | 光盘头、区域码、分区 TMD 与数据（或解密分区的 dol）；`.wad` 为 TMD + 各内容 |
| Nintendo 3DS | NCSD 头 + NCCH 头（清除加密标志）+ **解密后**的 ExeFS；CIA、3DSX、ELF |
| PS3 | `PARAM.SFO` + `EBOOT.BIN`（`.iso`/`.chd` 或 `游戏.ps3/` 文件夹） |
| MS-DOS / Arduboy | zip 目录项（小写文件名 + CRC + 大小，排序）；`.dosz` 父包与 `.dosc`；`.hex` 按文本 |
| 其他受支持平台 | 文件 md5 |

- 支持的镜像/容器：`.cue`/`.bin`、`.gdi`、`.iso`、**`.chd`**（CD、DVD、GD-ROM）、**`.cdi`**
  （DiscJuggler v2/v3/v3.5，含 2048/2336/2352/2448 字节扇区）、`.m3u`（取第一张碟）、PSP
  `.cso`、GameCube/Wii `.iso`/`.gcm`/`.gcz`/`.ciso`/`.wbfs`/**`.rvz`/`.wia`**（Zstandard/
  bzip2/LZMA/LZMA2/PURGE/无压缩）；所有哈希最多计算前 64 MiB（与 rcheevos 相同）。
- Wii 的 `.rvz`/`.wia` 以"解密且去掉哈希"的形式存储分区；retrobox 按母盘规则重建：逐 2 MiB
  重新计算 H0/H1/H2 哈希树、套用文件中记录的哈希例外，再用文件中**明文保存的分区 title key**
  重新加密——不需要任何主机密钥。注意 wit 默认会 scrub（丢弃"未使用"扇区），这样的 WIA
  与原盘不同，RA 哈希自然也不同；Dolphin 转换（不勾选"删除垃圾数据"）与 `wit --raw` 是无损的。
- **NKit 镜像**（`.nkit.iso`/`.nkit.gcz`）是精简重编码后的光盘，不是原盘，无法得到正确的 RA
  哈希：retrobox 识别后给出警告、不输出哈希（识别方式与 Dolphin 相同）。请先用 NKit 还原为完整镜像。
- **3DS 加密镜像需要你自己的密钥**：零售卡带/CIA 为加密格式，retrobox 不附带任何主机密钥，
  会读取用户提供的 Citra/Azahar 格式 `aes_keys.txt`（及可选 `seeddb.bin`），查找顺序见
  [docs/configuration.md](docs/configuration.md)。已解密（NoCrypto）、固定密钥与自制 3DSX
  镜像无需密钥。没有密钥时加密镜像不输出 cheevosHash（不会给出错误哈希）。
- DAT 模式只在 RA 哈希等于 ROM md5 的平台使用 DAT 中的（无头）md5；光盘平台与 NDS 需要实际
  文件，请用 `scrape roms`。
- 可用 `retrobox hash --system <目录> <文件>` 查看任一文件的 RA 哈希；加 `--explain` 会列出
  使用的规则、选中的轨道与扇区格式、找到的引导文件、容器类型、3DS 密钥来源等每一步：

```
$ retrobox hash --system psx --explain "Game (USA).cue"
  ra hash: fb62be9057d33b4f3d1a4659cead88cb
    - platform 'psx': RetroAchievements console 12, rule Disc
    - Game (USA).cue: track Num(1): 2352-byte sectors, user data at +24, first sector 0 (pregap 0)
    - ISO 9660: SYSTEM.CNF at sector 28, 50 bytes
    - SYSTEM.CNF: BOOT = SLUS_123.45
    - ISO 9660: SLUS_123.45 at sector 22, 11048 bytes
    - hashing 8192 bytes from sector 22
```

```xml
<!-- gamelist.xml 输出（Batocera 原生标签；cheevosHash 为小写 md5） -->
<game>
  <path>./mslug.zip</path>
  <name>Metal Slug - Super Vehicle-001 (1996) SNK</name>
  <cheevosId>11750</cheevosId>
  <cheevosHash>b43c8b4ec999588c04dad79bb8bcc745</cheevosHash>
</game>
```

刮削后 Batocera ES 直接显示成就图标，无需再跑 "Index Games"。

## 输出结构

```
roms/<platform>/
├── gamelist.xml          路径为 ./<rom>、./<subdir>/<media>（合并写入）
├── .retrobox-state.json  断点续跑状态（删除即重置）
├── images/               box-2D→<image>、ss→<thumbnail>、ss-title→<titleshot>、
│                         wheel→<wheel>（无 marquee 时同时写 <marquee>）、
│                         box-2D-back→<boxback>、fanart→<fanart>
├── marquees/             marquee→<marquee>
├── videos/               video→<video>
└── manuals/              manuel→<manual>
```

媒体文件以游戏在磁盘上的路径命名（`<rom 文件名去扩展名> - box.png`，子目录中的游戏为
`子目录_<文件名> - box.png`），不同区域版本、不同子目录的同名游戏都不会共用图片。同一平台目录内相同内容（sha1 或字节哈希一致）的媒体只存一份；去重不跨平台目录，
因此每个平台的引用都指向本目录下真实存在的文件。单个媒体下载失败只跳过该类型。

## 支持的平台

retrobox 的平台与 Batocera 一致（约 150 个可刮削平台：主机、掌机、电脑、街机与弹珠台；游戏引擎移植、
工具与媒体浏览器不在其中）。`retrobox systems list` 列出全部平台目录与 ScreenScraper 系统 ID（含别名
目录，如 `ps1` 之于 `psx`、`segacd` 之于 `megacd`）；`--json` 输出包含每个目录的完整信息。每个平台的
属性集中在一张注册表（`src/systems/table.rs`）中：

| 属性 | 用途 |
|---|---|
| 目录名（规范名 + 别名） | 规范名就是 Batocera 的 `roms/` 目录名，DAT 模式写入规范目录；别名与规范目录完全等价 |
| ScreenScraper 系统 ID | 与 Batocera 自带刮削器一致；ScreenScraper 有更具体的子系统时用子系统（如 Naomi 2、Amiga AGA） |
| RetroAchievements 主机 ID | 与 Batocera 一致；街机家族（`mame`/`fbneo`/`naomi`…）与 Neo Geo 为 RA 主机 27 |
| 文件类型 | 卡带/电脑镜像、光盘镜像、街机 romset：决定查询方式、RA 哈希来源与 `--rename` 是否安全 |
| 扩展名 | `scrape roms` 扫描的文件类型，与 Batocera 的列表相同；未登记的目录使用 `zip`/`bin`/`rom` |
| DAT 名称关键词 | `scrape dat` 与 `dats scan` 由 DAT 头部名称识别平台 |

注册表与 Batocera 源码、ScreenScraper 系统表逐项核对过，新增与改动的 ScreenScraper ID 另经实际查询确认。某个目录对应的 RA 主机与哈希规则可用 `retrobox hash --explain --system <目录> <文件>` 查看
（说明的第一条，形如 `platform 'psx': RetroAchievements console 12, rule Disc`）。

## 退出码

| 码 | 含义 |
|---|---|
| 0 | 全部成功 |
| 1 | 部分未匹配（有条目未刮到或查询失败） |
| 2 | 参数/环境错误 |
| 3 | 配额耗尽 / 凭据被拒 / API 关闭或离线，提前终止（进度与 gamelist 已保存，可 `--resume` 续跑） |
| 130 | Ctrl-C 中断（第一次 Ctrl-C 会完成进行中的条目并保存进度；再按一次立即退出） |

`auth check` 同样遵循：凭据被拒为 3，无法检查（网络等）为 2。`quota --watch` 以 Ctrl-C 结束时为 130。

## 发布产物

每个版本的 GitHub Release 包含完全静态的 Linux x86_64 二进制 `retrobox`、`SHA256SUMS` 与 CycloneDX
SBOM（`retrobox.cdx.json`，列出编译进程序的全部依赖）。下载后可校验：

```bash
sha256sum --check --ignore-missing SHA256SUMS
```

Release 还附有 `THIRD-PARTY-NOTICES.txt`（编译进程序的第三方库许可证）。`retrobox -U` 在替换前自动执行 SHA-256 校验。预编译版本免费使用；源代码暂未公开，问题与建议请在
GitHub Issues 中反馈。

## 试用与反馈

retrobox 仍在持续开发中，欢迎试用、测试并反馈结果。遇到问题、匹配错误或遗漏、无法识别的平台，或有
改进建议，请在本仓库的 Issues（<https://github.com/rshi0212/RetroBoxCli/issues>）中提交。有帮助的
信息包括：retrobox 版本（`retrobox --version`）、执行的命令、平台目录与相关输出（加 `-v` 可得到更多
细节；输出中不会包含凭据，提交前仍请检查一遍）。哈希相关的问题可附上
`retrobox hash --explain --system <目录> <文件>` 的输出。

## API 致谢

数据来自 [ScreenScraper](https://www.screenscraper.fr)（CC BY-NC-SA 4.0）与
[RetroAchievements](https://retroachievements.org)。ScreenScraper API 仅限
免费软件使用；请遵守两者的使用条款，贡献数据/赞助以提升配额。