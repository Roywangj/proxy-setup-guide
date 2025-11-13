# 统一代理配置方案

完整的代理配置指南，涵盖跨设备代理共享和统一配置管理两种场景。

**场景一**：让设备B（Ubuntu服务器）通过设备A（macOS）的代理访问外网  
**场景二**：统一管理终端、VSCode、Cursor 的代理配置，一处修改全局生效

## 📋 目录

- [概述](#概述)
- [场景一：跨设备代理配置](#场景一跨设备代理配置)
  - [设备A（macOS）配置](#设备amacOS配置)
  - [设备B（Ubuntu服务器）配置](#设备bubuntu服务器配置)
  - [验证和测试](#验证和测试)
- [场景二：统一代理配置管理](#场景二统一代理配置管理)
  - [配置文件](#配置文件)
  - [使用方法](#使用方法)
- [工作原理](#工作原理)
- [故障排除](#故障排除)

---

## 概述

### 🎯 目标

实现统一的代理配置管理：
- ✅ 所有工具共享同一个代理配置源
- ✅ 只需修改一处即可全局生效
- ✅ 支持终端、VSCode、Cursor 等工具
- ✅ 支持跨设备代理共享

### 📦 使用场景

本指南涵盖两种主要使用场景：

1. **场景一：跨设备代理配置**
   - 设备A（macOS）可以翻墙
   - 设备B（Ubuntu服务器）通过SSH访问设备A
   - 让设备B通过设备A的代理访问外网

2. **场景二：统一代理配置管理**
   - 在单台服务器上配置代理
   - 让终端、VSCode、Cursor 统一使用相同代理配置

---

## 场景一：跨设备代理配置

### 背景说明

当您有以下需求时，可以使用此方案：
- 设备A（macOS）已配置翻墙工具（如 ClashX）
- 设备B（Ubuntu服务器）通过SSH访问设备A
- 希望设备B也能通过设备A的代理访问外网

### 架构图

```
┌─────────────────┐           ┌──────────────────┐
│   设备A (macOS)  │           │ 设备B (Ubuntu)    │
│                 │           │                  │
│   ClashX/代理   │◄──────────│  SSH连接         │
│   端口: 7890    │   代理请求  │  应用程序        │
│   IP: 内网地址  │           │  VSCode/Cursor   │
└─────────────────┘           └──────────────────┘
```

---

### 设备A（macOS）配置

#### 步骤1：配置代理软件允许局域网访问

##### 方法1：通过ClashX菜单（最简单）

1. 点击菜单栏的 **ClashX 图标**
2. 查找并勾选以下选项之一：
   - **「允许局域网连接」** 或
   - **「Allow Connect from Lan」**

##### 方法2：手动编辑配置文件（推荐）

如果找不到菜单选项，手动编辑配置文件：

```bash
# 在macOS终端执行

# 1. 打开配置文件目录
open ~/.config/clash

# 2. 编辑 config.yaml 文件
# 找到并修改以下配置（通常在文件开头）：
```

在打开的文件夹中，找到 `config.yaml`，用文本编辑器打开，修改：

```yaml
# 找到这几行（通常在文件开头）
port: 7890              # HTTP代理端口
socks-port: 7891        # SOCKS5代理端口
allow-lan: true         # 改为 true（重要！）
bind-address: '*'       # 改为 '*' 或删除这行（重要！）
```

**关键配置说明：**
- `allow-lan: true` - 允许局域网设备连接（必须）
- `bind-address: '*'` - 监听所有网络接口（必须）

##### 方法3：使用命令行快速修改

```bash
# 备份原配置
cp ~/.config/clash/config.yaml ~/.config/clash/config.yaml.backup

# 查看当前配置
grep -E "allow-lan|bind-address" ~/.config/clash/config.yaml

# 修改 allow-lan 为 true
sed -i '' 's/allow-lan: false/allow-lan: true/' ~/.config/clash/config.yaml
```

**修改后重启 ClashX：**
1. 点击菜单栏 ClashX 图标
2. 选择「退出 ClashX」
3. 重新打开 ClashX

#### 步骤2：获取设备A的内网IP地址

```bash
# 在macOS终端执行
ifconfig | grep "inet " | grep -v 127.0.0.1
```

记下输出的IP地址，例如：`192.168.1.100`

#### 步骤3：验证端口监听状态

```bash
# 检查端口是否监听在所有地址上
netstat -an | grep -E "7890|7891"

# 应该看到类似这样的输出（注意是 *.7890，不是 127.0.0.1.7890）：
# tcp4       0      0  *.7890                 *.*                    LISTEN
# tcp4       0      0  *.7891                 *.*                    LISTEN
```

**重要说明：**
- 如果看到 `127.0.0.1.7890`，说明只监听本地，需要重新配置
- 如果看到 `*.7890`，说明配置正确，已监听所有网络接口

#### 步骤4：配置防火墙（可选）

确保macOS防火墙允许代理端口的连接：

**方式1：临时关闭防火墙测试**
- 系统偏好设置 → 安全性与隐私 → 防火墙 → 关闭

**方式2：允许特定应用**
- 系统偏好设置 → 安全性与隐私 → 防火墙 → 防火墙选项
- 确保 ClashX 被允许接收传入连接

**注意：** SSH能通（22端口）不代表代理端口（7890/7891）也能通，需要单独确认。

---

### 设备B（Ubuntu服务器）配置

#### 方法1：临时配置（当前会话有效）

通过SSH连接到设备B后，执行：

```bash
# 第一步：设置代理服务器地址（只需修改这一行）
PROXY_HOST="192.168.1.100"  # 替换为您的macOS IP

# 第二步：应用代理设置（以下四行无需修改）
export http_proxy=http://${PROXY_HOST}:7890
export https_proxy=http://${PROXY_HOST}:7890
export all_proxy=socks5://${PROXY_HOST}:7891
export no_proxy=localhost,127.0.0.1,localaddress,.localdomain.com
```

#### 方法2：永久配置（推荐）

让每次登录都自动启用代理：

```bash
# 在设备B上执行

# 设置您的macOS IP地址
PROXY_HOST="192.168.1.100"  # 替换为您的实际IP

# 添加到 ~/.bashrc（Bash用户）
cat >> ~/.bashrc <<EOF

# ==================== 代理配置 ====================
# 通过设备A (macOS) 访问外网
PROXY_HOST="${PROXY_HOST}"

export http_proxy=http://\${PROXY_HOST}:7890
export https_proxy=http://\${PROXY_HOST}:7890
export all_proxy=socks5://\${PROXY_HOST}:7891
export no_proxy=localhost,127.0.0.1,localaddress,.localdomain.com

# VSCode/Cursor 特定环境变量
export VSCODE_HTTP_PROXY=http://\${PROXY_HOST}:7890
# ==================================================
EOF

# 使配置立即生效
source ~/.bashrc
```

**对于 Zsh 用户：**

```bash
# 添加到 ~/.zshrc
cat >> ~/.zshrc <<EOF

# ==================== 代理配置 ====================
# 通过设备A (macOS) 访问外网
PROXY_HOST="${PROXY_HOST}"

export http_proxy=http://\${PROXY_HOST}:7890
export https_proxy=http://\${PROXY_HOST}:7890
export all_proxy=socks5://\${PROXY_HOST}:7891
export no_proxy=localhost,127.0.0.1,localaddress,.localdomain.com

# VSCode/Cursor 特定环境变量
export VSCODE_HTTP_PROXY=http://\${PROXY_HOST}:7890
# ==================================================
EOF

source ~/.zshrc
```

#### 方法3：快捷命令管理（高级）

创建便捷的代理开关命令：

```bash
# 添加到 ~/.bashrc 或 ~/.zshrc
cat >> ~/.bashrc <<'EOF'

# 代理快捷命令
alias proxy-on='export http_proxy=http://192.168.1.100:7890 https_proxy=http://192.168.1.100:7890 all_proxy=socks5://192.168.1.100:7891'
alias proxy-off='unset http_proxy https_proxy all_proxy'
alias proxy-status='env | grep -i proxy'
alias proxy-test='curl -I https://www.google.com'
EOF

source ~/.bashrc
```

**使用方式：**
- `proxy-on` - 开启代理
- `proxy-off` - 关闭代理
- `proxy-status` - 查看代理状态
- `proxy-test` - 测试代理连接

---

### 验证和测试

#### 1. 测试端口连通性

在设备B上测试设备A的代理端口是否可访问：

```bash
# 测试HTTP代理端口（7890）
nc -zv 192.168.1.100 7890

# 或使用 telnet
telnet 192.168.1.100 7890

# 测试SOCKS5代理端口（7891）
nc -zv 192.168.1.100 7891
```

**预期结果：**
```
Connection to 192.168.1.100 7890 port [tcp/*] succeeded!
```

**如果失败：** 返回[设备A配置](#设备amacOS配置)，检查防火墙和端口监听。

#### 2. 测试HTTP访问

```bash
# 测试访问Google
curl -I https://www.google.com

# 应该看到：
# HTTP/1.1 301 Moved Permanently
# Server: gws
# Proxy-Connection: keep-alive
```

#### 3. 测试其他服务

```bash
# 测试GitHub
curl -I https://github.com

# 测试YouTube
curl -I https://www.youtube.com

# 查看当前IP（应该显示代理IP）
curl https://ipinfo.io
curl https://myip.ipip.net
```

#### 4. 测试VSCode/Cursor插件下载

1. **重新连接远程服务器**
   - 在VSCode/Cursor中断开SSH连接
   - 重新连接到设备B

2. **尝试安装插件**
   - 搜索并安装任意插件
   - 观察是否能正常下载

3. **检查环境变量**
   - 打开集成终端（`` Ctrl+` ``）
   - 执行：`echo $http_proxy`
   - 应该显示：`http://192.168.1.100:7890`

---

### 常见问题排查

#### 问题1：连接被拒绝 (Connection Refused)

**现象：**
```bash
curl: (7) Failed to connect to 192.168.1.100 port 7890: Connection refused
```

**可能原因及解决方案：**

1. **设备A的代理软件未开启局域网访问**
   ```bash
   # 在设备A上检查配置
   cat ~/.config/clash/config.yaml | grep allow-lan
   # 确保显示: allow-lan: true
   ```

2. **设备A的防火墙阻止了连接**
   - 临时关闭防火墙测试
   - 或在防火墙中允许 ClashX

3. **端口未监听在所有接口上**
   ```bash
   # 在设备A上检查
   netstat -an | grep 7890
   # 应该看到 *.7890，而不是 127.0.0.1.7890
   ```

#### 问题2：设备A进入睡眠后代理断开

**解决方案：**

在设备A（macOS）上禁用睡眠：
- 系统偏好设置 → 节能 → 勾选「防止电脑自动进入睡眠」

#### 问题3：网络切换后IP地址变化

**现象：** 设备A更换网络后，IP地址改变，设备B无法连接

**解决方案：**

1. **在设备A上查看新IP**
   ```bash
   ifconfig | grep "inet " | grep -v 127.0.0.1
   ```

2. **在设备B上更新配置**
   ```bash
   # 编辑 ~/.bashrc 或 ~/.zshrc
   vim ~/.bashrc
   # 修改 PROXY_HOST 的值
   
   # 重新加载
   source ~/.bashrc
   ```

#### 问题4：VSCode/Cursor插件仍然无法下载

**解决方案：**

1. **完全重启VSCode Server**
   ```bash
   # 在设备B上执行
   killall node
   rm -rf ~/.vscode-server/bin/*/.nfs*
   ```

2. **手动配置VSCode Server**
   ```bash
   # 创建配置目录
   mkdir -p ~/.vscode-server/data/Machine
   
   # 创建配置文件
   cat > ~/.vscode-server/data/Machine/settings.json <<EOF
   {
       "http.proxy": "http://192.168.1.100:7890",
       "http.proxySupport": "on",
       "http.proxyStrictSSL": false
   }
   EOF
   ```

3. **重新连接VSCode**
   - 断开SSH连接
   - 重新连接到设备B
   - 在VSCode中执行：`Cmd+Shift+P` → `Reload Window`

---

### 注意事项

⚠️ **使用限制：**
- 设备A必须保持开机且代理软件运行中
- 设备A进入睡眠时，代理会断开
- 网络切换后，设备A的IP可能变化

💡 **安全建议：**
- 仅在可信任的网络环境中使用
- 考虑为代理设置用户名密码认证
- 不要将代理端口暴露到公网

🎯 **性能优化：**
- 使用 `no_proxy` 排除内网地址，避免不必要的代理
- 考虑在设备B上安装Clash并配置上游代理（参考高级方案）

---

## 场景二：统一代理配置管理

当您已经在设备B上配置好代理（无论是通过设备A还是其他方式），以下方案可以帮助您统一管理终端、VSCode、Cursor的代理配置。

### 配置文件

#### 1. Shell 配置文件 (`~/.zshrc`)

在 `~/.zshrc` 文件末尾添加以下内容：

```bash
# ==================== 代理配置 ====================
# 第一步：设置代理主机（修改这里即可更换代理地址）
PROXY_HOST="10.106.1.36"

# 第二步：应用代理设置（以下四行无需修改）
export http_proxy=http://${PROXY_HOST}:7897
export https_proxy=http://${PROXY_HOST}:7897
export all_proxy=socks5://${PROXY_HOST}:7897
# export no_proxy=localhost,127.0.0.1,localaddress,.localdomain.com,10.0.0.0/8

# 同步代理配置到 VSCode/Cursor
alias sync-proxy='~/sync-proxy-config.sh'
```

**说明：**
- `PROXY_HOST`：代理服务器的 IP 地址（**只需修改这一行**）
- `http_proxy` 等：终端环境变量，影响 curl、wget、Python 等工具
- `no_proxy`：不走代理的地址列表（目前已注释，如需启用请去掉 `#`）
- `sync-proxy`：快捷命令，用于同步配置到编辑器

#### 2. 同步脚本 (`~/sync-proxy-config.sh`)

创建文件 `~/sync-proxy-config.sh`，内容如下：

```bash
#!/bin/bash
# 自动同步代理配置到 VSCode/Cursor

# 从 .zshrc 中读取 PROXY_HOST
PROXY_HOST=$(grep -E '^PROXY_HOST=' ~/.zshrc | cut -d'"' -f2)

if [ -z "$PROXY_HOST" ]; then
    echo "❌ 无法从 .zshrc 中读取 PROXY_HOST"
    exit 1
fi

echo "📡 检测到代理主机: $PROXY_HOST"
echo ""

# 配置文件路径
CURSOR_SETTINGS="$HOME/.cursor-server/data/Machine/settings.json"
VSCODE_SETTINGS="$HOME/.vscode-server/data/Machine/settings.json"

# 生成配置内容
generate_config() {
    cat <<EOF
{
    "http.proxy": "http://${PROXY_HOST}:7897",
    "http.proxySupport": "on",
    "http.proxyStrictSSL": false,
    "http.noProxy": [
        "localhost",
        "127.0.0.1",
        "10.0.0.0/8"
    ]
}
EOF
}

# 更新 Cursor 配置
if [ -d "$HOME/.cursor-server" ]; then
    mkdir -p "$(dirname "$CURSOR_SETTINGS")"
    generate_config > "$CURSOR_SETTINGS"
    echo "✅ Cursor 代理配置已更新: http://${PROXY_HOST}:7897"
else
    echo "⚠️  Cursor Server 未安装，跳过"
fi

# 更新 VSCode 配置
if [ -d "$HOME/.vscode-server" ]; then
    mkdir -p "$(dirname "$VSCODE_SETTINGS")"
    generate_config > "$VSCODE_SETTINGS"
    echo "✅ VSCode 代理配置已更新: http://${PROXY_HOST}:7897"
else
    echo "⚠️  VSCode Server 未安装，跳过"
fi

echo ""
echo "💡 提示: 请在编辑器中执行 'Developer: Reload Window' 使配置生效"
```

**设置执行权限：**

```bash
chmod +x ~/sync-proxy-config.sh
```

---

### 使用方法

#### 🚀 初次设置

1. **编辑 `~/.zshrc`**，添加上述配置内容
2. **创建同步脚本** `~/sync-proxy-config.sh`，设置执行权限
3. **重新加载配置**：
   ```bash
   source ~/.zshrc
   ```
4. **同步到编辑器**：
   ```bash
   sync-proxy
   ```

#### 🔄 日常使用

##### 更换代理地址

当需要更换代理服务器时：

**步骤 1：修改配置**

编辑 `~/.zshrc`，修改 `PROXY_HOST` 行：

```bash
PROXY_HOST="新的IP地址"
```

**步骤 2：重新加载**

```bash
source ~/.zshrc
```

**步骤 3：同步配置**

```bash
sync-proxy
```

输出示例：
```
📡 检测到代理主机: 10.106.1.36

✅ Cursor 代理配置已更新: http://10.106.1.36:7897
✅ VSCode 代理配置已更新: http://10.106.1.36:7897

💡 提示: 请在编辑器中执行 'Developer: Reload Window' 使配置生效
```

**步骤 4：重新加载编辑器窗口**

在 VSCode/Cursor 中：
1. 按 `Cmd+Shift+P` (macOS) 或 `Ctrl+Shift+P` (Linux/Windows)
2. 输入 `Reload Window`
3. 选择 `Developer: Reload Window`

##### 临时禁用代理

如果需要临时禁用代理（当前终端会话）：

```bash
unset http_proxy https_proxy all_proxy
```

重新开启（重新加载配置）：

```bash
source ~/.zshrc
```

##### 永久禁用代理

注释掉 `~/.zshrc` 中的相关行：

```bash
# PROXY_HOST="10.106.1.36"

# export http_proxy=http://${PROXY_HOST}:7897
# export https_proxy=http://${PROXY_HOST}:7897
# export all_proxy=socks5://${PROXY_HOST}:7897
```

---

## 工作原理

### 架构图

```
┌─────────────────────────────────────────────┐
│              ~/.zshrc                        │
│   PROXY_HOST="10.106.1.36"  ← 唯一配置源    │
└─────────────────┬───────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
┌───────────────┐   ┌──────────────────┐
│  终端环境变量  │   │  sync-proxy-config.sh│
│               │   │   (同步脚本)       │
│ http_proxy=...│   └──────────┬────────┘
│ https_proxy=..│              │
│ all_proxy=... │         ┌────┴────┐
└───────────────┘         │         │
                          ▼         ▼
                  ┌──────────┐  ┌──────────┐
                  │ Cursor   │  │ VSCode   │
                  │ settings │  │ settings │
                  └──────────┘  └──────────┘
```

### 配置层级

1. **配置源**：`~/.zshrc` 中的 `PROXY_HOST` 变量
2. **终端代理**：通过环境变量自动生效
3. **编辑器代理**：通过 `sync-proxy` 命令手动同步

### 生效范围

| 工具/场景 | 配置方式 | 生效时机 |
|----------|---------|---------|
| 终端命令 (curl, wget, git) | 环境变量 | 自动（开启新终端或 source） |
| Python requests | 环境变量 | 自动 |
| VSCode 插件 | settings.json | 手动（运行 sync-proxy 后） |
| Cursor 插件 | settings.json | 手动（运行 sync-proxy 后） |
| npm/yarn | 环境变量 | 自动 |

---

## 故障排除

本节包含两个场景的常见问题解决方案。

### 场景一相关问题

详见[场景一：常见问题排查](#常见问题排查)部分。

### 场景二相关问题

#### 问题 1：连接被拒绝 (Connection Refused)

**现象：**
```
ConnectionRefusedError: [Errno 111] Connection refused
```

**可能原因：**
- 代理服务器未运行
- 代理地址或端口错误
- 内网地址不应走代理

**解决方案：**

1. 检查代理服务器是否运行
2. 验证 `PROXY_HOST` 和端口是否正确
3. 启用 `no_proxy` 排除内网地址：
   ```bash
   # 在 ~/.zshrc 中取消这一行的注释
   export no_proxy=localhost,127.0.0.1,localaddress,.localdomain.com,10.0.0.0/8
   ```

#### 问题 2：sync-proxy 命令不存在

**现象：**
```
command not found: sync-proxy
```

**解决方案：**

1. 确认脚本已创建：
   ```bash
   ls -la ~/sync-proxy-config.sh
   ```

2. 重新加载配置：
   ```bash
   source ~/.zshrc
   ```

3. 直接运行脚本：
   ```bash
   bash ~/sync-proxy-config.sh
   ```

#### 问题 3：VSCode/Cursor 插件下载失败

**可能原因：**
- 编辑器配置未同步
- 编辑器未重新加载

**解决方案：**

1. 运行同步命令：
   ```bash
   sync-proxy
   ```

2. 重新加载编辑器窗口：
   - `Cmd+Shift+P` → `Reload Window`

3. 检查配置文件：
   ```bash
   cat ~/.cursor-server/data/Machine/settings.json
   cat ~/.vscode-server/data/Machine/settings.json
   ```

#### 问题 4：部分内网地址无法访问

**现象：**
校园网认证、内网服务等无法访问

**解决方案：**

启用 `no_proxy` 配置，排除内网地址段：

```bash
export no_proxy=localhost,127.0.0.1,localaddress,.localdomain.com,10.0.0.0/8
```

并在编辑器配置中同样添加（脚本已自动处理）。

---

## 配置文件位置

### 终端配置

- **Zsh**: `~/.zshrc`
- **Bash**: `~/.bashrc`

### 编辑器配置

- **Cursor**: `~/.cursor-server/data/Machine/settings.json`
- **VSCode**: `~/.vscode-server/data/Machine/settings.json`

### 同步脚本

- **位置**: `~/sync-proxy-config.sh`
- **别名**: `sync-proxy`

---

## 最佳实践

### ✅ 推荐做法

1. **统一管理**：只修改 `~/.zshrc` 中的 `PROXY_HOST`
2. **及时同步**：修改后立即运行 `sync-proxy`
3. **版本控制**：将配置文件纳入版本控制（注意敏感信息）
4. **文档记录**：记录常用的代理地址

### ❌ 避免的做法

1. **不要直接修改编辑器配置文件**（会被 sync-proxy 覆盖）
2. **不要在多处维护代理配置**（容易不同步）
3. **不要忘记重新加载**（配置不会自动生效）

---

## 附录

### A. 完整设置示例

```bash
# 1. 编辑 ~/.zshrc
vim ~/.zshrc
# 添加本文档中的配置内容

# 2. 创建同步脚本
cat > ~/sync-proxy-config.sh << 'EOF'
# [脚本内容见上文]
EOF
chmod +x ~/sync-proxy-config.sh

# 3. 重新加载配置
source ~/.zshrc

# 4. 同步到编辑器
sync-proxy

# 5. 验证配置
echo "终端代理: $http_proxy"
cat ~/.cursor-server/data/Machine/settings.json
```

### B. 常用代理端口

| 协议 | 默认端口 | 说明 |
|-----|---------|-----|
| HTTP | 7890 | HTTP 代理 |
| HTTPS | 7890 | HTTPS 代理 |
| SOCKS5 | 7891 | SOCKS5 代理 |
| HTTP (本配置) | 7897 | 自定义端口 |

### C. 环境变量说明

| 变量名 | 作用 | 示例 |
|-------|------|------|
| `http_proxy` | HTTP 请求代理 | `http://10.106.1.36:7897` |
| `https_proxy` | HTTPS 请求代理 | `http://10.106.1.36:7897` |
| `all_proxy` | SOCKS5 代理 | `socks5://10.106.1.36:7897` |
| `no_proxy` | 不走代理的地址 | `localhost,127.0.0.1,10.0.0.0/8` |

---

## 更新日志

- **2025-11-13 v2.0**: 重大更新
  - 新增场景一：跨设备代理配置（设备B通过设备A访问外网）
  - 新增macOS ClashX配置详细步骤
  - 新增Ubuntu服务器代理配置方法
  - 新增VSCode/Cursor远程开发代理配置
  - 新增详细的验证测试和故障排查方法
  - 重组文档结构，区分两种使用场景

- **2025-11-13 v1.0**: 初始版本
  - 创建统一代理配置方案
  - 支持终端、VSCode、Cursor
  - 实现一键同步功能

---

## 许可证

此配置方案可自由使用和修改。

---

**最后更新**: 2025-11-13
**适用系统**: 
- macOS (设备A) + Ubuntu 22.04 LTS (设备B)
- Ubuntu 22.04 LTS 及其他 Linux 发行版
**Shell**: Zsh / Bash

