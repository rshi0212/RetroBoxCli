# 故障排查

## 安装与更新

### `retrobox: command not found`

安装目录不在 `PATH` 中。`echo $PATH` 确认是否包含 `~/.local/bin`；不包含时重新登录，或在
`~/.bashrc` 中加入 `export PATH="$HOME/.local/bin:$PATH"`。见 [安装文档](installation.md)。

### `no write permission in /usr/local/bin — re-run with the permissions that installed it`

可执行文件所在目录对当前用户不可写。以安装时的权限执行：`sudo retrobox -U`。

### `GitHub API refused the request (HTTP 403 ...)`

未认证的 GitHub API 限制为每个 IP 每小时 60 次请求，共享出口 IP 的网络更容易触发。稍后重试，
或按[安装文档](installation.md#安装发布版二进制)中的 curl 命令直接下载覆盖。

### `no published release found` / `release vX.Y.Z not found`

GitHub 上没有可见的正式发布，或指定的 tag 不存在（tag 形如 `v2026.10.01-7a20a85`，见发布页）。
`-U` 只查找正式版本（不含草稿）。

### `checksum mismatch ... nothing was changed`

下载内容与 `SHA256SUMS` 不一致（下载被代理改写或中断）。原文件未修改，重新执行 `retrobox -U`；
持续出现时用 `curl` 手动下载并按[安装文档](installation.md#校验下载文件可选)校验。

### `... is managed by cargo` / `... is a build inside a source tree`

通过 `cargo install` 安装或在源码目录中构建的 retrobox（开发版本）不会被自更新替换，按提示中的命令
更新；需要改用发布版时，删除该文件后按安装文档下载。

## 凭据与准入

### `403 Erreur de login : Vérifier vos identifiants développeur !`

ScreenScraper 拒绝了程序级的访问凭据（与你的会员账号无关）。请先更新到最新版本（`retrobox -U`）；
更新后仍出现时，请在 GitHub Issues 中反馈。

### `403 Erreur de login : ... identifiants utilisateurs !`

会员凭据（`SS_SSID`/`SS_SSPASSWORD`）错误。清除这两行可回退匿名模式。

### `426 Le logiciel de scrape utilisé a été blacklisté`

ScreenScraper 拒绝了本程序的请求标识。请更新到最新版本（`retrobox -U`）；仍出现时请在 GitHub Issues
中反馈。

## 配额类

### `430 Votre quota de scrape est dépassé`

当日配额耗尽。retrobox 已自动熔断：停止派发新请求，已完成的结果合并写入
gamelist，续跑状态已保存，退出码 3。次日用 `--resume` 从断点继续。其他选项：
- 等到 **CET 午夜** 重置（`retrobox quota` 可看剩余量）
- `--media none` 或减少媒体种类（每图/视频各耗 1 配额）
- `--ra-only` 过滤掉不需要的条目
- `--offline` 生成无媒体骨架
- 贡献数据库或[捐赠](https://www.screenscraper.fr)提升等级

### `431 Faite du tri dans vos fichiers roms`

"未识别 ROM" 配额耗尽（总量约 1/10）。hack/翻译/汉化 ROM 极易触顶。
清理无法识别的文件，或减少无效查询。retrobox 把 431 与 430 分开报告。

### `401 API fermé pour les non membres`

服务器 CPU > 60% 时对非会员关闭。retrobox 立即停止（退出码 3，进度已保存），稍后
`--resume` 重试；注册会员可缓解。423（API 离线）同理。

## 匹配类

### 纯 DAT 模式生成的 gamelist 路径与磁盘文件名不一致

纯 DAT 模式的路径只取自 DAT：头部声明 `forcepacking="zip"`（FinalBurn Neo）时为 `<游戏名>.zip`，
否则为 DAT 中的 ROM 文件名（如 `Game (USA).gb`）。合集的文件命名与 DAT 不同（如逐个压缩的
No-Intro 合集 `Game (USA).zip`）时，路径不会对应；这类合集用 `scrape roms`，它按磁盘文件匹配。

### `dats scan` 显示 `<skipped: …>` 或 `<unmapped>`

`<skipped: 原因>`：DAT 来源已识别，但内容不是可刮削的游戏集合（FinalBurn Neo 的 `hdd`/`samples`、
MAME CHD DAT 与软件列表、Redump 街机基板光盘、No-Intro 采样数据）。`<unmapped>`：头部名称不对应
任何已知平台（例如 Visual Pinball、Xbox、PC-98）；需要时用 `--dat-file <dat> --system <dir>` 指定。
平台判定规则见 [README · DAT 来源与平台目录](../README.md#dat-来源与平台目录)。

### `fds` 的 gamelist 同时有 `.fds` 与 `.qd` 条目（或 MAME/FBNeo 同时用了多个存储方式的 DAT）

同一平台的不同转储格式（`(FDS)`/`(QD)`、`(Headered)`/`(Headerless)`、N64 字节序等）与 MAME、FinalBurn
Neo 的不同存储方式（`(merged)`/`(split)`/`(non-merged)`）是并列的，没有优先级：一个文件属于哪一种只能由
它的校验值判断，因此目录中有的 DAT 全部使用，运行时输出 "… equal alternatives, all are used" 或
"… are the same version of …: all are used"。MAME/FinalBurn Neo 只使用最新版本的 DAT，旧版本不论哪种
存储方式都跳过。只需要一种格式或存储方式时，DAT 目录中只放对应的 DAT 即可；只放一种、或全放，结果
都正确。

### `nds`/`3ds` 的 gamelist 包含 DSi / New 3DS 游戏

No-Intro 把 Nintendo DSi、New Nintendo 3DS 列为独立平台，Batocera 分别放在 `nds`、`3ds` 目录，
因此两个 DAT 都会使用，运行时输出 "… are different platforms that share this dir"。同一游戏若
两个 DAT 都有，只写一次。

### 运行时提示 "skipped … (older version of the same DAT)"

目录中有同一 DAT 的多个版本（例如三个日期不同的 Game Boy DAT），只用最新的一个；规则见
[README · DAT 来源与平台目录](../README.md#dat-来源与平台目录)。不想看到提示时，只保留最新的 DAT 即可。

### 升级后 Sega CD、Odyssey 2、VIC-20、Amiga 的 gamelist 写到了另一个目录

DAT 模式写入平台的规范目录，规范目录现在与 Batocera 的 `roms/` 目录一致：Sega CD 为 `megacd`
（原 `segacd`）、Odyssey 2 为 `odyssey2`（原 `o2em`）、VIC-20 为 `c20`（原 `vic20`）、Amiga 为
`amiga500`（原 `amiga`）。Batocera 只读取新目录；以前写到旧目录的 `gamelist.xml` 与媒体可以移到
新目录后再运行一次（合并写入会保留已有记录）。刮削时若旧目录中还有 `gamelist.xml`，retrobox 会输出一行
`[note]` 指出它；`--system segacd` 等旧名称仍然可用（DAT 模式写入对应的新目录）。ROM 模式按磁盘上
实际的目录工作，旧目录名仍然可用，但会提示 Batocera 读取的是哪个目录（其他前端可能有意使用旧名称，
因此只是提示）。

### FinalBurn Neo 的 nes/megadrive 等子平台游戏

这些游戏属于 `roms/fbneo/<子平台>/`（如 `fbneo/nes/x.zip`），不放在 `roms/nes/`。它们与其他 fbneo romset
一样，在 ScreenScraper 的 75 号系统下按集合文件名查询，RA 哈希按 rcheevos 规则计算（`md5("nes_x")`）；
查到什么、查不到，都是 ScreenScraper 与 RA 当前的数据，retrobox 原样记录。小众子平台大多查不到属于
正常情况，数据补充后重新刮削即可匹配。

### 大量 404 未找到

1. 运行 `retrobox dats scan` 确认 DAT 识别出了正确平台
2. 确认 ScreenScraper 系统映射正确（`retrobox systems map <dir>`）
3. zip ROM：retrobox 用**内部最大文件**的哈希——确认 zip 内是完整 ROM
4. hack/翻译版本可能不在库中——这正是 431 配额的消耗来源
5. 运行时提示 `DAT(s) name no known platform and were skipped`：DAT 头部名称不对应已知平台，
   用 `--dat-file x.dat --system <dir>` 强制指定平台
6. 加 `--report report.json` 运行一次，`missing`/`failed`/`fuzzy` 列表就是需要人工处理的清单
7. 未找到的游戏仍在 `gamelist.xml` 中（名称取 RA 标题、DAT 名称或文件名，没有 `ssid`），RA 结果照常
   写入；ScreenScraper 收录后用 `--only-new` 重跑即可补全资料

### 按标题搜索匹配错了游戏

ROM 模式在哈希查不到时会按文件名解析出的标题搜索，并取 ScreenScraper 排名第一的结果——
这一步可能匹配错。`--report` 的 `fuzzy` 列表逐条列出这类匹配（文件名 → 匹配到的标题与
ssid），核对后手工修正即可。hack/汉化合集建议直接加 `--hash-only`：只接受哈希精确匹配，
不做标题搜索，也不会消耗"未识别 ROM"配额（431）。

### RA 匹配不上

1. 先用 `retrobox hash --system <目录> --explain <文件>` 看 retrobox 算出的 RA 哈希及其
   计算过程（规则、轨道、引导文件、容器、密钥），与 RetroAchievements 网站该游戏
   "Supported Game Files"中的哈希对比。算法移植自 rcheevos
   （RetroArch/Batocera 使用的同一实现），卡带、光盘（cue/bin、gdi、iso、chd、cdi、m3u）与
   GameCube/Wii 容器（gcz/ciso/wbfs/rvz/wia）规则一致
2. 显示 `<not recognized>`：文件不是该平台的有效镜像（如 Saturn 目录里放了 PS1 光盘），或是
   下列需要额外条件的情况：
   - 加密的 3DS 镜像但没有找到 `aes_keys.txt`（见 docs/configuration.md「3DS 密钥」）；
   - NKit 镜像（`.nkit.iso`/`.nkit.gcz`，刮削时会有 `[warn] ... NKit image` 提示）：先用 NKit
     还原为完整镜像；
   - 损坏或不完整的镜像（声明的数据段超出文件范围）；
   - `.7z`、`.rar` 等 retrobox 无法打开的压缩包：RA 哈希针对其中的内容，retrobox 不会写出压缩包本身
     的（错误）哈希，`--explain` 会注明；需要成就标识时请解压（或改用 zip）。
   显示 `<unsupported platform>`：该平台在 RetroAchievements 没有对应主机，或 rcheevos 本身也不计算哈希
3. 哈希算出来了但与 RA 网站不同：
   - Wii/GameCube 的 `.wia`/`.rvz` 若由 wit 以默认模式（scrub）或 Dolphin 勾选"删除垃圾数据"
     转换，未使用的扇区已被丢弃，内容与原盘不同——这是转换造成的，请用原始 `.iso` 或无损转换
     （Dolphin 默认、`wit copy --raw`）重新生成；
   - Dreamcast `.cdi` 多为自制的 selfboot 转换版：转换时通常会修改 IP.BIN 或重排、压缩数据，
     内容与 GD-ROM 原盘不同。RA 网站没有登记该版本时，哈希对不上是正常的，请改用 GDI/CHD 原盘
4. DAT 模式无法得到光盘平台与 NDS 的 RA 哈希（需要实际文件）——对这些平台用 `scrape roms`
5. arcade = 集合名 md5（如 `md5("mslug")`）；FBNeo 子系统目录下为 `md5("nes_x")` 形式
6. 游戏在 RA 登记了但没有成就：`hash` 能算出哈希、RA 网站也能查到该游戏，但 gamelist 中只有
   `cheevosHash` 没有 `cheevosId`，`--ra-only` 也会跳过它——这是预期行为，没有成就的游戏不需要
   成就标识
7. RA 刚登记的新哈希：公开哈希库缓存 24 小时，`retrobox cache clear` 后重跑即获取最新数据

### `[warn] ... the disc image is corrupt or unsupported (decoder error)`

该光盘镜像（通常是损坏或非标准的 CHD）无法解码，retrobox 跳过它的 RA 哈希并继续刮削其余
游戏。可用 `chdman verify -i <文件>` 检查镜像完整性。

## 刮削

### ROM 模式没有扫描到某些文件

每个平台扫描的扩展名与 Batocera 相同（`retrobox systems list --json` 的 `extensions`），Batocera 不显示
的文件也不刮削——例如 `psx` 目录中没有被 `.cue` 引用的单独 `.bin`、`nes` 目录中的 `.fds`（属于
`fds` 平台）。被描述文件引用的文件（`.cue`/`.toc` 的轨道、`.gdi`、`.m3u` 的各张碟、`.ccd` 的
`.img`/`.sub`、`.mds` 的 `.mdf`）不单独刮削，以描述文件为准。

### 只想补刮新加入的游戏

加 `--only-new`：gamelist.xml 中已有 ScreenScraper id（`<ssid>`）的游戏会被跳过，ROM 模式下
这些文件连哈希都不计算。与 `--resume` 不同，它不依赖上次运行的选项，适合长期维护的库。

### ROM 模式第二次扫描仍然很慢

文件哈希缓存按"相对路径 + 大小 + 修改时间"校验；复制/解压会改变修改时间，导致重新计算。
`.cue`/`.gdi`/`.m3u` 与 PS3 文件夹游戏不缓存（它们的 RA 哈希依赖其他文件）。确认没有使用
`--no-cache`。

## gamelist.xml

### 目录中出现 `gamelist.xml.corrupt`

现有 gamelist.xml 无法安全解析（不是 `<gameList>` 根元素、含 XML 不允许的控制字符、元素名
非法等）。retrobox 不会在其基础上合并以免写出损坏文件，而是把它改名为 `.corrupt` 后重新生成。
可手工修复后改回原名再运行。

### 目录中出现 `gamelist.xml.bak`

首次改写不是 retrobox 生成的 gamelist 时自动保留的原文件备份，确认无误后可删除。

## TorrentZip

### 重打包后校验工具报不一致

retrobox 使用静态编译的 zlib 以 raw deflate、level 9 压缩，写完后逐成员解压校验
CRC 与大小，校验通过才替换原文件。与其他 torrentzip 实现的字节级一致性取决于双方
zlib 版本与参数，未针对官方种子做逐字节对照测试；若需要做种，请用 trrntzip/RomVault
复核。多文件 zip 只改容器名，不重打包。

≥4 GiB 的成员按 zip64 写出（APPNOTE 4.5，已用 Info-ZIP、7-Zip、Python 验证可正常解压）。
TorrentZip 对 zip64 的字节级约定未与 RomVault 逐字节对照，这类大包的 torrentzip 校验值
可能与其他工具不同。

### 带非法字符的文件名

`/ \ : * ? " < > |` 和控制字符统一替换为 `_`。RA/SS 标题里的特殊字符
（如 `Metal Slug - Super Vehicle-001`）保持原样。

## 性能

### 大库跑得慢

1. `retrobox quota` 看线程数——贡献者账号可到 8 线程；retrobox 按该值并发
   （`--threads` 可调低）
2. 响应缓存命中时重跑接近瞬时（`retrobox cache stats` 查看）
3. `--offline` 先出骨架，媒体分几天补
4. MAME 全集建议先 `--no-clones` 减半请求数

## 常见问题

**转换 GameCube/Wii 镜像时怎样才不影响 RA 哈希？**
Dolphin 转换 RVZ/WIA 时不要勾选"删除垃圾数据"（Remove Junk Data）；wit 要加 `--raw`。
这两种方式都是无损的，retrobox 能还原出与原盘相同的哈希。scrub、NKit 这类精简方式会
改动数据。

**Wii 的 `.rvz` 需要 Wii 密钥吗？**
不需要。RVZ/WIA 文件本身明文保存了分区的 title key，retrobox 用它重建加密数据。只有加密的
3DS 镜像需要用户自己的 `aes_keys.txt`。

**Dreamcast 的 `.cdi` 哈希与 RA 不符？**
大多数 CDI 是自制的 selfboot 版本，内容与 GD-ROM 原盘不同。retrobox 对 CDI 的读取方式与
flycast 相同（flycast 是为 CDI 计算 RA 哈希的模拟器）；如果 RA 只登记了原盘的哈希，请改用
GDI/CHD。

**同一个游戏在 RA 网站上有多个哈希？**
不同地区、修订版和容器格式各有各的哈希（CHD 与 cue/bin 相同，因为哈希针对的是内容）。
只要 retrobox 的结果与其中任意一个一致即可。

**`hash` 显示 `<unsupported platform>`，但 RA 有这个平台？**
该目录名没有映射到 RA 主机，或者 rcheevos 对该主机不计算文件哈希。`retrobox hash --explain
--system <目录> <文件>` 的第一条说明显示该目录对应的 RA 主机与哈希规则；没有这条说明表示目录未映射到
RA 主机（缺少的映射欢迎在 GitHub Issues 中反馈）。

**报告里有 `missing_media`，是下载失败了吗？**
可能是 ScreenScraper 没有该类媒体，也可能是下载失败（日志中会有 `media download failed`
警告）。不带 `--resume`/`--only-new` 重新刮削该平台会补下缺失的文件（这两个选项会跳过已
匹配的条目）；已存在的媒体文件不会重复下载。

