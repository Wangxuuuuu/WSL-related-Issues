### 2026.7.18 总结："WSL 下 Codex CLI 连接中转服务故障排查与解决方案报告.md"

因为在尝试使用中转站的key来连接使用Codex时,发现怎么都无法进行对话:

```
向codex输入1,但是始终无任何响应
```

初始时以为是我没有配置好中转站的一些配置或者key异常,但我们在终端中可以正常启动codex,以及使用/status,而尝试使用 Codex 进行对话时，却始终没有任何响应。于是我们发现 curl 测试发现网络链路存在阻塞。

经调试发现根本原因是 **WSL2 的 Hyper-V 防火墙会拦截来自 WSL 子网的入站流量**，即使常规 Windows Defender 防火墙已放行 Clash 代理端口，WSL 发往 Windows 宿主机的代理请求仍会被阻断，和此前 ChatGPT 网页登录时 undici 不读取代理的问题成因不同。

通过以下分流策略解决：

- 在 `~/.bashrc` 中,将 itssx 相关域名加入 `no_proxy` 环境变量，让 Codex CLI 直连中转服务，其他流量（如 git 操作）仍走 Clash 代理
- 同时调整 `config.toml` 适配 apikey 鉴权模式，清理旧版 OAuth 登录态。

因此在当时进行了问题发现、解决与总结。