# Codex CLI 登录失败（Token Exchange Failed 403）总结报告

## 1. 背景

在使用 WSL（Ubuntu）环境下的 Codex CLI 进行登录时，遇到以下错误：

```
Sign-in could not be completed
Token exchange failed: token endpoint returned status 403 Forbidden: Country, region, or territory not supported
Error code: token_exchange_failed
```

**环境特征**：

- 操作系统：Windows 11 + WSL2（Ubuntu）

- 终端：WSL 内 Bash

- 网络环境：Windows 宿主机开启了 Clash 代理（混合端口 7897，允许 LAN 连接）

- 浏览器可正常访问 chat.openai.com，但 Codex CLI 登录始终失败

- Git 在 WSL 中已通过 `git config --global http.proxy`配置代理，可正常访问 GitHub

------

## 2. 问题根因分析

| 现象                                                         | 原因                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 浏览器可访问 OpenAI 网页                                     | 浏览器自动读取系统代理设置，流量经 Clash 转发                |
| `curl -I https://auth.openai.com`返回 `200 Connection established` | curl 读取环境变量 `http_proxy`，代理正常工作                 |
| Codex login 报 403 token_exchange_failed                     | Codex CLI 使用 Node.js 的 **undici** 库发送 token exchange 请求，而 undici **默认不读取环境变量 `http_proxy`**，导致请求直接通过 WSL 公网出口（中国大陆 IP）发出，被 OpenAI 拒绝 |
| WSL 中 `git config --global http.proxy`不影响 Codex          | Git 配置仅影响 Git 自身，Codex 是独立的 Node.js 进程         |

**核心结论**：

- WSL 中 `curl`等工具走代理是因为它们读取 `http_proxy`环境变量。

- Codex CLI 的底层 HTTP 库（undici）**不读取**这些环境变量，需要额外开关 `NODE_USE_ENV_PROXY=1`才能生效。

- 仅靠 `source ~/.bashrc`可能不够，因为 Codex 可能已有后台进程未继承新变量，**重启 WSL 能确保所有进程重新加载环境变量**。

------

## 3. 解决方案

### 3.1 在 `~/.bashrc`中补充完整的代理环境变量

将以下内容追加到 `~/.bashrc`末尾：

```
# 固定 Windows 宿主机 IP（WSL 中可通过 ip route 动态获取，此处直接写已知地址）
export WIN_HOST=172.22.80.1

# 标准 HTTP/HTTPS 代理变量（大小写同时设置，兼容不同工具）
export http_proxy="http://$WIN_HOST:7897"
export https_proxy="http://$WIN_HOST:7897"
export HTTP_PROXY="http://$WIN_HOST:7897"
export HTTPS_PROXY="http://$WIN_HOST:7897"
export all_proxy="http://$WIN_HOST:7897"
export ALL_PROXY="http://$WIN_HOST:7897"

# 本地回环地址不走代理（避免 OAuth 回调失败）
export no_proxy="localhost,127.0.0.1,::1"
export NO_PROXY="localhost,127.0.0.1,::1"

# 让 Node.js undici 库读取环境代理变量（关键！）
export NODE_USE_ENV_PROXY=1

# 给老版本 Node 的 global-agent 兜底（可选）
export GLOBAL_AGENT_HTTP_PROXY="http://172.22.80.1:7897"
```

### 3.2 重新加载配置并重启 WSL

```
source ~/.bashrc
```

然后**关闭所有 WSL 窗口并重新打开**（或执行 `wsl --shutdown`再重新进入），以确保所有进程继承新环境变量。

### 3.3 验证代理是否生效

```
# 检查出口 IP 是否为支持区域（非中国大陆）
curl -s https://api.ipify.org && echo

# 检查能否连接 auth.openai.com
curl -I https://auth.openai.com
```

预期输出：出口 IP 应为日本/新加坡/美国等区域，`auth.openai.com`返回 `200 Connection established`。

### 3.4 重新登录 Codex

```
codex logout
codex login
```

此时浏览器弹出授权页面，完成授权后即可成功登录。

------

## 4. 各环境变量的作用详解

| 变量名                      | 作用域           | 说明                                                         |
| --------------------------- | ---------------- | ------------------------------------------------------------ |
| `http_proxy`/ `https_proxy` | Unix 传统        | 被 `curl`, `wget`, Python `requests`等工具读取               |
| `HTTP_PROXY`/ `HTTPS_PROXY` | 大写变体         | 被 Node.js 原生 `http.request`、Java JVM 等读取              |
| `all_proxy`/ `ALL_PROXY`    | 通用代理         | 对 `ftp://`、`ws://`等非 HTTP 协议也生效，Codex 可能使用 WebSocket |
| `no_proxy`/ `NO_PROXY`      | 豁免列表         | 防止 OAuth 本地回调（`127.0.0.1`）被代理转发                 |
| `NODE_USE_ENV_PROXY=1`      | **Node.js 专属** | 告诉 undici 库自动读取 `http_proxy`环境变量，**此变量是解决 Codex 问题的关键** |
| `GLOBAL_AGENT_HTTP_PROXY`   | global-agent     | 部分 Node 应用使用 `global-agent`库，提供兜底代理配置        |

------

## 5. 为什么重启 WSL 后才成功？

- `source ~/.bashrc`仅影响当前 shell 及其后续子进程。

- Codex 的 OAuth 流程中，浏览器授权后会启动一个新的后台进程（或复用守护进程）执行 token exchange。若该进程在 `source`之前已运行，则未继承新变量。

- **重启 WSL 会终止所有进程并重新初始化**，确保 `.bashrc`中的变量被所有新进程读取。

替代方案（不重启 WSL）：

```
pkill -f codex          # 杀死所有 codex 相关进程
source ~/.bashrc
codex login
```

------

## 6. 备用方案：直接使用 API Key 绕过 OAuth

如果您有 OpenAI API Key，可以完全避开 OAuth 的地域限制：

```
export OPENAI_API_KEY="sk-..."
printenv OPENAI_API_KEY | codex login --with-api-key
```

此方式直接调用 `api.openai.com`，不走 `auth.openai.com/oauth/token`，不受地域锁影响。

------

## 7. 其他常见排查项

### 7.1 确认 Windows 宿主机 Clash 设置

- **Allow LAN** 必须开启（允许局域网连接）。

- 规则中不要屏蔽 `auth.openai.com`或 `*.openai.com`，建议设为 `DIRECT`或走代理节点。

- 端口 `7897`需与 WSL 中配置一致。

### 7.2 清除旧认证缓存

```
rm -rf ~/.codex/auth.json
```

------

## 8. 最佳实践建议

| 场景                        | 推荐配置                                                     |
| --------------------------- | ------------------------------------------------------------ |
| 仅需 Codex CLI 登录         | 在 `.bashrc`中补充完整代理变量 + `NODE_USE_ENV_PROXY=1`      |
| 希望彻底规避 OAuth 地域限制 | 使用 API Key 登录（`codex login --with-api-key`）            |
| 同时使用 Git 和 Codex       | 保留 `git config --global http.proxy`+ 上述环境变量（两者独立） |
| 频繁切换网络环境            | 编写脚本一键启用/禁用代理变量                                |
| 不想修改 `.bashrc`          | 每次登录前手动 export 上述变量（不推荐）                     |

**示例脚本（Bash）**：

```
# 启用 Codex 代理
function enable_codex_proxy() {
    export WIN_HOST=172.22.80.1
    export http_proxy="http://$WIN_HOST:7897"
    export https_proxy="http://$WIN_HOST:7897"
    export HTTP_PROXY="http://$WIN_HOST:7897"
    export HTTPS_PROXY="http://$WIN_HOST:7897"
    export all_proxy="http://$WIN_HOST:7897"
    export ALL_PROXY="http://$WIN_HOST:7897"
    export no_proxy="localhost,127.0.0.1,::1"
    export NO_PROXY="localhost,127.0.0.1,::1"
    export NODE_USE_ENV_PROXY=1
    export GLOBAL_AGENT_HTTP_PROXY="http://172.22.80.1:7897"
    echo "✅ Codex proxy enabled"
}

# 禁用所有代理
function disable_codex_proxy() {
    unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
    unset all_proxy ALL_PROXY
    unset no_proxy NO_PROXY
    unset NODE_USE_ENV_PROXY
    unset GLOBAL_AGENT_HTTP_PROXY
    echo "✅ Codex proxy disabled"
}
```

------

## 9. 总结

- **Codex CLI 的 token exchange 失败（403）** 的根本原因是其底层 Node.js 的 undici 库不读取环境变量 `http_proxy`，导致请求从中国大陆 IP 发出被拒。

- **解决方案**：在 `.bashrc`中设置 `NODE_USE_ENV_PROXY=1`以及完整的代理环境变量，并重启 WSL 使所有进程生效。

- **关键变量**：`NODE_USE_ENV_PROXY=1`是专用于 undici 的开关，是解决此问题的核心。

- **备用方案**：使用 API Key 登录可完全绕过 OAuth 地域限制，更稳定。

- 本方案已在 WSL2 + Ubuntu + Clash 环境下实测通过。

------

*本报告基于 Windows 11 + WSL2 (Ubuntu) + Clash 代理 + Codex CLI 环境实测整理。*