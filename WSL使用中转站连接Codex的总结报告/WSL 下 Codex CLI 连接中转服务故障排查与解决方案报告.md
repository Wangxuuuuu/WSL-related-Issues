# WSL 下 Codex CLI 连接中转服务故障排查与解决方案报告

**日期**：2026-07-18

**环境**：Windows 11 + WSL2 (Ubuntu) + Clash Verge Rev + itssx 8号车 (Nexus)

**涉及组件**：Codex CLI v0.142.5, Node.js v24.x, nvm

------

## 1. 问题概述

在 WSL 环境中配置 Codex CLI 以连接第三方中转服务 `itssx (Nexus)` 时，遭遇网络连接故障。具体表现为：Codex 启动后发送消息，UI 长时间处于“思考 (Thinking)”状态，无内容输出，Token 消耗为 0。经排查，根因并非 Codex 配置错误，而是 **Windows Hyper-V 防火墙拦截了 WSL 发往 Windows 宿主机的代理流量**，导致请求无法送达中转站。

## 2. 故障现象

1. **Codex 行为异常**：
   - 执行 `codex -m gpt-5.5`。
   - 界面显示已连接至 `crs - https://nexusacc.itssx.com/api/codex/codex`。
   - 输入任意指令（如 `1`），UI 卡在思考状态超过 10 分钟。
   - `/status` 显示 Token usage 始终为 0。
2. **网络诊断**：
   - 在 WSL 内执行 `curl https://nexusacc.itssx.com/api/common/models` 长时间无输出或超时。
   - 在 WSL 内执行 `curl http://172.22.80.1:7897` (Clash 代理端口) 提示 `Connection timed out`。
3. **环境特征**：
   - Windows 防火墙已开启。
   - Clash 已开启“允许局域网连接”，且监听 `0.0.0.0:7897`。
   - 高级安全防火墙中已手动放行 Clash 应用及 7897 端口，但问题依旧。

## 3. 根因分析

### 3.1 网络拓扑与流量走向

在 WSL2 架构中，WSL 运行在一个轻量级虚拟机中，拥有独立的 IP 地址（如 `172.22.80.x`），通过虚拟交换机与 Windows 宿主机通信。

- **预期走向**：`Codex -> WSL Env(http_proxy) -> 172.22.80.1:7897 (Clash) -> Internet`
- **实际阻断**：Windows 的 **Hyper-V 防火墙**（独立于常规 Windows Defender 防火墙）将来自 `172.22.80.x` (WSL 子网) 的入站连接视为“外部流量”。即使常规防火墙已放行，Hyper-V 防火墙层依然拦截了发往 `7897` 端口的连接请求，导致 TCP 握手失败。

### 3.2 关键证据

执行以下测试证实了防火墙拦截：

1. **临时关闭 Windows 防火墙**：WSL 内 `curl http://172.22.80.1:7897` 立即返回 `Connected`。
2. **PowerShell 测试**：`Test-NetConnection 172.22.80.1 -Port 7897` 显示 `TcpTestSucceeded : True`，这是因为该测试是从宿主机连宿主机，未经过 WSL 虚拟交换机过滤层。
3. **接口别名**：`Test-NetConnection` 结果显示接口为 `vEthernet (WSL (Hyper-V firewall))`，明确指出了拦截发生在 Hyper-V 防火墙层。

## 4. 解决方案

鉴于 Hyper-V 防火墙规则复杂且难以通过 GUI 完全管控，采用**“分流策略”**解决此问题：让 Codex 流量直连中转站（国内 ACC 线路稳定），仅让 Git/Apt 等流量走代理。这样既规避了防火墙拦截，又保证了其他开发工具的访问速度。

### 4.1 修改 WSL 环境变量 (.bashrc)

核心思路：将 itssx 的域名加入 `no_proxy` 列表，使其绕过 `http_proxy` 设置。

**修改前**：

```
export http_proxy="http://$WIN_HOST:7897"
export https_proxy="http://$WIN_HOST:7897"
# ...
export no_proxy="localhost,127.0.0.1,::1"
export NO_PROXY="$no_proxy"
```

**修改后**：

```
# 在 ~/.bashrc 末尾确保有以下配置

# 动态获取 Windows 主机 IP (推荐)
export WIN_HOST=$(ip route | grep default | awk '{print $3}')

# 代理设置 (供 git, apt, curl 等使用)
export http_proxy="http://$WIN_HOST:7897"
export https_proxy="http://$WIN_HOST:7897"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export all_proxy="socks5://$WIN_HOST:7890" # 如有需要
export ALL_PROXY="$all_proxy"

# 关键：将 itssx 相关域名加入直连列表
# 这样 Codex 访问这些域名时将不使用代理，避开防火墙拦截
export no_proxy="localhost,127.0.0.1,::1,$WIN_HOST,nexusacc.itssx.com,nexus.itssx.com,ai.itssx.com"
export NO_PROXY="$no_proxy"

# Codex / Node.js 相关
export NODE_USE_ENV_PROXY=1
export GLOBAL_AGENT_HTTP_PROXY="http://$WIN_HOST:7897"

# key密钥设置处
export CRS_OAI_KEY="sk-ab45xxxxxx"
```

### 4.2 更新 Codex 配置 (config.toml)

确认配置文件指向正确的中转分组地址，并使用环境变量鉴权。

**路径**：`~/.codex/config.toml`

```
model_provider = "crs"
model = "gpt-5.5"
sandbox_mode = "workspace-write"
approval_policy = "on-request"
model_reasoning_effort = "medium"
service_tier = "priority"
disable_response_storage = true
preferred_auth_method = "apikey"

[model_providers.crs]
name = "crs"
# 使用 codex#1 分组的专用加速地址
base_url = "https://nexusacc.itssx.com/api/codex/codex"
wire_api = "responses"
requires_openai_auth = false
env_key = "CRS_OAI_KEY" # 对应 .bashrc 中的环境变量名
supports_websockets = false

[features]
apps = false # 关闭内置 App，避免非 apikey 模式的警告
fast_mode = false

[projects."/home/ds-wangxu"]
trust_level = "trusted"

[tui.model_availability_nux]
"gpt-5.5" = 3
```

### 4.3 清理认证文件 (auth.json)

由于使用 `apikey` 模式，需确保 `auth.json` 不包含旧的 ChatGPT 登录态。

**路径**：`~/.codex/auth.json`

```
{
  "OPENAI_API_KEY": null
}
```

### 4.4 重启与验证

1. 关闭所有 WSL 窗口，重新打开。
2. 验证环境变量：`echo $no_proxy` (应包含 itssx 域名)。
3. 验证直连：`curl -s https://nexusacc.itssx.com/api/common/models -H "Authorization: Bearer $CRS_OAI_KEY"` (应秒回模型列表 JSON)。
4. 启动 Codex：`codex -m gpt-5.6-sol`。
5. 发送测试指令：`你好`。

## 5. 最终验证结果

- ✅ **连接状态**：Codex 成功连接至 `https://nexusacc.itssx.com/api/codex/codex`。
- ✅ **响应速度**：发送消息后，模型即时响应，无长时间等待。
- ✅ **Token 消耗**：`/status` 显示 Token 正常计数。
- ✅ **防火墙状态**：Windows 防火墙保持开启，系统安全不受影响。
- ✅ **分流效果**：Codex 直连 itssx**(Codex指运行在 WSL 里的 Codex CLI 客户端)**；`git clone github.com` 仍通过 Clash 代理拉取。

## 6. 经验总结

1. **WSL2 网络特殊性**：WSL2 与 Windows 宿主机的通信属于“网络间”通信，受多层防火墙管制（尤其是 Hyper-V 防火墙），不能简单等同于本机回环 (127.0.0.1)。
2. **no_proxy 的重要性**：在配置了全局代理的 WSL 环境中，对于国内可达的 API 服务（如 itssx ACC），使用 `no_proxy` 进行分流是最稳定、最简洁的解决方案，能有效规避复杂的防火墙配置。
3. **诊断技巧**：
   - 使用 `ip route | grep default` 获取真实的 Windows 主机 IP。
   - 对比 `curl` (直连) 与 `curl -x http://proxy` (代理) 的结果，定位网络阻塞点。
   - 利用 PowerShell 的 `Test-NetConnection` 和 WSL 的 `curl -v` 交叉验证端口连通性。
4. **Codex 配置**：在使用第三方中转时，务必确认 `preferred_auth_method = "apikey"`，`requires_openai_auth = false`，并将 Key 置于环境变量中，避免与旧的 OAuth 登录态冲突。

------

*本报告基于实际排查过程整理，旨在为后续环境配置提供参考。*