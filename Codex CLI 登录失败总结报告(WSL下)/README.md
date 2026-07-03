### 2026.7.3 总结："Codex CLI 登录失败总结报告.md"

因为在尝试利用 Codex CLI 进行 PPT 制作时，发现怎么都无法完成登录，始终报错：

```
Token exchange failed: token endpoint returned status 403 Forbidden: Country, region, or territory not supported
```

- 初始时以为是梯子或网络问题，但在浏览器中可以正常访问 OpenAI 网页，而 Codex 登录却始终卡在最后一步。

经调试发现根本原因是 **WSL 中 Codex CLI 的 Node.js 进程（undici 库）默认不读取环境变量 `http_proxy`**，导致 token exchange 请求直接通过 WSL 的大陆 IP 出口发出，被 OpenAI 拒绝。通过以下方式解决：

- 在 `~/.bashrc` 中补充完整的代理环境变量（`http_proxy`、`https_proxy`、`all_proxy`、`no_proxy`）
- 添加关键变量 `NODE_USE_ENV_PROXY=1`，强制 undici 读取环境代理
- 重启 WSL 使所有进程重新加载配置

因此在当时进行了问题发现、解决与总结。

