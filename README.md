
# Antigravity AI Proxy Fix (Linux / Remote SSH)

---

<a name="english"></a>

## 🇬🇧 English Description

This project provides a robust solution for connecting **Antigravity / Cursor AI** features on remote Linux servers (via VS Code Remote SSH) when behind a corporate proxy or firewall.

### The Problem

Modern AI tools often use Go-based binaries (Language Servers) that:

1. Ignore standard `http_proxy` / `all_proxy` environment variables.
2. Fail to resolve DNS correctly in proxy environments.
3. Automatically update, overwriting any manual modifications.
4. Suffer from HTTP/2 connection drops.

### The Solution

This repository uses **[graftcp](https://github.com/hmgle/graftcp)** to intercept TCP connections at the syscall level and force them through a SOCKS5 proxy. It includes an **automated wrapper script** that:

* Detects the Antigravity binary automatically.
* Backs up the original binary.
* Injects a wrapper to force `graftcp` usage.
* Sets critical Go environment variables (`netdns=cgo`, `http2client=0`).
* **Survives updates**: Just run the script again if the plugin updates.

---

<a name="中文说明"></a>

## 🇨🇳 中文说明

本项目提供了一套完整的解决方案，用于修复在 Linux 远程服务器（特别是autodl这种无法自己配置虚拟网卡的容器）环境下，**Antigravity / Cursor AI** 无法连接网络的问题。

### 痛点分析

现有的 AI 插件通常使用 Go 语言编写的后端服务，存在以下问题：

1. **不服管教**：经常忽略系统的 `http_proxy` 环境变量。
2. **DNS 解析失败**：在代理环境下无法正确解析域名。
3. **自动覆盖**：插件自动更新后，手动修改的配置会被原版文件覆盖。
4. **连接不稳定**：HTTP/2 协议在某些代理下会导致 EOF 错误。

### 核心功能

利用 **[graftcp](https://github.com/hmgle/graftcp)** 在系统调用层级强制接管 TCP 流量，并配合**一键修复脚本**实现：

* 🔍 **自动定位**：自动查找 Antigravity 的安装目录（无视 Hash 变化）。
* 🛡️ **智能注入**：备份原程序，植入代理 Wrapper 脚本。
* ⚙️ **环境调优**：强制开启 `netdns=cgo` 和禁用 `http2client`，解决断连问题。
* 🔄 **对抗更新**：当插件更新导致 AI 失效时，运行一次脚本即可“满血复活”。

---

## 🚀 Quick Start / 快速开始

### 1. Prerequisites / 准备工作

You need to install `graftcp` on your Linux server first.
首先需要在 Linux 服务器上编译安装 `graftcp`。

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y git make gcc golang-go

# Clone and Build
cd ~
git clone https://github.com/hmgle/graftcp.git
cd graftcp
make

```

### 2. Install the Fix / 安装修复脚本

Download the `fix_ai.sh` script or copy the content below.
下载本仓库的 `fix_ai.sh` 或直接复制以下内容。

1. Create the file / 创建文件:
```bash
nano ~/fix_ai.sh

```


2. Paste the content (Check `GRAFTCP_DIR`!) / 粘贴内容 (注意修改路径):
> **Note:** Set `PROXY_URL` to your local SOCKS5 proxy port (e.g., the port forwarded via SSH).
> **注意：** 修改 `PROXY_URL` 为你的代理端口（通常是你 SSH 远程转发过来的端口）。


```bash
#!/bin/bash

# ============ CONFIGURATION ============
# Path to your graftcp directory
GRAFTCP_DIR="/root/graftcp"
# Your Proxy URL (Matches your SSH Remote Forward port)
PROXY_URL="127.0.0.1:7890"
# Log file
LOG_FILE="/tmp/antigravity_wrapper.log"
# =======================================

# 1. Locate the binary
TARGET=$(find ~/.antigravity-server -name "language_server_linux_x64" | head -n 1)

if [ -z "$TARGET" ]; then
    echo "❌ Error: Antigravity directory not found."
    exit 1
fi

echo "📍 Target found: $TARGET"

# 2. Backup original binary
if ! grep -q "Wrapper Start" "$TARGET"; then
    mv "$TARGET" "$TARGET.bak"
    echo "📦 Original binary backed up to .bak"
fi

# 3. Inject Wrapper Script
cat > "$TARGET" <<EOF
#!/bin/bash
GRAFTCP_DIR="$GRAFTCP_DIR"
PROXY_URL="$PROXY_URL"
LOG_FILE="$LOG_FILE"

echo "=== Wrapper Start: \$(date) ===" >> "\$LOG_FILE"

# Start graftcp-local daemon if not running
if ! pgrep -f "graftcp-local" > /dev/null; then
    echo "[wrapper] Starting graftcp-local..." >> "\$LOG_FILE"
    nohup "\$GRAFTCP_DIR/local/graftcp-local" -socks5="\$PROXY_URL" >> "\$LOG_FILE" 2>&1 &
    sleep 0.5
fi

# Optimization for Go Network
export GODEBUG=netdns=cgo,http2client=0

# Execute original binary via graftcp
# IMPORTANT: Do not redirect stdout (>>), only stderr (2>>)
exec "\$GRAFTCP_DIR/graftcp" "\$0.bak" "\$@" 2>> "\$LOG_FILE"
EOF

chmod +x "$TARGET"
echo "✅ Fix applied successfully!"
echo "👉 Please press [F1] -> 'Reload Window' in VS Code."

```


3. Run it / 运行脚本:
```bash
chmod +x ~/fix_ai.sh
~/fix_ai.sh

```


4. **Restart VS Code Window**: Press `F1` (or `Ctrl+Shift+P`) -> Type `Reload Window`.

---

## 🛠 Troubleshooting / 常见问题排查

### Issue 1: "Waiting for lock..." / VS Code Stuck

If VS Code hangs on connection.
如果 VS Code 卡在连接界面。

**Fix:** Run these commands on the server:

```bash
# Kill stuck processes
pkill -f vscode-server
pkill -f vscode-ipc

# Remove lock files
rm -rf /tmp/vscode*
rm -rf ~/.vscode-server/bin/*

```

### Issue 2: Port Occupied (Address already in use)

If log says port 7890 (or your proxy port) is in use.
如果日志提示端口被占用。

**Fix:**

```bash
# Check who is using the port
ss -lptn | grep 7890
# Or kill graftcp directly
pkill -f graftcp

```

### Issue 3: AI Still Not Working

Check the log file:
查看日志文件：

```bash
tail -f /tmp/antigravity_wrapper.log

```

* Ensure `graftcp-local` is running.
* Ensure your `PROXY_URL` is correct and accessible.

---

## 📝 Technical Details / 技术原理

1. **Wrapper Injection**: We replace the original `language_server_linux_x64` with a shell script. This script acts as a middleware.
2. **Syscall Interception**: `graftcp` uses `ptrace` to redirect socket connections, bypassing the application's internal proxy ignorance.
3. **Go Tweaks**:
* `GODEBUG=netdns=cgo`: Forces the binary to use the system's DNS resolver (libc) instead of Go's native resolver, ensuring DNS requests go through the proxy.
* `GODEBUG=http2client=0`: Disables HTTP/2 to prevent connection resets common with some proxy servers.



---

## Credits

* [graftcp](https://github.com/hmgle/graftcp) by hmgle.

---
