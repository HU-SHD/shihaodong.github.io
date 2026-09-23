#本地Python环境管理与配置：Anaconda环境移植Miniconda流程与注意事项（以本地环境Py39为实战全记录）

## 前言

我们在本地部署大模型、进行深度学习训练或计算机视觉开发时，Python 环境管理是绕不过去的第一道坎，并且第一道坎也是最大的坎，环境冲突、环境移植、环境管理、版本隔离……python环境贯穿整个项目。我在刚开始接触机器学习深度学习时，选择的是安装 Anaconda：它自带 Python、Conda、Jupyter、Spyder、大量科学计算库，能做到“开箱即用”。但后续真正进入多项目、多 CUDA 版本、多框架共存阶段后，Anaconda 的缺点会迅速暴露：体积臃肿、base 环境污染严重、官方源访问不稳定、SSL 报错频繁、系统级配置与用户级配置互相干扰、`PYTHONPATH` 污染导致 Conda 自身异常……这些问题在我这里全部都发生过，并且干扰过很久项目的进展。

本文将完整回顾我从 **彻底卸载 Anaconda** 到 **Miniconda 环境完美移植** 的全过程。以 `py39` 环境（PyTorch 2.5.1 + CUDA 12.1 + ONNX 等）为实战案例。

本文不仅给出命令，还会重点说明：

- 为什么推荐 Miniconda 而不是 Anaconda；
- 为什么必须清理 `defaults` 频道；
- 为什么不建议直接 `conda env create -f` 全量 YAML；
- 为什么 PyTorch 要用官方源而不是国内镜像；
- 为什么 `sympy`、`protobuf`、`opencv-python`、`numpy` 这些包必须锁版本；
- 为什么 `--no-deps` 有时候不是“危险操作”，而是保护核心框架的必要手段；
- 其他常见做法为什么看似省事，实际会带来更大的隐患。

---

## 一、为何放弃 Anaconda？（小错误不断）

在决定卸载 Anaconda 之前，我遇到了以下几个致命问题。它们并不是孤立的，而是 Anaconda 默认配置、Conda 依赖解析机制、Windows 多 Python 环境、国内网络环境共同作用的结果。

### 1.1 `repodata.json` 解析失败

执行：

```cmd
conda create -n dsh_project python=3.10 -y
```

Conda 报错：

```text
RuntimeError: Unable to read repodata JSON file 'https://repo.anaconda.com/pkgs/main/win-64'
```

这个错误的本质是：Conda 在创建环境前，需要先从配置的 channel 下载 `repodata.json`，也就是包索引元数据。如果这个文件下载失败、被代理截断、SSL 证书验证失败，或者镜像源返回了不完整内容，Conda 就无法解析依赖树。

**为什么会出现？**

 默认走 `repo.anaconda.com`，国内访问不稳定，同时请求可能被代理、杀毒软件、公司网络拦截；镜像源配置不完整，导致部分 channel 仍走官方源，`defaults` 与镜像源混用，元数据不一致

**其他做法的坏处：**

我尝试过只换一部分源，但是Conda 仍可能访问官方源，问题依旧，第三方库依旧无法成功下载，更不用说忽略SSL错误导致后续依赖解析更乱。不建议自己多次重复尝试，我自己在不断尝试的时候在 base 环境里乱装包，使得 Conda 自身依赖也变复杂，修复难度翻倍以至于最后无法修复，选择导出yaml文件完全卸载。


### 1.2 `defaults` 频道阴魂不散

明明在用户级 `.condarc` 中删除了 `defaults`，但执行：

```cmd
conda config --show channels
```

依然显示 `defaults`。原因是 Conda 配置有多个层级：

1. 系统级配置，例如 `D:\Anaconda3\.condarc`；
2. 用户级配置，例如 `C:\Users\你的用户名\.condarc`；
3. 环境级配置；
4. `condarc.d` 目录下的额外配置，例如 `anaconda-auth.yml`；
5. 环境变量与命令行参数。

只要系统级配置或 `condarc.d` 还在，`defaults` 就可能继续生效。结果就是：网络请求频繁跑偏到官方源，引发 SSL 证书验证失败：

```text
SSLError: Hostname mismatch
```

**其他做法的坏处：**

我试过只改用户级 `.condarc`，但事实上继续运行show channels命令，依旧会展示defaults通道。因为系统级配置优先级更高，问题没有根治。而直接删除整个conda的安装目录也不行，PATH、注册表等内容残留，很影响后续的重装（=^=）。以及后续如果修改了.condarc文件，需要重启anaconda prompt或者cmd，否则修改内容不会生效。

### 1.3 `PYTHONPATH` 污染

设置了：

```cmd
PYTHONPATH=E:\PythonLibs\modelscope
```

之后，Conda 在启动时加载了该路径下的 `urllib3`，与 Conda 自带版本冲突，引发各种莫名其妙的 SSL 和请求异常。

Python 导入包时，`sys.path` 的顺序非常关键。`PYTHONPATH` 会插入到标准库和 site-packages 之前或之间，导致 Conda 自己运行时加载到外部版本的 `urllib3`、`requests`、`certifi` 等包。Conda 本身也是 Python 程序，它同样依赖这些库。一旦版本不匹配，就可能出现：

- SSL 证书验证失败；
- `requests` 异常；
- `urllib3` 版本冲突；
- Conda 命令时好时坏；
- 某些环境正常，某些环境报错。

**其他做法的坏处：**

不要重复设置环境变量！！！全局设置的 `PYTHONPATH`被无理由地修改或者添加会导致所有 Python、所有 Conda 环境都被污染！不要轻易动环境变量！！！尤其不要把模型库、项目库和下载库全都放进全局路径里！后续路径冲突了完全发现不了！！！


正确做法应该是：不要让全局 `PYTHONPATH` 污染 Conda 环境。如果某个环境确实需要额外路径，使用环境级变量：

```cmd
conda env config vars set PYTHONPATH= -n 环境名
```

或者干脆在项目代码里通过 `sys.path.append` 处理，而不是全局污染。

### 1.4 Anaconda base 环境过于臃肿

Anaconda 的 `base` 环境预装了大量包：Jupyter、NumPy、SciPy、Matplotlib、Pandas、Scikit-learn、Spyder、Qt、各种图像库等。看起来方便，实际带来几个问题：

- 依赖关系复杂，安装新包时解析时间极长；
- 版本冲突概率高；
- 升级某个包可能连带升级 Conda 自身依赖，导致 Conda 崩溃；
- 很多包你根本不用，却参与依赖求解；
- 一旦 base 坏了，修复成本极高。

**其他做法的坏处：**

- 在 base 里直接 `pip install` 所有项目依赖：不同项目互相冲突。
- 用 base 跑 PyTorch/TensorFlow：CUDA、NumPy、Protobuf 版本容易互相打架。
- 把 base 当万能环境：最终变成“什么都有，什么都跑不稳”。

**最终结论：**

Anaconda 的 `base` 环境过于臃肿，且系统级配置与用户级配置互相干扰。为了获得一个干净、可控、轻量的环境，我决定彻底卸载 Anaconda，转投 **Miniconda**。

Miniconda 只包含 Conda、Python 和少量基础依赖。它把“环境管理”交给你，而不是替你做一堆不可控的决定。对于 AI 开发来说，这种可控性远比“预装一大堆包”重要。

---

## 二、Miniconda 安装与“终极”配置

### 2.1 解决没有 Miniconda Prompt 的问题

由于 Miniconda 安装时默认不勾选添加至 PATH，且系统同时存在多个 Python 环境，导致开始菜单没有 Miniconda Prompt。

有些教程会让你把 Conda 加入全局 PATH，但这在 Windows 上非常危险：

- `python` 命令可能被 Conda 的 Python 覆盖；
- 系统其他工具可能调用错误 Python；
- 多版本 Conda、Anaconda、Miniconda 互相抢占；
- 卸载时 PATH 残留，后续命令混乱。

更推荐的做法是：不把 Conda 加入全局 PATH，而是创建一个专用快捷方式。

创建一个新的快捷方式，目标填写（根据实际安装路径调整，注意我的是 `D:\Miniconda3\miniconda`）：

```cmd
cmd.exe /K "D:\Miniconda3\miniconda\Scripts\activate.bat"
```

这样每次打开这个快捷方式，就进入 Miniconda 的 `base` 环境，再通过 `conda activate` 切换环境。它不会污染系统全局 Python。

**其他做法的坏处：**

- 把 Conda 加入全局 PATH：容易和系统 Python、Windows Store Python、其他 Conda 冲突。
- 每次手动输入完整路径激活：效率低，容易忘。
- 直接双击 `python.exe`：绕过 Conda 环境，装包位置错乱。

### 2.2 彻底根除 `defaults` 频道（核心难点）

即使修改了用户级 `.condarc`，`defaults` 依然存在，因为系统级配置（`D:\Miniconda3\miniconda\.condarc`）和 `condarc.d` 目录下的 `anaconda-auth.yml` 在作祟。

Conda 的配置合并规则是：多个配置文件会按优先级合并。你以为删了 `defaults`，但系统级配置里还有；你以为只改了一个文件，但 `condarc.d` 又加回来了。

**终极解决步骤（实在没招了就这么干吧，亲测有效）：**

1. 使用以下命令查看所有生效的配置文件：

```cmd
conda config --show-sources
```

2. **删除或重命名系统级 `.condarc`**：

例如：

```cmd
D:\Miniconda3\miniconda\.condarc -> .condarc.bak
```

同时删除 `condarc.d` 目录。

3. 将用户级 `C:\Users\你的用户名\.condarc` 完全覆盖为以下内容（彻底禁用官方源，转战清华源）：

```yaml
channels:
  - conda-forge
  - pytorch
  - nodefaults
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
show_channel_urls: true
```

这里的关键点：

- `nodefaults`：明确禁用 `defaults`，不再访问官方源。
- `conda-forge`：社区维护，包新且全。
- `pytorch`：PyTorch 相关 Conda 包。
- `default_channels` 和 `custom_channels`：指向清华镜像，提升国内下载速度。
- `show_channel_urls: true`：安装时显示包来自哪个源，便于排查。

4. **重启命令行窗口**，执行：

```cmd
conda config --show channels
```

如果只显示 `conda-forge` 和 `pytorch`，说明彻底成功。

**其他做法的坏处：**

- 只用 `defaults`：国内下载慢，SSL 报错多，repodata 解析失败概率高。
- 只删除用户级 `.condarc` 中的 `defaults`：系统级和 `condarc.d` 可能继续生效。
- 不执行 `--show-sources`：根本不知道哪个配置在起作用。
- 不重启命令行：旧进程仍使用旧配置。
- 多个镜像源混用：元数据可能不一致，依赖解析更容易失败。

### 2.3 验证配置是否生效

建议每次大改配置后都执行：

```cmd
conda config --show-sources
conda config --show channels
conda info
```

确认：

- 没有 `defaults`；
- channel 顺序符合预期；
- `conda info` 中的 `base environment` 和 `channel URLs` 正确；
- 没有意外的系统级配置。

**其他做法的坏处：**

- 不验证就继续建环境：后面报错时根本不知道源头。
- 看到 `defaults` 还在却忽略：后面 SSL、repodata 问题会反复出现。

---

## 三、环境迁移策略：为何不能直接运行 YAML？

最初我从 Anaconda 导出了所有环境的 YAML 文件：

```text
py39.yaml
tire.yaml
labelme.yaml
```

直接尝试：

```cmd
conda env create -f py39.yaml
```

时，遭遇了三大致命错误。

### 3.1 YAML 不是跨平台“镜像”

`conda env export` 会导出：

- 精确到 build 的包版本；
- channel 信息；
- pip 安装的包列表；
- 平台相关依赖。

这意味着 YAML 在另一台机器、另一个系统、另一个 CUDA 环境下，很可能无法复现。尤其是 Windows、Linux、macOS 之间，很多包的 build 完全不同。

**其他做法的坏处：**

- 把 YAML 当 Docker 镜像：以为一键恢复，实际一键报错。
- 不区分平台：`win-64` 的包在 Linux 上找不到。
- 不清理 pip 段：pip 依赖解析可能陷入死锁。

### 3.2 网络中断

YAML 中包含 PyTorch 2.5.1 及 CUDA 库，体积高达 1GB 以上。国内网络下载极易触发：

```text
ConnectionResetError: 10054
IncompleteRead
```

一旦中断，Conda 可能留下半成品缓存，后续重试更慢，甚至出现包损坏。

**其他做法的坏处：**

- 反复全量 `conda env create`：每次都可能重新解析、重新下载，效率极低。
- 不清理缓存：坏包残留，后续安装莫名失败。
- 不用断点续传：大包下载失败后从头再来。

### 3.3 Pip 依赖死锁（`py39` 环境）

YAML 中指定了：

```text
sympy==1.13.1
```

但：

```text
onnxslim==0.1.59 要求 sympy>=1.13.3
```

Pip 在尝试寻找兼容版本时陷入死循环，报错：

```text
ResolutionImpossible
```

这是典型的依赖死锁：

- PyTorch 2.5.1 依赖 `sympy==1.13.1`；
- `onnxslim` 要求 `sympy>=1.13.3`；
- 两者无法同时满足；
- Pip 回溯大量版本后放弃。

**其他做法的坏处：**

- 直接 `pip install -r requirements.txt`：遇到死锁就卡死。
- 盲目升级 `sympy`：可能破坏 PyTorch 依赖。
- 盲目降级 `onnxslim`：可能缺少需要的功能。
- 不锁核心版本：每次安装结果都可能不同。

### 3.4 Conda 依赖死锁（`tire` 环境）

YAML 中：

```text
pandas=2.2.3
```

明确要求：

```text
python>=3.10
```

但环境是：

```text
python=3.9.23
```

直接不兼容。

同时：

```text
mkl-service
mkl_random
```

对 `mkl` 基础库的版本要求截然不同，导致：

```text
LibMambaUnsatisfiableError
```

Conda 的新求解器 LibMamba 虽然快，但遇到硬冲突时依然无法求解。

**其他做法的坏处：**

- 强行让 Conda 求解：可能长时间卡住，最后仍失败。
- 手动降级 `pandas`：可能又和其他包冲突。
- 混用 Conda 和 Pip 的 MKL/NumPy：底层库冲突，运行时报错更隐蔽。

### 3.5 黄金法则

**不要盲目使用全量 YAML 恢复环境！**

策略转换为：

> **Conda 管基础环境（Python），Pip 管复杂包（尤其是 PyTorch 和底层依赖）。分批安装，逐个击破。**

具体原则：

1. Conda 只负责 Python 本体和少量系统库；
2. PyTorch、TensorFlow、ONNX、Ultralytics、Labelme 等用 Pip；
3. 先锁核心依赖，再装外围工具；
4. 遇到冲突工具，用 `--no-deps` 保护主框架；
5. 每装一批就验证一次 `import`。

**其他做法的坏处：**

- 全量 YAML 一键恢复：看似省事，实则把控制权交给解析器。
- 所有包都交给 Conda：复杂框架的 CUDA 版本往往不匹配。
- 所有包都交给 Pip：某些系统级库可能缺少。
- 不记录版本：下次复现困难。

---

## 四、实战案例：从零完美移植 `py39` 环境

以下是 `py39` 环境（Python 3.9 + PyTorch 2.5.1 + CUDA 12.1）从零开始的完整命令与步骤，彻底避开依赖冲突和网络断连。

### 目标环境

- Python 3.9.23
- PyTorch 2.5.1+cu121
- torchvision 0.20.1
- torchaudio 2.5.1
- ONNX 1.17.0
- onnxruntime / onnxruntime-gpu 1.19.2
- Ultralytics 8.0.124
- Labelme 5.8.1
- NumPy 1.26.4
- Pandas 2.3.0
- SciPy 1.13.1
- OpenCV 4.11.0.86
- 其他 CV、标注、科学计算依赖

### 第一步：创建纯 Conda 基础环境

```cmd
conda create -n py39 python=3.9.23 -y
conda activate py39
```

**为什么这样做？**

只让 Conda 创建 Python 本体，不引入 PyTorch、TensorFlow、MKL 等复杂依赖。这样环境最干净，后续 Pip 安装时可控性最高。

**其他做法的坏处：**

- 直接 `conda create -n py39 pytorch ...`：Conda 可能从镜像源拿到非 CUDA 版本，或者依赖解析引入大量 MKL 包。
- 在 base 里直接装：污染 base，后期无法收拾。
- 不指定 Python 小版本：不同小版本可能影响二进制兼容性。

### 第二步：手动安装 PyTorch（必须用官方源）

不要使用清华源安装 PyTorch，因为国内镜像往往没有带有 `+cu121` 后缀的 CUDA 版本 whl 包。

```cmd
pip install torch==2.5.1 torchvision==0.20.1 torchaudio==2.5.1 --index-url https://download.pytorch.org/whl/cu121
```

*提示：下载 1.2GB 的 torch 包时，如中途断连卡死，按 `Ctrl+C` 终止，重新执行此命令，pip 会自动利用缓存断点续传。*

**为什么这样做？**

PyTorch 官方为不同 CUDA 版本提供独立 whl 索引。`--index-url https://download.pytorch.org/whl/cu121` 能确保安装的是 CUDA 12.1 版本，而不是 CPU 版本。

**其他做法的坏处：**

- 用清华源装 PyTorch：可能只有 CPU 版，或者没有 `+cu121` 后缀。
- 用 Conda 装 PyTorch：channel 复杂，容易和 MKL 冲突。
- 不指定版本：可能装到最新版，和 CUDA 驱动不匹配。
- 不检查 `CUDA: True`：等到训练时才发现是 CPU 版。

### 第三步：锁死 `sympy` 版本（解决 Pip 冲突关键）

PyTorch 2.5.1 依赖 `sympy==1.13.1`。必须在安装其他工具前先锁死这个版本：

```cmd
pip install sympy==1.13.1 mpmath==1.3.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**为什么这样做？**

后面安装 `onnxslim` 时，它要求 `sympy>=1.13.3`。如果你先装 `onnxslim`，Pip 可能升级 `sympy`，破坏 PyTorch 的依赖约束。先锁死 `sympy==1.13.1`，再对 `onnxslim` 使用 `--no-deps`，可以保护 PyTorch。

**其他做法的坏处：**

- 不锁 `sympy`：Pip 可能自动升级，导致 PyTorch 依赖警告或运行异常。
- 先装 `onnxslim`：直接触发 `ResolutionImpossible`。
- 强行升级 PyTorch：可能引入新的 CUDA 和依赖问题。

### 第四步：分批安装基础科学计算包（清华源）

```cmd
pip install numpy==1.26.4 pandas==2.3.0 scipy==1.13.1 matplotlib==3.9.4 scikit-learn==1.6.1 scikit-image==0.24.0 seaborn==0.13.2 tqdm==4.67.1 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**为什么这样做？**

NumPy 1.26.4 是很多旧版 CV、标注工具的兼容分水岭。NumPy 2.x 虽然新，但会强制要求 OpenCV、SciPy、Scikit-image 等升级，容易引发连锁冲突。分批安装也便于定位错误。

**其他做法的坏处：**

- 直接装 NumPy 2.x：`opencv-python==4.12` 可能要求 `numpy>=2`，与旧工具冲突。
- 一次装几十个包：Pip 回溯爆炸，耗时长，错误难定位。
- 不指定版本：不同时间安装结果不同。

### 第五步：安装 ONNX 核心包（先不管 `onnxslim`）

```cmd
pip install onnx==1.17.0 onnxruntime==1.19.2 onnxruntime-gpu==1.19.2 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**为什么这样做？**

ONNX 负责模型格式，ONNX Runtime 负责推理。`onnxruntime-gpu` 提供 GPU 推理能力，但依赖 CUDA/cuDNN。先装核心包，确保基础推理可用，再处理 `onnxslim` 这种依赖冲突工具。

**其他做法的坏处：**

- 只装 `onnxruntime`：GPU 不可用。
- 只装 `onnxruntime-gpu`：某些环境可能需要 CPU fallback。
- 版本不匹配 CUDA：运行时报 `cudnn`、`cublas` 错误。

### 第六步：安装 CV 与标注工具链

```cmd
pip install ultralytics==8.0.124 ultralytics-thop==2.0.14 thop==0.1.1-2209072238 -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install labelme==5.8.1 pyqt5==5.15.11 pyqt5-qt5==5.15.2 pyqt5-sip==12.17.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install opencv-python==4.11.0.86 pillow==10.4.0 imageio==2.37.0 imgviz==1.7.6 natsort==8.4.0 loguru==0.7.3 gdown==5.2.0 kaggle==1.7.4.5 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**为什么这样做？**

Ultralytics、Labelme、PyQt5、OpenCV 之间版本敏感。尤其是：

- Labelme 依赖 PyQt5；
- OpenCV 与 NumPy 版本强相关；
- Ultralytics 依赖 THOP、Matplotlib、Pillow 等。

分批安装可以避免一次性解析所有依赖。

**其他做法的坏处：**

- 装 `opencv-python==4.12.0.88`：它可能要求 `numpy>=2`，与 `numpy==1.26.4` 冲突。
- 不锁 PyQt5：可能装到不兼容版本，Labelme 无法启动。
- 混用 `opencv-python` 和 `opencv-contrib-python`：容易冲突。

### 第七步：强制安装 `onnxslim`（化解死锁）

由于 `onnxslim` 要求的 `sympy>=1.13.3` 会破坏 PyTorch，我们使用 `--no-deps` 强行安装它，不升级其依赖：

```cmd
pip install onnxslim==0.1.59 --no-deps -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**为什么这样做？**

`onnxslim` 是一个模型简化工具，它的依赖声明与 PyTorch 的 `sympy==1.13.1` 冲突。使用 `--no-deps` 表示：

- 只安装 `onnxslim` 本身；
- 不安装或升级它声明的依赖；
- 保护已经锁定的 `sympy`、`onnx` 等核心包。

这不是“乱来”，而是在明确知道风险的情况下，优先保护主框架。

**其他做法的坏处：**

- 直接 `pip install onnxslim`：Pip 会尝试升级 `sympy`，可能破坏 PyTorch。
- 放弃 `onnxslim`：后续模型简化、转换流程可能缺失。
- 升级 PyTorch 来适配：可能引入新的 CUDA、TorchVision 兼容问题。

### 第八步：终极验证

```cmd
python -c "import torch; import sympy; import onnx; print('Torch:', torch.__version__); print('CUDA:', torch.cuda.is_available()); print('Sympy:', sympy.__version__); print('ONNX:', onnx.__version__)"
```

**预期输出：**

```text
Torch: 2.5.1+cu121
CUDA: True
Sympy: 1.13.1
ONNX: 1.17.0
```

看到 `CUDA: True`，标志着这个最复杂的环境移植彻底成功。

**其他做法的坏处：**

- 不验证：等到训练时才发现 CUDA 不可用。
- 只看 `pip list`：版本对不代表能 `import`。
- 不检查 `sympy`：可能已被 `onnxslim` 升级，留下隐患。

---

## 五、其他环境避坑指南（Tire & Labelme）

### 5.1 Tire 环境（多框架共存：PyTorch + TensorFlow）

Tire 环境需要同时支持 PyTorch 和 TensorFlow，这是最容易出现依赖死锁的场景之一。

#### 错误 1：Conda 安装 MKL 体系导致 `LibMambaUnsatisfiableError`

执行：

```cmd
conda install mkl...
```

时遭遇：

```text
LibMambaUnsatisfiableError
```

因为 `mkl-service` 与 `mkl_random` 互相锁死版本，而它们又依赖不同版本的 `mkl` 基础库。

**解决：**

**放弃 Conda 安装 MKL 体系**，直接用：

```cmd
pip install numpy pandas matplotlib scipy
```

Pip 的 NumPy 自带独立的 MKL/OpenBLAS 库，完美避开 Conda 的 MKL 死锁。

**其他做法的坏处：**

- 强行 Conda 安装 MKL：求解器可能长时间卡住，最后仍失败。
- 手动降级 `mkl`：可能导致 NumPy、SciPy 运行异常。
- 混用 Conda 和 Pip 的 NumPy：底层 BLAS 库冲突，运行时报错隐蔽。

#### 错误 2：`pip install tensorflow-base` 找不到包

报错：

```text
No matching distribution found
```

原因是 `tensorflow-base` 是 Conda 特有的包名，Pip 源里只有 `tensorflow`。

**解决：**

```cmd
pip install tensorflow==2.18.1 keras==3.6.0
```

**其他做法的坏处：**

- 继续搜索 `tensorflow-base`：Pip 永远找不到。
- 装 `tf-nightly`：版本不稳定，依赖更复杂。
- 不锁 Keras 版本：TensorFlow 2.18 与 Keras 3.x 有兼容要求。

#### 错误 3：Protobuf 终极死锁

- TF 2.18.1 要求 `protobuf>=4.21.0`；
- `tf2onnx 1.16.1` 要求 `protobuf<=3.20.2`；
- 同时 `onnxslim` 与 `sympy 1.13.1` 冲突。

**解决：**

1. 强行装新版 protobuf：

```cmd
pip install "protobuf>=4.21.0,<6.0.0" onnx==1.17.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

2. 无视依赖，强行装 `tf2onnx`：

```cmd
pip install tf2onnx==1.16.1 --no-deps -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**结果：**

验证成功输出：

```text
Torch: 2.5.1+cu121
TF: 2.18.1
CUDA: True
ONNX: 1.17.0
```

**其他做法的坏处：**

- 强行降级 protobuf 到 3.20：TensorFlow 2.18 可能无法运行。
- 强行升级 tf2onnx：可能引入不兼容的 ONNX 依赖。
- 不用 `--no-deps`：Pip 会继续陷入死锁。

### 5.2 Labelme 环境（轻量图像标注）

Labelme 本身不复杂，但它对 PyQt5、OpenCV、NumPy 的版本很敏感。

#### 错误：`opencv-python==4.12.0.88` 强制要求 `numpy>=2`

与锁定的：

```text
numpy==1.26.4
```

冲突，导致：

```text
ResolutionImpossible
```

**解决：**

**降低 `opencv-python` 版本以适配 `numpy`**。执行：

```cmd
pip install numpy==1.26.4 pillow==11.3.0 labelme==5.2.1 pyqt5==5.15.11 pyqt5-qt5==5.15.2 pyqt5-sip==12.17.0 opencv-python==4.10.0.84 scikit-image==0.24.0 scikit-learn==1.6.1 scipy==1.13.1 matplotlib==3.9.4 imgviz==1.7.5 natsort==8.4.0 loguru==0.7.3 gdown==5.2.0 osam==0.2.4 onnxruntime==1.19.2 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**其他做法的坏处：**

- 升级 NumPy 到 2.x：可能破坏其他依赖旧 NumPy 的工具。
- 强行装最新 OpenCV：与 NumPy 1.26.4 冲突。
- 不锁 PyQt5：Labelme 可能无法启动或界面异常。

---

## 六、总结与避坑宝典

（图方便直接让蓝色大肥鱼总结了……请不要期待一只脆脆鲨的总结能力）：

1. **精简 Conda，禁用 `defaults`**  
   遇到 `defaults` 频道报错或 SSL 错误，请检查系统级和用户级 `.condarc`，用 `nodefaults` 和清华源彻底替换，不留死角。  
   **其他做法的坏处**：只改一个文件、不查 `--show-sources`，问题会反复出现。

2. **隔离 `PYTHONPATH`**  
   不要让全局的 `PYTHONPATH` 污染 Conda 环境。如果在特定环境需要 `modelscope`，请使用：
   ```cmd
   conda env config vars set PYTHONPATH= -n 环境名
   ```
   单独处理。  
   **其他做法的坏处**：全局污染会导致 Conda 自身加载错误版本的 `urllib3`、`requests`，引发 SSL 和依赖异常。

3. **拒绝全量 YAML**  
   复现环境时，优先使用 `conda create` 建 Python，然后用 `pip` 分批装库。这样可以完全掌控依赖解析过程。  
   **其他做法的坏处**：全量 YAML 容易遇到平台不兼容、网络中断、依赖死锁，排错成本极高。

4. **区分 Pip 与 Conda 的安装策略**  
   - **Conda 负责**：Python 本体，以及简单的系统库。  
   - **Pip 负责**：PyTorch、TensorFlow、ONNX 等高阶框架（使用官方源或国内镜像源）。  
   **其他做法的坏处**：把复杂框架交给 Conda，容易遇到 CUDA 版本不对、MKL 冲突；把系统库全交给 Pip，可能缺少底层支持。

5. **遇到依赖死锁的解药**  
   谁报错，就单独把它拎出来。如果确实需要安装一个会破坏核心依赖（如 `sympy` 或 `protobuf`）的工具（如 `onnxslim` 或 `tf2onnx`），**使用 `pip install 包名 --no-deps` 强行安装，忽略其依赖**，以保护主框架的完整。  
   **其他做法的坏处**：盲目让 Pip 求解，可能升级或降级核心包，导致 PyTorch、TensorFlow 无法运行。

6. **不要迷信依赖解析器的报错**  
   只要核心框架（Torch/TF）在 `import` 后能跑通，`pip` 红字警告和 `oneDNN` 提示完全可以无视。  
   **其他做法的坏处**：为了消除所有警告而反复调整版本，可能把原本稳定的环境搞崩。

2026.9.23 stone
CUDA: True
```