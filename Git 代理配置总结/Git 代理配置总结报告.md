# Git 代理配置总结报告

## 1. 背景

在使用 Git 从 GitHub 克隆仓库时，遇到以下错误：

```
fatal: unable to access 'https://github.com/xxx/xxx.git/':
  Recv failure: Connection was reset
```

或

```
fatal: unable to access 'https://github.com/xxx/xxx.git/':
  Failed to connect to github.com port 443 after ... ms: Could not connect to server
```

**环境特征**：

- 操作系统：Windows
- 终端：PowerShell（VSCode 内置终端）
- 网络环境：开启了 Clash 代理（系统代理模式，混合端口 7897）
- 浏览器可正常访问 GitHub，但 Git 命令行无法连接

------

## 2. 问题根因分析

| 现象                                                       | 原因                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| 浏览器可访问 GitHub                                        | 浏览器自动读取系统代理设置，流量经 Clash 转发                |
| `Test-NetConnection github.com -Port 443`显示 TCP 连接成功 | 该命令不经过系统代理，直接发起 TCP 四次握手，但 TLS 层握手被中间设备（DPI/防火墙）重置 |
| Git clone 失败                                             | Git 默认**不读取系统代理**，也不读取环境变量 `HTTP_PROXY`，直接发起 HTTPS 连接，被网络阻断 |

**核心结论**：

- Clash 的系统代理模式仅影响“主动读取系统代理”的应用（如浏览器），Git 不属于此类。
- 需要显式为 Git 配置代理，使其流量经过 Clash 的本地代理端口。

------

## 3. 解决方案

### 3.1 全局代理（所有 Git 远程走代理）

```
git config --global http.proxy http://127.0.0.1:7897
git config --global https.proxy http://127.0.0.1:7897
```

- 优点：简单粗暴，一条命令解决所有 Git HTTPS 请求。
- 缺点：**所有** Git 远程仓库（包括Github、内网 GitLab、Gitee、Bitbucket、还是你自己搭建的 Git 服务器 等）都会走代理，可能造成不必要的延迟或连接失败。

### 3.2 精确代理（仅 GitHub 走代理）

```
git config --global http.https://github.com.proxy http://127.0.0.1:7897
```

- 原理：Git 支持基于 URL 的配置覆盖，`http.https://github.com.proxy`表示“仅对以 `https://github.com/`开头的 URL 生效”。
- 优点：其他 Git 远程仓库保持直连，互不干扰。
- 注意：只需配置这一条，因为 GitHub 远程地址均为 HTTPS 协议。

------

## 4. 代理配置的查看与删除

### 4.1 查看当前代理配置

```
git config --global --get http.proxy
git config --global --get https.proxy
git config --global --get http.https://github.com.proxy
```

### 4.2 删除全局代理

```
git config --global --unset http.proxy
git config --global --unset https.proxy
```

### 4.3 删除精确代理

```
git config --global --unset http.https://github.com.proxy
```

### 4.4 删除所有代理配置（批量）

```
git config --global --list | Select-String "proxy" | ForEach-Object {
    $key = ($_ -split "=")[0]
    git config --global --unset $key
}
```

------

## 5. 环境变量与环境检测

- **系统环境变量 `HTTP_PROXY`/ `HTTPS_PROXY`**：Git 默认**不读取**，但某些第三方工具（如 `curl`）会读取。

- 检查方法：

  ```
  echo $env:HTTP_PROXY
  echo $env:HTTPS_PROXY
  ```

- 若存在值且不希望使用，可临时清除：

  ```
  $env:HTTP_PROXY = $null
  $env:HTTPS_PROXY = $null
  ```

------

## 6. 其他常见排查项

### 6.1 代理端口确认

- Clash 默认混合端口通常为 `7890`或 `7897`，可在 Clash 面板中查看。
- 其他 VPN（V2RayN、Shadowsocks 等）端口各异，可通过 `netstat -ano | findstr LISTENING`查找。

------

## 7. 最佳实践建议

| 场景                                | 推荐配置                                 |
| ----------------------------------- | ---------------------------------------- |
| 仅需访问 GitHub，其他仓库直连       | 精确代理 `http.https://github.com.proxy` |
| 所有 Git 远程都在海外（或均需代理） | 全局代理 `http.proxy`+ `https.proxy`     |
| 希望彻底规避代理问题                | 使用 SSH 协议克隆                        |
| 频繁切换网络环境                    | 编写批处理脚本一键启用/禁用代理          |

**示例脚本（PowerShell）**：

```
# 启用 GitHub 代理
function Enable-GitProxy {
    git config --global http.https://github.com.proxy http://127.0.0.1:7897
    Write-Host "✅ GitHub proxy enabled"
}

# 禁用所有代理
function Disable-GitProxy {
    git config --global --unset http.https://github.com.proxy 2>$null
    git config --global --unset http.proxy 2>$null
    git config --global --unset https.proxy 2>$null
    Write-Host "✅ All Git proxies disabled"
}
```

------

## 8. 总结

- **Git 不读系统代理**，这是导致浏览器能访问但 Git 不能的根本原因。
- 通过 `git config`显式设置代理，可以解决 Git 的 HTTPS 连接问题。
- **精确代理配置**（`http.https://github.com.proxy`）比全局代理更优雅，推荐作为首选。
- 使用 SSH 协议可完全绕过代理问题，适合长期稳定的开发环境。
- 掌握配置的查看与清理方法，便于在不同网络环境下灵活切换。

------

*本报告基于 Windows + Clash 系统代理模式 + Git for Windows 环境实测整理。*