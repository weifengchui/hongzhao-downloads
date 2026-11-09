[English](README.md) | **简体中文**

# 鸿爪 Hongzhao · 伴生进程下载

[![最新版本](https://img.shields.io/github/v/release/weifengchui/hongzhao-downloads?label=companion)](https://github.com/weifengchui/hongzhao-downloads/releases/latest)
[![下载量](https://img.shields.io/github/downloads/weifengchui/hongzhao-downloads/total)](https://github.com/weifengchui/hongzhao-downloads/releases)

鸿爪浏览器扩展的可选本机组件，本仓库只存放它的安装包。源码在私有仓库，因此这里只有已发布的二进制文件与公开的隐私政策。

## 这个仓库是什么

鸿爪是一个本地优先的浏览器扩展：你看过的页面会变成可检索的「痕」，并保存正文快照，原网页下线后依然能读。扩展本体不在本仓库分发。

伴生进程是一个完全跑在你机器上的单文件 Rust 程序，有两项职责：

- **原生消息宿主**（`com.hongzhao.companion`）——接收扩展主动推送的元数据镜像，让扩展能用系统默认程序打开本地文件夹和文件。
- **stdio 的 MCP 服务**——把这份镜像交给 Claude Desktop 这类本地 AI 客户端检索。

不装也能正常使用：浏览、留痕、足迹墙都不受影响，少的是「在本机应用中打开」和「本机 AI 检索痕库」。

伴生进程不读取浏览器的任何数据库，不发起任何网络连接，只看到扩展主动推给它的镜像。

## 环境要求

- **Windows**：Windows 10/11 x64。已发布的安装器编译目标为 `x86_64-pc-windows-msvc`。
- **macOS / Linux**：尚无已发布的安装包，需从伴生进程源码目录安装；若你的平台没有预编译二进制，还需要 Rust。
- **浏览器**：Chrome 116+、Edge、Brave 或 Chromium；Firefox 也可用，但要额外提供附加组件 ID。
- **扩展 ID**：Chrome 在 `chrome://extensions` 打开开发者模式后查看（32 个 `a`–`p` 字母）；Firefox 在 `about:debugging` → 本 Firefox → 「扩展 ID」查看。

## 安装

### Windows

1. 打开 [Releases](../../releases/latest)，下载 `Hongzhao-Companion-Setup.exe`（`companion-v0.2.1` 中为 879,616 字节，约 859 KB）。
2. 双击运行。单个文件，无需解压，也不需要安装 Rust。
3. 安装器会识别已安装的 Chromium 系浏览器，从浏览器配置里反查鸿爪的扩展 ID，把宿主写到 `%LOCALAPPDATA%\Hongzhao`，在同一目录写原生消息清单，并注册 `HKCU\Software\<厂商>\<浏览器>\NativeMessagingHosts\com.hongzhao.companion`。识别不到扩展时，它会提示你粘贴 ID。
4. 完全退出浏览器（所有窗口，包括托盘图标）再重新打开。
5. 在扩展里进入 **设置 → 伴生进程 → 检查连接**，然后点 **同步镜像**。

安装器没有代码签名，Windows SmartScreen 可能提示「未知发布者」，点「更多信息」→「仍要运行」即可继续。运行前请先按下面的方法核对 SHA-256。

这个安装器不注册 Firefox，Firefox 用户请用[下文](#firefox任意平台)的脚本。

### macOS / Linux

目前还没有发布 macOS 安装包：`companion-v0.2.1` 只有 Windows 安装包、宿主二进制和 `SHA256SUMS.txt`。发布配置里为 `Hongzhao-Companion-Setup-macos.zip` 留了位置，但尚未产出。

在伴生进程源码目录中执行：

```bash
cd companion
./install.sh <Chrome扩展ID>
```

macOS 也可以直接双击 `companion/install-macos.command`，它调用的就是同一个脚本。脚本会优先复用 `companion/bin/` 中与你的系统和 CPU 匹配的预编译二进制；没有的话用 `cargo` 编译（需先安装 Rust：<https://rustup.rs>）。

它会往找到的每个 Chromium 系浏览器配置目录写入 `com.hongzhao.companion.json`——Linux 下在 `~/.config/...`，macOS 下在 `~/Library/Application Support/...`，覆盖 Chrome、Chromium、Edge、Brave；传入 Firefox ID 时还会写入 `~/.mozilla/native-messaging-hosts/`。如果一次都没启动过任何 Chromium 系浏览器，脚本会直接退出，不会安装。

### Firefox（任意平台）

Firefox 用的是附加组件 ID，不是 Chrome 扩展 ID，所以传第二个参数：

```bash
./install.sh <Chrome扩展ID> <Firefox附加组件ID>
```

Windows 上：

```powershell
cd companion
.\install-windows.ps1 <Chrome扩展ID> <Firefox附加组件ID>
```

这会写入 `%APPDATA%\Mozilla\native-messaging-hosts\com.hongzhao.companion.json`。

## 校验下载文件

每个 Release 都附带 `SHA256SUMS.txt`。`companion-v0.2.1` 的内容：

```text
b12b1821997e27914b735c7448ef22c98eab5715b6c68de36313c6dd618b5dd2  Hongzhao-Companion-Setup.exe
aba8f338ce5d295b8a09f5fd40a4a867bdbb8b02aa5a944bba85793735934e1c  hongzhao-companion-windows-x64.exe
```

和下载到的文件比对：

```powershell
# Windows PowerShell
Get-FileHash .\Hongzhao-Companion-Setup.exe -Algorithm SHA256
```

```bat
:: Windows，没有 PowerShell 时
certutil -hashfile Hongzhao-Companion-Setup.exe SHA256
```

```bash
# macOS
shasum -a 256 Hongzhao-Companion-Setup.exe

# Linux（两个文件放在当前目录）
sha256sum -c SHA256SUMS.txt
```

## 接入 AI 客户端（MCP）

伴生进程是一个 stdio MCP 服务。`hongzhao-companion mcp` 是显式启动方式，不传参数时 `mcp` 也是默认模式。把它加进客户端已有的 `mcpServers` 配置——以 Claude Desktop 为例：

```json
{
  "mcpServers": {
    "hongzhao": {
      "command": "C:\\Users\\<你>\\AppData\\Local\\Hongzhao\\hongzhao-companion.exe",
      "args": ["mcp"]
    }
  }
}
```

macOS / Linux 用安装脚本最后打印的二进制路径，例如 `/path/to/companion/target/release/hongzhao-companion`。

**可选令牌。** 启动时加 `--token <值>` 或设置环境变量 `HZ_TOKEN`。一旦设了令牌，每次工具调用都必须带一个匹配的 `_token` 参数，否则会被拒绝；客户端传不了这个参数时就不要设。

提问前先同步：**设置 → 伴生进程 → 同步镜像**。镜像最多 20000 条痕；标记为「禁止 AI 使用」的痕和回收站里的痕不会进入镜像。

| 工具 | 作用 |
| --- | --- |
| `traces.search` | 按标题、标签、站点、摘录、笔记检索。参数 `query`，可选 `limit`（1–50，默认 8）。 |
| `traces.get` | 按 `id` 读取一条痕的完整元数据。 |
| `traces.add` | 把一条痕放进待入库队列，扩展下次同步时以 `agent` 来源落库。 |
| `traces.cite` | 给一组 `ids` 生成引用文本（标题、链接、访问日期）。 |

服务使用 MCP 协议版本 `2024-11-05`，按行分隔的 JSON 走 stdin/stdout，服务名为 `hongzhao-companion`。

## 隐私

伴生进程的权限范围刻意收得很窄：

- **只有元数据。** 镜像里是网址、标题、站点、标签、笔记、扩展存的摘录和时间戳；没有正文快照，也没有语义向量。
- **只在本机。** 二进制不打开任何网络连接，也不读浏览器存储；只有扩展往里推，没有反向读取。
- **按需写入。** 只有你点「同步镜像」时才会写 `~/.hongzhao/mirror.json`。设置 `HZ_HOME` 可以换目录。
- **有审计日志。** 每次工具调用往 `~/.hongzhao/audit.jsonl` 追加一行：时间戳、工具名、是否成功、命中数、简短状态说明，不记录页面内容。日志超过 2 MB 时轮转为 `audit.jsonl.1`。
- **可彻底删除。** 删掉 `%LOCALAPPDATA%\Hongzhao`、它创建的注册表项、原生消息清单和 `~/.hongzhao` 即可，远端没有留存任何副本。

## 常见问题

### 设置页提示未检测到本地助手

按顺序排查：

1. **宿主清单没注册。** Windows 下 `%LOCALAPPDATA%\Hongzhao\com.hongzhao.companion.json` 必须存在，且 `HKCU\Software\Google\Chrome\NativeMessagingHosts\com.hongzhao.companion`（Edge / Brave / Chromium 对应键）要指向它。macOS / Linux 下 `com.hongzhao.companion.json` 必须放在浏览器的 `NativeMessagingHosts` 目录里。
2. **扩展 ID 不匹配。** 清单里的 `allowed_origins` 必须正好是 `chrome-extension://<你的ID>/`。用正确的 ID 重跑安装器。
3. **浏览器没完全重启。** 原生消息宿主只在浏览器启动时解析。关掉所有窗口和托盘图标，再重新打开。
4. **Firefox 没传附加组件 ID。** `chrome-extension://` 来源对 Firefox 无效；带上附加组件 ID 重跑，让 `allowed_extensions` 有值。
5. **二进制路径失效。** 运行 `%LOCALAPPDATA%\Hongzhao\hongzhao-companion.exe --version`，这个版本会打印 `0.2.1`。如果你移动了安装目录，清单里的 `path` 就对不上，宿主无法启动。

### MCP 返回「没有匹配的痕迹」

镜像为空或已过期。先在扩展设置里同步镜像，再重试。AI 通过 `traces.add` 提交的新痕要等下一次同步才会出现在库里。

### MCP 返回「令牌无效」

设了 `HZ_TOKEN`（或 `--token`），但客户端没带匹配的 `_token` 参数。去掉令牌，或让客户端能传这个参数。

### 必须安装伴生进程吗

不必须。它是可选项，不同步镜像时扩展根本不会联系它。

## 相关链接

- **全部下载**：<https://github.com/weifengchui/hongzhao-downloads/releases/latest>
- **Windows 安装包直链**：<https://github.com/weifengchui/hongzhao-downloads/releases/latest/download/Hongzhao-Companion-Setup.exe>
- **源码仓库**：<https://github.com/weifengchui/hongzhao>（私有仓库，需维护者授权，公开访问会 404）
- **隐私政策**：<https://github.com/weifengchui/hongzhao-downloads/blob/main/PRIVACY.md>（公开），同时涵盖扩展本体；上面的「隐私」一节是伴生进程相关的部分。
- **支持**：hongzhaosupport@163.com
- **扩展本体**：不在本仓库分发。若有官方商店页面请从商店安装，否则从源码仓库自行构建。

## 许可与免责

伴生进程采用 MIT 许可，声明在源码仓库的 `companion/Cargo.toml` 中。本仓库只存放构建产物，自身没有源码树，也没有 `LICENSE` 文件。

二进制按「现状」提供，不附带任何形式的担保。它们以你的用户权限运行，只会改动上面列出的用户级浏览器目录与注册表位置。若需要确认究竟执行了什么，请阅读源码仓库中的安装脚本。
