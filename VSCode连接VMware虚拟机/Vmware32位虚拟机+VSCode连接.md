# VS Code 连接 32 位 VMware 虚拟机 (Ubuntu) 开发配置指南

## 1. 背景与问题说明

在使用 VS Code 远程连接 VMware 中的 32 位 Ubuntu 虚拟机（如 `ubuntu-16.04.4-desktop-i386`）时，直接使用官方的 `Remote - SSH` 插件会失败，并提示 `Unsupported architecture: i686`（不支持远程主机的体系结构）。

**失败原因：**
官方 `Remote - SSH` 插件需要在目标虚拟机上运行一个 "VS Code Server" 后端程序，而该程序目前仅支持 64 位（x86_64 或 ARM64）系统，无法在 32 位（i686）系统上运行。失败的不是 SSH 协议，而是微软的服务端程序无法启动。

## 2. 解决方案概述

为了在不重装 64 位系统的前提下，依然获得丝滑的开发体验，我们采用 **SFTP 代码同步 + 终端原生 SSH 编译** 的方案。

* **代码编辑**：在 Windows 本地编辑代码，享受 VS Code 本地插件的完整支持（如代码高亮、瞬间跳转、自动补齐等）。
* **代码同步**：利用 `SFTP` 扩展，每次按下 `Ctrl+S` 保存时，自动将修改的代码瞬间同步到虚拟机对应的目录中。
* **代码运行**：在 VS Code 下方的集成终端中，使用系统原生的 SSH 命令连接虚拟机进行 `make` 编译和调试。

---

## 3. 详细配置步骤

### 步骤一：在虚拟机中安装并开启 SSH 访问
在虚拟机的终端中，依次执行以下命令来安装开启 `openssh-server`，并获取虚拟机的 IP 地址：

```bash
# 1. 更新软件包列表
sudo apt-get update

# 2. 安装 openssh-server 服务
sudo apt-get install openssh-server -y

# 3. 启动 ssh 服务并设置开机自启
sudo systemctl start ssh
sudo systemctl enable ssh

# 4. 查看虚拟机的 IP 地址 (需找到类似 192.168.100.142 的地址)
ip addr
```
*(假设获取到的虚拟机 IP 为 `192.168.100.142`，用户名为 `wangxu`)*

### 步骤二：准备 Windows 本地工作区
1. 在 Windows 上（例如桌面）创建一个空文件夹，例如命名为 `PA-Workspace`。
2. 打开 VS Code，选择“文件” -> “打开文件夹”，打开刚才创建的 `PA-Workspace` 文件夹。

### 步骤三：安装并配置 SFTP 扩展
1. 在 VS Code 左侧边栏打开“扩展”（`Ctrl+Shift+X`）。
2. 搜索 `SFTP`，推荐安装作者为 **Natizyskunk** 的 SFTP 扩展。
3. 按下 `Ctrl+Shift+P` 打开命令面板，输入并选择 `SFTP: Config`。会在工程下自动生成一个 `.vscode/sftp.json` 配置文件。
4. 将生成的文件修改为以下内容并保存：

```json
{
    "name": "NKU-PA-Ubuntu",
    "host": "192.168.100.142",      // 替换为你的虚拟机 IP
    "protocol": "sftp",
    "port": 22,
    "username": "wangxu",           // 替换为你的虚拟机用户名
    "password": "你的虚拟机密码",    // 替换为你的密码
    "remotePath": "/home/wangxu/ics2017", // 虚拟机中大作业代码的完整存放路径
    "uploadOnSave": true,           // 开启保存时自动上传
    "useTempFile": false,
    "openSsh": false
}
```

### 步骤四：拉取虚拟机代码到本地
配置保存后，在 VS Code 左侧资源管理器（Explorer）空白处右键，选择 **`SFTP: Download Project/Download Folder`**。  
此时，VS Code 会将虚拟机 `/home/wangxu/ics2017` 目录下的所有文件完整下载到 Windows 的 `PA-Workspace` 中。

### 步骤五：在 VS Code 终端中使用 SSH 连接与编译
1. 在 VS Code 顶部菜单栏选择“终端” -> “新建终端”（或按快捷键 `` Ctrl + ` ``）。
2. 在下方弹出的终端中输入原生 SSH 连接命令：
   ```bash
   ssh wangxu@192.168.100.142
   ```
   *(首次连接需输入 `yes` 确认指纹，随后输入虚拟机密码)*
3. 连接成功后，进入你的项目目录，即可在虚拟机环境中进行编译测试：
   ```bash
   cd ~/ics2017
   make
   ```

---

## 4. 终极日常开发工作流总结

在完成上述全部配置后，你的日常 PA 实验开发流程将变得极其简单、高效：

1. **写代码**：在 VS Code 的代码编辑区中愉快编写代码。
2. **保存同步**：按下 `Ctrl + S` 保存，代码会自动、无感地上传到虚拟机（可通过左下角状态栏确认上传完成）。
3. **编译运行**：鼠标点击下方的 SSH 终端窗口，按 `↑` 键调出上一次的 `make` 命令，回车执行，查看运行并查看结果。
