# Windows 无虚拟机本地部署 OpenCode 全指南：接入 Ollama 与避坑实录

## 前言
在 Windows 上部署 AI 编程助手，传统方案通常推荐 WSL2。然而作者并不会用Ubunton终端，如果你会用或者打开了虚拟化，可以用如下命令：
、、、powershell
# 安装Ubunton终端（WSL）
wsl --install -d Ubuntu

# 更新包列表
sudo apt update && sudo apt upgrade -y

# 安装 OpenCode（官方一键脚本）
curl -fsSL https://opencode.ai/install | bash
、、、
（当然，使用npm安装也可以）
但并不是所有电脑都开启了虚拟化，且 WSL 也会额外消耗系统资源。本文不使用WSL或者Docker，基于windows11原生环境安装opencode，记录了如何将 OpenCode 与本地 Ollama 模型（我部署了两个模型来使用，这里以Qwen2.5-Coder-7B 为例）结合，打造一款完全属于你自己的本地 AI 编码助手。

**⚠️ 硬件配置参考**：我的电脑时 Intel i7-14650HX + NVIDIA RTX 4060 (8GB 显存) + 16GB 内存的笔记本环境，电脑配置高低只和模型大小相关，按照本文的Ollama的安装和OpenCode的安装过程不会因为电脑的配置而安装失败。但建议低配置的电脑不要部署过大的模型，8GB 显存是运行 7B 级别量化模型（如 Q4_K_M 格式）的安全底线。

---

## 一、 环境准备与 OpenCode 原生安装

### 1.1 安装 Node.js 与 Git
OpenCode 依赖 Node.js 运行，依赖 Git 进行版本控制。请前往官网下载 Windows 的 `.msi` 安装包。建议安装 Node.js LTS 版本（v20以上）。

### 1.2 自定义路径安装 OpenCode
为了避免污染 C 盘，我们尝试将 OpenCode 临时安装到 `F:\OpenCode`。这里需要特别注意 npm 的路径参数。

**正确命令**（在 Git Bash 或 PowerShell 中执行）：
```bash
npm install -g opencode-ai@latest --prefix "F:/OpenCode" --registry=https://registry.npmjs.org
```

**🚨 避坑点 1：postinstall 安全警告**
安装完成后，npm 可能会报出 `npm warn install-scripts` 警告。这是因为新版 npm 默认拦截了第三方包的 `postinstall` 脚本。如果不放行，OpenCode 将无法运行。
**解决方案**：再次执行以下命令，允许特定包运行脚本：
```bash
npm install -g opencode-ai --allow-scripts=opencode-ai --prefix "F:/OpenCode" --registry=https://registry.npmjs.org
```

---

## 二、 配置 Ollama 与下载本地模型

### 2.1 配置 Ollama 环境变量
为了不让模型撑爆 C 盘，请将 Ollama 的模型存储路径转移到 F 盘。
在系统环境变量中添加：
*   `OLLAMA_MODELS` = `F:\Ollama\models`
*   随后重启 Ollama 服务。

### 2.2 绕过网络限制拉取模型
这是整个部署过程中最容易卡住的地方。

**🚨 避坑点 3：TLS 证书校验失败**
直接执行 `ollama pull qwen2.5-coder:7b` 时，由于国内网络环境的干扰，往往会报错：`tls: failed to verify certificate: x509: certificate is not valid for any names...`，导致下载到 26% 左右中断。

**解决方案 A（临时跳过证书校验）**：
在 CMD 中执行：
```cmd
set OLLAMA_INSECURE_REGISTRY=1
ollama pull qwen2.5-coder:7b
```

**解决方案 B（强烈推荐：使用国内 ModelScope 镜像）**：
```cmd
ollama pull modelscope.cn/Qwen/Qwen2.5-Coder-7B-Instruct-GGUF
```
该命令速度极快（实测可达 6.4 MB/s），我是用的时解决方案A和B同时使用，不确定解决方案B单独使用是否可行，建议加上临时跳过证书校验的代码，关闭cmd界面后不会影响其余界面和后续命令的证书校验。

**🚨 避坑点 4：下载残留文件（垃圾文件）**
如果在下载失败后决定换源，之前中断下载产生的所有 `sha256-xxxx` 无后缀文件，都会残留在 `F:\Ollama\models\blobs` 目录下。
**解决方案**：退出 Ollama，直接清空 `blobs` 目录，干净地重新开始。但需要注意的是如果之前已经安装好了模型，例如我在此之前本地部署了DeepSeek-R1-0528-Qwen3-8B-GGUF模型，这时候如果清空blobs目录就会连带原有的所有模型都失效。这里给出两个解决方案：
1.按照文件下载日期，删除今日下载的文件即可。
2.如果很可惜你在同一天安装了模型，按照前缀来辨别。例如：我的电脑中“sha256-60e05f...”的前缀是Qwen2.5-Coder-7B-Instruct-GGUF模型的相关下载文件，而“sha256-509287”是部署的DeepSeek-R1-0528-Qwen3-8B-GGUF模型。按照前缀删除即可。

### 2.3 给模型改个短名字
从 ModelScope 拉取下来的模型名字很长，执行以下命令复制一个短名字：
```cmd
ollama cp modelscope.cn/Qwen/Qwen2.5-Coder-7B-Instruct-GGUF qwen2.5-coder:7b
```
*(注：`ollama cp` 相当于创建硬链接，不额外占用双倍硬盘空间，底层数据是共享的。)*

---

## 三、 OpenCode 接入本地模型

### 3.1 找到神秘的配置文件
教程常说的 `~/.config/opencode/opencode.json` 在 Git Bash 中看似是个 Linux 路径，但在 Windows 中真实物理路径是：
`C:\Users\你的用户名\.config\opencode\opencode.json`
*(注：它是一个隐藏文件夹，需要在资源管理器中开启“显示隐藏项目”才能看到。)*

### 3.2 写入配置
打开该文件，写入：
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (Local)",
      "options": {
        "baseURL": "http://127.0.0.1:11434/v1"
      },
      "models": {
        "qwen2.5-coder:7b": {
          "name": "Qwen2.5 Coder 7B"
        }
      }
    }
  }
}
```
*(注：使用 `127.0.0.1` 比 `localhost` 更稳定，能绕过 IPv6 解析问题。)*

---

## 四、 OpenCode 使用必读与操作技巧

### 4.1 TUI 界面操作
**🚨 避坑点 5：不要双击 exe 打开**
直接双击 `opencode.exe` 打开 Windows 默认控制台，会导致 TUI 界面内**方向键、Tab 键完全失灵**，无法选中 `ollama` 选项。
**解决方案**：**在 Git Bash 中启动**。在终端中 `cd` 到你的项目目录，然后执行 `/f/OpenCode/opencode.cmd`，方向键即可恢复灵敏。

**🚨 避坑点 6：复制粘贴快捷键**
在 OpenCode 中，`Ctrl+V` 无法粘贴。请使用：
*   复制：鼠标拖拽选中松开即复制（然而实际上还是拖拽选中松开再右键一下才会显示copied，建议复制的时候多右键一下会更方便）；或 `Ctrl+X` 后按 `Y`复制最后一次（latest）AI回复的内容。
*   粘贴：`Ctrl+Shift+V` 或 `Shift+Insert`。

### 4.2 理解上下文窗口限制
OpenCode 右下角会显示类似 `8.2k (4%)` 的数字。**这是一个极易产生误导的提示**。
*   `8.2k` 是当前已消耗的 Token 数。
*   `4%` 是 OpenCode 默认按云端大模型（如 200k 上下文）计算的占比。
*   **警告**：你的 RTX 4060 (8GB) 跑本地 7B 模型时，Ollama 的实际最大上下文（Context Length）通常只能承受 **8k**。一旦 `8.2k` 的绝对值超过 8k，你的本地模型会报错，或者默默丢弃最前面的对话，导致 AI “失忆”。
*   **建议**：不要在同一个会话里聊几十轮。发现 `8.2k` 接近上限时，新建会话。

### 4.3 更换模型
要切换模型（例如切回 `deepseek-r1:8b`），直接修改 `opencode.json` 的 `models` 字段，或者将 `models` 里的模型名改成 `deepseek-r1:8b`。重启 OpenCode 即可。
当然，实际操作的时候发现再OpenCode界面使用/model命令可以找到Ollama下的所有模型，可以不用修改opencode.json文件来切换模型。
---

## 五、 附录：如果还想跑 Hermes Agent
如果在原生 Windows 下还想尝试安装 Hermes Agent会比较麻烦，主要是国内环境访问国外网址时加载超时的问题。我自己在部署的时候遇到的问题主要是第二点网络卡点，哪怕挂梯子也可能出现HTTPS 克隆被中途切断，SSH 连接 GitHub 超时等问题。不过都是比较好解决或者避开的问题，就不单开一篇博客来写了，主要注意下面两个问题就行：
1.  **必须使用便携版 Python 环境**：Hermes 脚本会强行在安装目录中创建虚拟环境。
2.  **网络卡点**：脚本会尝试从 GitHub 克隆源码并下载 `uv`。
    *   如果卡在 `Installing managed uv`：在 Git Bash 中用 `pip install uv` 装好，然后手动将 `uv.exe` 复制到 `F:\HermesAgent\bin` 目录下，骗过安装脚本。
    *   如果卡在 Git Clone：直接在浏览器下载 ZIP 源码包，解压重命名为 `hermes-agent`，放到 `F:\HermesAgent\` 目录下，再重新运行安装脚本，即可跳过下载直接配环境。


---
