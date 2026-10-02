# 安装与更新

retrobox 以单个可执行文件发布，下载后放入 `PATH` 中的目录即可使用，无需安装运行库。预编译版本
免费使用；源代码暂未公开。

## 平台

| 平台 | 发布产物 | 说明 |
|---|---|---|
| Linux x86_64 | `retrobox` | musl 完全静态链接，不依赖 glibc 版本，适用于 Ubuntu、Debian、Fedora、Arch 等发行版 |
| 其他平台（aarch64、macOS 等） | — | 目前没有预编译版本 |

每个版本的 GitHub Release 包含以下文件：

| 文件 | 内容 |
|---|---|
| `retrobox` | Linux x86_64 可执行文件 |
| `SHA256SUMS` | `retrobox`、`retrobox.cdx.json` 与 `THIRD-PARTY-NOTICES.txt` 的 SHA-256 校验和 |
| `retrobox.cdx.json` | CycloneDX 格式的 SBOM（依赖清单） |
| `THIRD-PARTY-NOTICES.txt` | 编译进程序的第三方库的许可证 |

最新版本的固定下载地址为：

```
https://github.com/rshi0212/RetroBoxCli/releases/latest/download/retrobox
```

指定版本的下载地址为 `https://github.com/rshi0212/RetroBoxCli/releases/download/<tag>/retrobox`，
例如 `.../download/v2026.10.01-7a20a85/retrobox`。

版本号为 `年.月.日-提交哈希`（如 `2026.10.01-7a20a85`）：构建所用源码提交的日期（UTC）与该提交哈希的
前 7 位；`retrobox --version` 输出同一版本号，tag 为 `v` 加版本号。

## 安装发布版二进制

以下命令以 Ubuntu 为例，其他 Linux 发行版相同。

### 安装到用户目录（不需要 root）

```bash
mkdir -p ~/.local/bin
curl -L https://github.com/rshi0212/RetroBoxCli/releases/latest/download/retrobox -o ~/.local/bin/retrobox
chmod a+rx ~/.local/bin/retrobox  # 设置可执行权限
```

Ubuntu 的默认 `~/.profile` 会在登录时把已存在的 `~/.local/bin` 加入 `PATH`。目录是刚创建的，
需要重新登录，或在当前 shell 中执行：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 安装到系统目录（所有用户可用）

```bash
sudo curl -L https://github.com/rshi0212/RetroBoxCli/releases/latest/download/retrobox -o /usr/local/bin/retrobox
sudo chmod a+rx /usr/local/bin/retrobox  # 设置可执行权限
```

### 使用 wget

系统中没有 `curl` 时可用 `wget` 代替（或先执行 `sudo apt install curl`）：

```bash
wget https://github.com/rshi0212/RetroBoxCli/releases/latest/download/retrobox -O ~/.local/bin/retrobox
chmod a+rx ~/.local/bin/retrobox
```

安装到 `/usr/local/bin` 时在两条命令前加 `sudo`。

### 确认安装

```bash
retrobox --version
retrobox doctor        # 检查凭据、API 连通性、DAT 目录与缓存目录
```

配置凭据见 [README 快速开始](../README.md#快速开始) 与 [配置参考](configuration.md)。

### 校验下载文件（可选）

`SHA256SUMS` 可校验下载文件的完整性：

```bash
cd ~/.local/bin
curl -LO https://github.com/rshi0212/RetroBoxCli/releases/latest/download/SHA256SUMS
sha256sum --check --ignore-missing SHA256SUMS && rm SHA256SUMS
```

`retrobox.cdx.json`（CycloneDX SBOM）列出编译进程序的全部依赖及其版本。

### Shell 补全（可选）

```bash
mkdir -p ~/.local/share/bash-completion/completions
retrobox completion bash > ~/.local/share/bash-completion/completions/retrobox
```

zsh、fish 的写法见 `retrobox completion --help`。

## 更新

### `retrobox -U`

```bash
retrobox -U            # 更新到最新版本
sudo retrobox -U       # 安装在 /usr/local/bin 等需要 root 写权限的目录时
```

输出示例：

```
Current version: 2026.09.27-1a2b3c4
Release version: 2026.10.01-7a20a85 (v2026.10.01-7a20a85)
Downloading https://github.com/rshi0212/RetroBoxCli/releases/download/v2026.10.01-7a20a85/retrobox
Updated retrobox 2026.09.27-1a2b3c4 -> 2026.10.01-7a20a85 (/home/user/.local/bin/retrobox)
```

已是最新版本时输出 `retrobox is up to date (2026.10.01-7a20a85)`，不修改任何文件。

### 安装指定版本

`--update-to <TAG>` 安装指定 tag 的版本，可用于降级或固定到某个版本；`v` 前缀可省略：

```bash
retrobox --update-to v2026.10.01-7a20a85
retrobox --update-to 2026.10.01-7a20a85
```

`-U` 不会安装日期更早的版本；同一天的另一个构建视为最新发布，会被安装。`--update-to` 安装给定的
版本，无论新旧。`-U` 与 `--update-to`
不能与子命令同时使用。

### 更新过程

1. 通过 GitHub REST API 读取发布信息：`-U` 读取 `releases/latest`，`--update-to` 读取对应 tag；
2. 比较发布版本与当前版本（`retrobox --version`）：按日期比较，版本号相同或发布版本日期更早时不更新；
3. 下载 `SHA256SUMS`，再把 `retrobox` 下载到可执行文件所在目录下的临时文件
   （`.retrobox.update-<pid>`），下载时同步计算 SHA-256；
4. 校验和一致后，沿用原文件的权限位，用 rename 原子替换原可执行文件。

以下情况不修改原文件，并删除临时文件：

- 发布中没有 `SHA256SUMS`，或其中没有 `retrobox` 条目；
- 下载内容的 SHA-256 与 `SHA256SUMS` 不一致；
- 网络错误、HTTP 错误或下载内容为空。

更新时替换的是实际执行的文件：`retrobox` 是符号链接时，替换链接指向的目标文件。

### 不支持自更新的安装方式

| 安装方式 | 判断依据 | 更新方法 |
|---|---|---|
| `cargo install` | 可执行文件位于 `~/.cargo/bin/` | 在源码目录中执行 `cargo install --locked --force --path .` |
| 源码树中的构建 | 可执行文件位于 `target/<profile>/` 或 `target/<triple>/<profile>/` | `git pull && cargo build --release` |
| 非 Linux x86_64 平台 | 发布中没有对应二进制 | — |

前两种是从源码构建的开发版本。这些情况下 `retrobox -U` 输出说明并以退出码 2 结束；改用发布版时，
删除该文件后按上文安装即可。

### 网络与环境变量

- 更新只访问 `api.github.com` 与 GitHub Release 的下载地址（会重定向到
  `*.githubusercontent.com`），不发送任何凭据。
- 未认证的 GitHub API 每个 IP 每小时限 60 次请求，超出时返回 HTTP 403/429，稍后重试即可。
- `RETROBOX_UPDATE_BASE_URL` 覆盖 GitHub API 根地址（默认 `https://api.github.com`），
  用于测试或内部镜像。
- 更新成功或已是最新版本时退出码为 0；失败时为 2。

## 卸载

```bash
rm ~/.local/bin/retrobox                 # 或 sudo rm /usr/local/bin/retrobox
```

以下目录保存配置与缓存，如不再需要可一并删除：

| 目录 | 内容 |
|---|---|
| `~/.config/retrobox/` | `.env` 凭据、DAT 文件、3DS 密钥 |
| `~/.cache/retrobox/` | API 响应缓存、DAT 摘要、RA 哈希库、ROM 哈希缓存 |

`roms/<platform>/` 下的 `gamelist.xml`、媒体文件与 `.retrobox-state.json` 由用户自行处理。
