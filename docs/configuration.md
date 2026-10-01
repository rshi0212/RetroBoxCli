# 配置参考

## 配置优先级

- 显式 `--env-file <path>`：文件中的变量**覆盖**已存在的环境变量。
- 未指定时按 `./.env` → `~/.config/retrobox/.env` 取第一个存在的文件，加载在环境变量**之下**
  （已 export 的变量优先）；文件格式错误会给出警告。
- 凭据没有命令行参数（避免进入 shell 历史与 `ps`）。

`auth show` 只显示账号名前两个字符，密码/密钥一律显示为 `***`。

## .env 完整参考

| 变量 | 必需 | 说明 |
|---|---|---|
| `SS_SSID` | 建议 | ScreenScraper 会员名，决定配额等级 |
| `SS_SSPASSWORD` | 否 | 会员密码。匿名配额只有注册用户的 1/2 |
| `RA_USERNAME` | 否 | RetroAchievements 用户名，仅在 `auth show` 中显示，请求不使用 |
| `RA_API_KEY` | `--ra-fill` 需要 | RA Web API key（[控制面板](https://retroachievements.org/controlpanel.php) → Web API Key）。`--ra-fill` 必需；`--ra`/`--ra-only` 不需要，配置后"有成就游戏列表"改由 Web API 获取，公开哈希库不可用时也由 Web API 按平台补上。不需要 RA 密码 |

### 配额等级参考

| 等级 | 每日请求 | 线程 |
|---|---|---|
| 匿名 | 10,000 | 1 |
| 注册会员 | 20,000 | 1 |
| 数据库贡献者 | 20,000+ | 最高 8 |
| 捐赠（5€/月） | 100,000 | +1 |
| 捐赠（10€/月） | 100,000 | +5 |

实际数值以每次响应的 `ssuser` 块为准——retrobox 自动适配，无需手动配置。

## 运行时目录

| 目录 | 用途 | 覆盖方式 |
|---|---|---|
| `~/.config/retrobox/dats/` | DAT 文件：需从 No-Intro、Redump 等网站手工下载后放入（retrobox 目前不能自动下载或更新 DAT） | `--dat-dir` |
| `~/.cache/retrobox/` | API 响应缓存 + DAT 摘要 + RA 哈希库 + ROM 文件哈希（0700/0600 权限；`--no-cache` 全部关闭） | `--cache-dir`（scrape 与 `cache stats/clear`）/ `$RETROBOX_CACHE_DIR` |
| `roms/<platform>/` | 输出目标（gamelist.xml + 媒体） | `--roms-root` |
| `roms/<platform>/.retrobox-state.json` | 断点续跑状态（每次联网运行都会记录，`--resume` 时跳过已记录条目；更改影响输出的选项会自动重置：媒体类型、图片尺寸、字段、区域优先级、RA 选项，DAT 模式的 DAT 与 `--no-clones`，ROM 模式的 `--rename`/`--hash-only`） | 删除即重置 |
| `roms/<platform>/gamelist.xml.bak` | 首次改写非 retrobox 生成的 gamelist 时的备份 | — |

## 3DS 密钥（可选）

零售 3DS 镜像（`.3ds`/`.cci`/`.cia`）是加密的，计算 RA 哈希需要主机密钥。retrobox **不附带
任何密钥**，只读取你自己的 Citra/Azahar 格式 `aes_keys.txt`（`slot0x2CKeyX=`、`slot0x25KeyX=`、
`slot0x18KeyX=`、`slot0x1BKeyX=`、`slot0x3DKeyX=`、`common0=`…`common5=`），种子加密的游戏另需
同目录的 `seeddb.bin`。查找顺序：

1. 环境变量 `RETROBOX_3DS_KEYS`（文件路径）
2. `~/.config/retrobox/aes_keys.txt`
3. `~/.local/share/azahar-emu/sysdata/aes_keys.txt`、`~/.local/share/citra-emu/sysdata/aes_keys.txt`

未找到密钥时，加密镜像不输出 cheevosHash；已解密、固定密钥与 3DSX 镜像不受影响。

其他平台都不需要密钥：Wii 的 `.rvz`/`.wia` 文件自带分区的 title key，retrobox 用它重建加密
数据，不需要也不读取 Wii 通用密钥。

## 输入大小上限

防止损坏或恶意输入耗尽内存（超过即报错/跳过，不会部分读取）：

| 对象 | 上限 |
|---|---|
| ScreenScraper API JSON 响应 | 32 MiB（嵌套深度由 serde_json 限制为 128 层） |
| 媒体文件（图片/视频/PDF） | 512 MiB |
| RetroAchievements 响应 | 64 MiB |
| DAT 文件 | 2 GiB、200 万条目 |
| 现有 gamelist.xml | 256 MiB |
| `.cue`/`.gdi`/`.m3u` | 1 MiB |
| CHD | hunk ≤ 16 MiB、≤ 400 万 hunk，映射表必须位于文件内 |
| GCZ / CSO / WIA / RVZ | 块 ≤ 64 MiB（CSO ≤ 16 MiB），表 ≤ 64 MiB，解压输出不超过声明大小；Wii 分区 ≤ 64 个 |
| CDI | 描述符 ≤ 1 MiB、≤ 99 个 session/轨道，轨道数据必须位于描述符之前 |
| GameCube/Wii/WAD/3DS 结构 | 声明的数据段必须位于文件内（否则视为损坏，不输出哈希） |
| RA 哈希计算范围 | 每个文件前 64 MiB（与 rcheevos 一致） |

测试/代理用环境变量：`RETROBOX_SS_BASE_URL`（ScreenScraper API 根）、
`RETROBOX_RA_BASE_URL`（RetroAchievements 根）、`RETROBOX_UPDATE_BASE_URL`（`retrobox -U`
使用的 GitHub API 根，默认 `https://api.github.com`）。日志级别：`RUST_LOG` 优先，
否则 `-q`=warn、默认 info、`-v`=debug、`-vv`=trace。

## 退出码

| 码 | 含义 | 常见原因 |
|---|---|---|
| 0 | 成功 | — |
| 1 | 有未匹配 | 部分 DAT 条目/ROM 在 SS 不存在或查询失败（检查 `--ra-only` 是否过严） |
| 2 | 环境错误 | 凭据缺失、目录不存在、参数错误、`--ra-only` 时 RA 索引加载失败、`-U`/`--update-to` 更新失败 |
| 3 | 提前终止 | 430 当日配额耗尽 / 403 凭据被拒 / 431 未识别配额耗尽 / 401 非会员关闭 / 423 离线 / 426 拉黑；进度已保存，剩余平台不再处理 |
| 130 | Ctrl-C | 进行中的条目完成后保存进度并写 gamelist，之后可 `--resume` |
