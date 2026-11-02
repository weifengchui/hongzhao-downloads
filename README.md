**English** | [简体中文](README.zh-CN.md)

# Hongzhao Companion — Downloads

[![Latest release](https://img.shields.io/github/v/release/weifengchui/hongzhao-downloads?label=companion)](https://github.com/weifengchui/hongzhao-downloads/releases/latest)
[![Downloads](https://img.shields.io/github/downloads/weifengchui/hongzhao-downloads/total)](https://github.com/weifengchui/hongzhao-downloads/releases)

The optional local companion for the Hongzhao browser extension, published here as installers only. The source code lives in a private repository, so this repo contains binaries and nothing else.

## What this repository is

Hongzhao (鸿爪) is a local-first browser extension that keeps a searchable memory of the pages you have read, including full-text snapshots so a page stays readable after the original goes offline. The extension itself is not distributed here.

The companion is a single-file Rust process that runs entirely on your machine:

- **Native messaging host** (`com.hongzhao.companion`) — receives the metadata mirror the extension pushes on demand, which lets the extension open local folders and files with your default applications.
- **MCP server over stdio** — exposes that mirror to local AI clients such as Claude Desktop.

The extension works without it: browsing, saving traces and the memory wall all keep working. You only lose "open in local app" and local AI retrieval.

The companion never reads browser databases, never opens a network connection, and sees only the mirror the extension chose to send.

## System requirements

- **Windows** — Windows 10/11, x64. The published installer is built for `x86_64-pc-windows-msvc`.
- **macOS / Linux** — no prebuilt package is published yet; install from the companion source tree. Rust is required unless a prebuilt binary exists for your platform.
- **Browser** — Chrome 116+, Edge, Brave or Chromium. Firefox works too, but needs its add-on ID.
- **Extension ID** — Chrome: `chrome://extensions` with Developer mode on (32 letters `a`–`p`). Firefox: `about:debugging` → This Firefox → Extension ID.

## Install

### Windows

1. Open [Releases](../../releases/latest) and download `Hongzhao-Companion-Setup.exe` (608,768 bytes in `companion-v0.1.2`, about 594 KB).
2. Double-click it. It is a single file: nothing to extract, no Rust required.
3. The installer detects installed Chromium browsers, looks up the Hongzhao extension ID in their profiles, unpacks the host to `%LOCALAPPDATA%\Hongzhao`, writes the native-messaging manifest there, and registers `HKCU\Software\<vendor>\<browser>\NativeMessagingHosts\com.hongzhao.companion`. If no ID is found, it asks you to paste one.
4. Quit the browser completely — every window, including the tray icon — and start it again.
5. In the extension: **Settings → Companion process → Check connection**, then **Sync mirror**.

The installer is not code-signed, so Windows SmartScreen may warn about an unknown publisher. Check the SHA-256 below before running it.

Firefox is not registered by this installer. Use the script described under [Firefox](#firefox-any-platform).

### macOS / Linux

There is no published macOS build yet: `companion-v0.1.2` ships a Windows installer and `SHA256SUMS.txt` only. `Hongzhao-Companion-Setup-macos.zip` is reserved in the release configuration but not produced.

From a companion source tree:

```bash
cd companion
./install.sh <chrome-extension-id>
```

On macOS you can also double-click `companion/install-macos.command`, which calls the same script. The script reuses a prebuilt binary from `companion/bin/` when one exists for your OS and CPU; otherwise it builds with `cargo` (install Rust from <https://rustup.rs>).

It writes `com.hongzhao.companion.json` into the native-messaging directory of every Chromium-family profile it can find — Chrome, Chromium, Edge and Brave under `~/.config/...` on Linux or `~/Library/Application Support/...` on macOS — and into `~/.mozilla/native-messaging-hosts/` when a Firefox ID is supplied. At least one Chromium browser must have been launched once, or the script exits without installing.

### Firefox (any platform)

Firefox uses the add-on ID instead of the Chrome extension ID, so pass it as the second argument:

```bash
./install.sh <chrome-extension-id> <firefox-addon-id>
```

On Windows:

```powershell
cd companion
.\install-windows.ps1 <chrome-extension-id> <firefox-addon-id>
```

That writes `%APPDATA%\Mozilla\native-messaging-hosts\com.hongzhao.companion.json`.

## Verify the download

Each release ships `SHA256SUMS.txt` next to the binaries. For `companion-v0.1.2`:

```text
bb8e909fef19fb889a72f51ba4573c16439f129fefb2a7039dc28dfcdd7d973f  Hongzhao-Companion-Setup.exe
```

Compare it with the file you downloaded:

```powershell
# Windows PowerShell
Get-FileHash .\Hongzhao-Companion-Setup.exe -Algorithm SHA256
```

```bat
:: Windows, no PowerShell
certutil -hashfile Hongzhao-Companion-Setup.exe SHA256
```

```bash
# macOS
shasum -a 256 Hongzhao-Companion-Setup.exe

# Linux, with both files in the current directory
sha256sum -c SHA256SUMS.txt
```

## Connect your AI client (MCP)

The companion is a stdio MCP server. `hongzhao-companion mcp` starts it explicitly, and `mcp` is also the default mode when no argument is given. Add it to your client's existing `mcpServers` block — for Claude Desktop:

```json
{
  "mcpServers": {
    "hongzhao": {
      "command": "C:\\Users\\<you>\\AppData\\Local\\Hongzhao\\hongzhao-companion.exe",
      "args": ["mcp"]
    }
  }
}
```

On macOS/Linux, use the binary path printed by the installer, for example `/path/to/companion/target/release/hongzhao-companion`.

**Optional token.** Start the server with `--token <value>` or set `HZ_TOKEN`. Once a token is configured, every tool call must include a matching `_token` argument, and calls without it are rejected — so leave it unset unless your client can pass that argument.

Sync before asking: **Settings → Companion process → Sync mirror**. The mirror holds up to 20,000 traces; traces marked as not allowed for AI, and traces in the trash, are never mirrored.

| Tool | What it does |
| --- | --- |
| `traces.search` | Searches title, tags, site, excerpt and note. Takes `query` and optional `limit` (1–50, default 8). |
| `traces.get` | Reads one trace's metadata by `id`. |
| `traces.add` | Queues a trace for import; the extension stores it on its next sync with the `agent` source. |
| `traces.cite` | Builds citation text (title, URL, visit date) for a list of `ids`. |

The server speaks MCP protocol version `2024-11-05` over newline-delimited JSON and identifies itself as `hongzhao-companion`.

## Privacy

The companion is deliberately narrow in scope:

- **Metadata only.** The mirror contains URL, title, site, tags, note, the extension's excerpt and timestamps. No full-text snapshots, no semantic vectors.
- **Local only.** The binary opens no sockets and never reads browser storage. The extension pushes the mirror; nothing flows back.
- **On demand.** `~/.hongzhao/mirror.json` is written only when you press "Sync mirror". Set `HZ_HOME` to relocate the directory.
- **Audited.** Every tool call appends one line to `~/.hongzhao/audit.jsonl`: timestamp, tool name, success flag, hit count and a short status note. Page content is never recorded. The log rotates to `audit.jsonl.1` at 2 MB.
- **Removable.** Delete `%LOCALAPPDATA%\Hongzhao`, the registry keys it created, the native-messaging manifest and `~/.hongzhao`. There is no remote state to clean up.

## FAQ

### The settings page says "Local helper not found"

Work through these in order:

1. **Manifest not registered.** On Windows, `%LOCALAPPDATA%\Hongzhao\com.hongzhao.companion.json` must exist, and `HKCU\Software\Google\Chrome\NativeMessagingHosts\com.hongzhao.companion` (or the Edge/Brave/Chromium equivalent) must point to it. On macOS/Linux, `com.hongzhao.companion.json` must sit in the browser's `NativeMessagingHosts` directory.
2. **Extension ID mismatch.** The manifest's `allowed_origins` must contain exactly `chrome-extension://<your-id>/`. Re-run the installer with the correct ID.
3. **Browser not fully restarted.** Native-messaging hosts are resolved at browser startup. Close every window and the tray icon, then relaunch.
4. **Firefox without an add-on ID.** A `chrome-extension://` origin means nothing to Firefox; re-run with the add-on ID so `allowed_extensions` is filled in.
5. **Broken binary path.** Run `%LOCALAPPDATA%\Hongzhao\hongzhao-companion.exe --version`; this release prints `0.1.1`. If you move the installation, the manifest's `path` no longer matches and the host cannot start.

### MCP answers "no matching traces"

The mirror is empty or stale. Sync it from the extension settings first, then retry. Annotation-only changes made by the AI arrive through `traces.add` and are visible after the next sync.

### MCP answers "token invalid"

`HZ_TOKEN` (or `--token`) is set but the client did not send a matching `_token` argument. Remove the token, or configure the client to pass it.

### Is the companion required?

No. It is optional, and the extension never contacts it unless you sync.

## Related links

- **All downloads** — <https://github.com/weifengchui/hongzhao-downloads/releases/latest>
- **Windows installer, direct link** — <https://github.com/weifengchui/hongzhao-downloads/releases/latest/download/Hongzhao-Companion-Setup.exe>
- **Source code** — <https://github.com/weifengchui/hongzhao> (private; access must be granted by the maintainer, and the URL is not publicly browsable)
- **Privacy policy** — `PRIVACY.md` in that same private source repository, which also covers the extension. The summary above is the companion-specific part.
- **Extension** — not distributed in this repository. Install it from its official browser add-on listing if one exists, or build it from the source repository.

## License & disclaimer

The companion is MIT-licensed; that is declared in `companion/Cargo.toml` in the source repository. This repository holds build artifacts only, with no source tree and no `LICENSE` file of its own.

The binaries are provided "as is", without warranty of any kind. They run with your user privileges and modify only the per-user browser and registry locations described above. Read the install scripts in the source repository if you need to know exactly what runs.
