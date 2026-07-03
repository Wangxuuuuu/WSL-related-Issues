# WSL Git 代理配置及路径兼容性问题解决报告

## 1. 背景

**初始场景**：在 Windows 主机上使用 Git 克隆个人仓库 `https://github.com/Wangxuuuuu/Data-Security.git`时，遇到两个错误：

- **错误1**：`error: invalid path '实验二/代码框架/.../02-19_03:04/arithmetic_evaluation_1.csv'`（Windows 文件系统不支持冒号 `:`）
- **错误2**：`fatal: unable to checkout working tree`（部分文件无法检出）

**解决思路**：由于仓库中存在大量带冒号的路径（性能日志目录），决定在 **WSL（Windows Subsystem for Linux）** 中克隆仓库，修复这些路径后推送至远程，从而让 Windows 端能够正常检出。

**然而**，在 WSL 中执行 `git clone`时立即遇到了新的问题：

```
fatal: unable to access 'https://github.com/Wangxuuuuu/Data-Security.git/': 
  gnutls_handshake() failed: Error in the pull function.
```

**核心矛盾**：Windows 浏览器可正常访问 GitHub（通过 Clash 系统代理），但 WSL 中的 Git 无法连接。

------

## 2. 问题根因分析

| 现象                                                         | 原因                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Windows 浏览器可访问 GitHub                                  | 浏览器自动读取系统代理设置，流量经 Clash 转发                |
| WSL 中 `git clone`失败（gnutls_handshake）                   | WSL 是独立于 Windows 的 Linux 子系统，**不继承 Windows 系统代理**，Git 直接发起 HTTPS 连接被网络阻断 |
| WSL 中 `curl -x http://127.0.0.1:7897`显示 `Connection reset` | WSL 的 `127.0.0.1`指向自身，并非 Windows 宿主机；Clash 运行在 Windows 上，需通过宿主机 IP 访问 |
| 使用宿主机 IP `172.22.80.1:7897`后仍 `reset`                 | Clash 默认只绑定 `127.0.0.1`，未开启“允许局域网连接”         |
| 开启 Allow LAN 后，WSL 中 `curl`卡住（无响应）               | Windows 防火墙阻止了来自 WSL 虚拟网卡的入站 TCP 连接         |
| 添加防火墙规则后 `EnforcementStatus: NATInboundRuleNotApplicable` | WSL2 使用 NAT 网络模式，Hyper-V 防火墙的入站规则对该模式不适用 |
| 最终通过 `Set-NetFirewallProfile -DisabledInterfaceAliases`解决 | 对 WSL 虚拟网卡单独关闭防火墙，允许其访问 Windows 本地服务   |

**核心结论**：

- WSL 的网络栈与 Windows 隔离，需要**显式配置代理**并打通 Windows 防火墙。
- WSL2 的 NAT 模式使得传统防火墙规则难以生效，需采用更直接的策略（如关闭虚拟网卡防火墙或使用端口转发）。

------

## 3. 解决方案（逐步排查与最终方案）

### 3.1 获取 Windows 宿主机在 WSL 中的 IP

在 WSL 中运行：

```
ip route | grep default | awk '{print $3}'
```

输出示例：`172.22.80.1`（此为 Windows 宿主机在 WSL 虚拟网络中的网关地址）。

### 3.2 配置 Git 代理（使用宿主机 IP）

```
git config --global http.proxy http://172.22.80.1:7897
git config --global https.proxy http://172.22.80.1:7897
```

> 注：Clash 的混合端口为 7897，已在 Windows 上开启“允许局域网连接”。

### 3.3 解决 Windows 防火墙拦截

#### 3.3.1 临时关闭防火墙验证

在 Windows 管理员 PowerShell 中：

```
netsh advfirewall set allprofiles state off
```

在 WSL 中测试：

```
curl -x http://172.22.80.1:7897 -I https://github.com
```

返回 `HTTP/2 200`确认是防火墙问题。

#### 3.3.2 最终方案：仅对 WSL 虚拟网卡禁用防火墙

```
Set-NetFirewallProfile -Name Public -DisabledInterfaceAliases "vEthernet (WSL (Hyper-V firewall))"
```

> 该命令将 WSL 虚拟网卡从 Public 配置文件中移除，其他网卡（WiFi、以太网）的防火墙保持不变，安全性高。

### 3.4 验证代理连通性

```
timeout 5 bash -c 'echo > /dev/tcp/172.22.80.1/7897' && echo "TCP OK"
curl -x http://172.22.80.1:7897 -I https://github.com
```

预期输出 `TCP OK`和 `HTTP/2 200`。

### 3.5 克隆仓库并修复路径问题

```
cd ~
git clone https://github.com/Wangxuuuuu/Data-Security.git
cd Data-Security
```

#### 3.5.1 删除带冒号的日志目录

```
rm -rf 实验二/代码框架/zk_lab_exp31/deps/libsnark/depends/libfqfft/libfqfft/profiling/logs
```

#### 3.5.2 提交并推送

```
git add -A
git commit -m "remove profiling logs (contain ':' in paths)"
git push origin main
```

### 3.6 Windows 端重新克隆

```
cd D:\temp
git clone https://github.com/Wangxuuuuu/Data-Security.git
```

此时不再报 `invalid path`错误，完整检出成功。

------

## 4. 代理配置的查看与删除

### 4.1 查看当前代理配置

```
git config --global --get http.proxy
git config --global --get https.proxy
```

### 4.2 删除全局代理

```
git config --global --unset http.proxy
git config --global --unset https.proxy
```

### 4.3 删除所有代理配置（批量）

```
git config --global --list | grep proxy | while IFS='=' read key value; do
    git config --global --unset "$key"
done
```

------

## 5. 其他常见排查项

### 5.1 确认 Clash 的“允许局域网连接”已开启

- Clash Verge：设置 → 允许局域网连接（Allow LAN）
- Clash for Windows：General → Allow LAN ✅

### 5.2 确认 WSL 的 DNS 解析正常

```
cat /etc/resolv.conf | grep nameserver
```

若 `nameserver`不是宿主机 IP，可手动添加：

```
echo "172.22.80.1 host.docker.internal" | sudo tee -a /etc/hosts
```

但更稳定的做法是直接使用 IP 地址配置代理（如 `172.22.80.1:7897`），因为 WSL 重启后宿主机 IP 可能变化。

### 5.3 使用 `host.docker.internal`替代 IP（可选）

如果希望避免 IP 变化，可在 WSL 的 `/etc/hosts`中添加映射：

```
echo "172.22.80.1 host.docker.internal" | sudo tee -a /etc/hosts
```

然后配置：

```
git config --global http.proxy http://host.docker.internal:7897
git config --global https.proxy http://host.docker.internal:7897
```

> 注意：WSL 重启后 `/etc/hosts`可能被重置，需重新添加或使用脚本自动更新。

------

## 6. 最佳实践建议

| 场景                     | 推荐配置                                                     |
| ------------------------ | ------------------------------------------------------------ |
| 仅需在 WSL 中访问 GitHub | 使用 IP 地址配置全局代理，并关闭 WSL 虚拟网卡防火墙          |
| 希望避免手动配置防火墙   | 使用 `netsh interface portproxy`将 WSL 请求转发到 Windows 本地代理（较复杂） |
| 长期稳定使用             | 在 WSL 中编写启动脚本，自动获取宿主机 IP 并配置代理          |
| 彻底绕过代理问题         | 使用 SSH 协议克隆（需配置 SSH Key），不依赖 HTTP 代理        |

**示例脚本（WSL 启动时自动配置代理）：**

```
#!/bin/bash
HOST_IP=$(ip route | grep default | awk '{print $3}')
if [ -n "$HOST_IP" ]; then
    git config --global http.proxy "http://$HOST_IP:7897"
    git config --global https.proxy "http://$HOST_IP:7897"
    echo "Proxy configured to $HOST_IP:7897"
fi
```

将此脚本添加到 `~/.bashrc`或 `~/.zshrc`中，每次登录自动生效。

------

## 7. 总结

- **WSL 不继承 Windows 系统代理**，需要显式为 Git 配置代理。
- **Windows 防火墙** 会拦截来自 WSL 虚拟网卡的入站连接，需对 WSL 网卡单独关闭防火墙或添加精准规则。
- **WSL2 的 NAT 模式** 使得传统防火墙规则难以生效，`Set-NetFirewallProfile -DisabledInterfaceAliases`是最简洁有效的方案。
- **路径兼容性问题**（文件名含冒号）通过在 WSL 中删除或重命名后推送解决。
- 推荐使用 **IP 地址** 配置代理，避免 DNS 解析问题；如需使用域名，需手动维护 `/etc/hosts`。

*本报告基于 Windows 11 + WSL2 (Ubuntu) + Clash Verge (混合端口 7897) 环境实测整理。*