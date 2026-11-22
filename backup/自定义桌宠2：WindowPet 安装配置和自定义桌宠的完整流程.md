上一篇把天使的立绘和精灵图做出来了（[绘世启动器使用教程和精灵图预处理](绘世启动器使用教程和精灵图预处理（适用于WindowPet等桌宠工具）.md)），这一篇讲怎么把它装进 WindowPet：下载安装、开机自启、注册自定义宠物、以及我调试时踩到的那个**高 DPI 大坑**。

先看结果——这是跑在桌面上的小天使（桌宠是独立的透明置顶窗口，就悬在文件窗口右下角）：

![桌宠实际运行效果：小天使停在桌面右下角，浮在文件窗口之上]

<img width="1280" height="764" alt="15-desktop-pet-full" src="https://github.com/user-attachments/assets/da76af7c-5e78-4182-a2bf-3544835b3c3c" />

整屏看是这样的（左边是我的大肥鱼，这里只看右边的小天使；整层宠物窗口都不挡鼠标点击）：

![整屏效果：透明悬浮的桌宠层叠在文件窗口之上]

<img width="2560" height="1552" alt="09-windowpet-folder" src="https://github.com/user-attachments/assets/9039a2ba-064a-4dfc-8c81-af675838d352" />

---

## 一、选型：为什么是 WindowPet

我需要同时满足三个条件：**开机自启**、**能自定义形象**、**开源**。

WindowPet 是 `SeakMengs/WindowPet`（MIT 协议），Tauri + React + Phaser 写的桌面挂件，截图时 676 stars。它支持：

- 内置 45+ 个宠物/角色（都是 shimeji 风格的精灵图）；
- **添加自己的宠物**（就是上一篇做的精灵图 + 帧配置）；
- 点击穿透（不挡鼠标）、拖拽、翻转、攀爬、置顶于任务栏；
- **开机自启**、自动更新、系统托盘、中英文界面。

它的定位更像"精灵图播放器"，正好接住我们上一环节的产物。装好之后「关于」页能看到版本号，也支持一键检查更新：

![WindowPet「关于」：版本 0.0.9、检查更新、源码与反馈入口]

<img width="1280" height="747" alt="12-windowpet-update" src="https://github.com/user-attachments/assets/044c3ee1-cb44-4cc1-b886-0742201eec7f" />

> 另外两类的区别：Live2D 类要绑定骨骼（门槛高），3D 类（VRM 引擎）要建模。我们这种"AI 生图 + 横排精灵图"的方案，最适合没有美术/建模基础的人。

---

## 二、下载与安装

### 2.1 下载

Release 页：`https://github.com/SeakMengs/WindowPet/releases`，最新版是 **v0.0.9**，Windows 提供 `.exe`（NSIS 安装器）和 `.msi` 两种：

| 资产 | 大小 |
| --- | --- |
| `WindowPet_0.0.9_x64-setup.exe` | 16.84 MB（17,660,165 字节） |
| `WindowPet_0.0.9_x64_en-US.msi` | 17.68 MB |

我这台机器 `github.com` 直连不通、只有 `api.github.com` 能通，所以是走 GitHub API 的 release asset 接口下载的（浏览器能打开 GitHub 的话直接点就行）：

```bash
# 取 asset id
curl -s https://api.github.com/repos/SeakMengs/WindowPet/releases/latest
# 按 asset id 下载（-L 跟随 302 到 objects.githubusercontent.com）
curl -L -H "Accept: application/octet-stream" \
  -o WindowPet_0.0.9_x64-setup.exe \
  https://api.github.com/repos/SeakMengs/WindowPet/releases/assets/245462097
```

下载完**先校验大小和哈希**（这个安装包没有数字签名，大小/哈希对得上才敢装）：

```text
大小：17,660,165 字节   （与 GitHub 资产完全一致）
SHA256：C8E432E2066D02F84CA7BCC925877E13D2CB090F5C47F6E980095AA609472EDC
签名：NotSigned（开源小项目常见，介意的话自己拉源码编译）
```

### 2.2 安装

NSIS 安装器支持静默参数，`/D=` 可以指定安装目录（**必须是最后一个参数，且不能带引号**）：

```bat
WindowPet_0.0.9_x64-setup.exe /S /D=F:\WindowPet\app
```

装完目录很干净，只有一个主程序和卸载器：

```
F:\WindowPet\
├─ app\
│  ├─ WindowPet.exe
│  └─ uninstall.exe
├─ downloads\          ← 安装包备份
└─ 天使形象制作\        ← 上一篇的素材/成品/脚本
```

![安装目录：app 是程序本体，downloads 是安装包备份，天使形象制作是素材与成品]

<img width="2560" height="1552" alt="09-windowpet-folder" src="https://github.com/user-attachments/assets/ead2ad70-b642-47e0-8c6a-87a8f86ec404" />

同时会自动创建开始菜单快捷方式，并在 `HKCU\...\Uninstall` 里登记卸载项，所以「设置 → 应用」里也能正常卸载。

> 它是**当前用户安装**（per-user），不需要管理员权限，装到 `F:\...` 这种非系统盘也没问题。

---

## 三、开机自启

WindowPet 自带「开机自启」开关（设置面板 → 设置 → **自动启动**）：

![WindowPet「设置偏好」：自动启动 / 宠物在任务栏上方 / 宠物互动 / 允许宠物攀爬 / 覆盖宠物缩放 / 语言]

<img width="1280" height="747" alt="12-windowpet-settings" src="https://github.com/user-attachments/assets/76076411-98c0-46a1-b93f-15690b795a6f" />

它底层是用 `tauri-plugin-autostart` 写注册表：

```text
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
  值名：WindowPet
  值：  "F:\WindowPet\app\WindowPet.exe" --flag1 --flag2
```

同一个软件还会顺手写 `StartupApproved\Run`（告诉任务管理器这个启动项是启用的）。

所以想手动开/关，有两条路：

1. 图形界面：托盘图标 → **设置** → 通用 → **开机自启** 开关；
2. 命令行：

```bat
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v WindowPet ^
  /t REG_SZ /d "\"F:\WindowPet\app\WindowPet.exe\" --flag1 --flag2" /f
```

验证方法：`Win+R` 输入 `shell:startup` 看启动文件夹里没有它（它是注册表方式），或者任务管理器 → 启动应用 → 看有没有 WindowPet 且状态为"已启用"。

---

## 四、把精灵图注册成自定义宠物

WindowPet 的配置都在 `%APPDATA%\WindowPet\`：

```
%APPDATA%\WindowPet\
├─ settings.json      ← 全局设置（语言、缩放、是否置顶等）
├─ pets.json          ← 「我的宠物」列表（当前桌面上有的）
├─ pet_linker.json    ← 自定义宠物配置文件清单
├─ custom-pets\       ← 每个自定义宠物一个 json
│  └─ angel_pet.json
└─ assets\            ← 自定义宠物的精灵图
   └─ angel_pet.png
```

有两条路：**用界面添加**，或者**直接写配置文件**。

### 4.1 方式 A：在设置面板里添加

1. **双击托盘图标**（或右键 → 设置）打开设置面板，切到「**添加自定义宠物**」；
2. 表单四项：**宠物名称**、**帧大小**、**精灵图路径**、**宠物状态**（可点「添加更多状态」继续加动作），填完点右下角「添加自定义宠物」：

![「添加自定义宠物」表单：宠物名称 / 帧大小 / 精灵图路径 / 宠物状态]

<img width="1280" height="747" alt="12c-windowpet-pets" src="https://github.com/user-attachments/assets/97e35324-e09c-4eec-aab3-8cbf97db44b1" />

3. 保存后**重启 WindowPet**，再去「**宠物商店**」里找到它、点「获取」添加，桌面上才会出现：

![「宠物商店」：内置 50 个角色，每个都能选动作预览并一键获取]

<img width="1280" height="747" alt="12d-windowpet-shop" src="https://github.com/user-attachments/assets/96cece9b-eb9f-4fd6-aee2-9442107aebad" />

回到「**我的宠物**」，就能看到自己的宠物卡片了——可以选播放哪个动作状态（图里选的是 `climb`），也可以点「移除」：

![WindowPet「我的宠物」面板：小天使 + 动作下拉框 + 移除按钮]

<img width="1280" height="747" alt="12e-windowpet-mypet" src="https://github.com/user-attachments/assets/111edced-0a83-4ce5-b7d0-f3d5fb80ed3d" />

### 4.2 方式 B：直接写配置（可脚本化）

自定义宠物的 JSON 结构如下（可以直接拿来改）：

```json
{
  "frameSize": 256,
  "imageSrc": "C:\\Users\\<你>\\AppData\\Roaming\\WindowPet\\assets\\angel_pet.png",
  "name": "小天使",
  "states": {
    "idle":  { "start": 5, "end": 5 },
    "stand": { "start": 5, "end": 5 },
    "walk":  { "start": 1, "end": 4 },
    "sit":   { "start": 5, "end": 5 },
    "jump":  { "start": 2, "end": 2 },
    "fall":  { "start": 2, "end": 2 },
    "drag":  { "start": 5, "end": 5 },
    "crawl": { "start": 5, "end": 5 },
    "climb": { "start": 5, "end": 5 },
    "bounce":{ "start": 2, "end": 2 }
  }
}
```

![配置文件：angel_pet.json 里的 frameSize / imageSrc / states（右上角就是桌面上运行的小天使）]

<img width="2576" height="1568" alt="10-windowpet-config" src="https://github.com/user-attachments/assets/6749d253-c61f-4681-a762-9ade019e7c1c" />

字段含义：

| 字段 | 说明 |
| --- | --- |
| `frameSize` | 每帧正方形边长（像素），必须和精灵图实际切片一致 |
| `imageSrc` | 精灵图绝对路径 |
| `name` | 显示名，也是程序内部的纹理 key（可以中文） |
| `states` | 动作名 → 帧范围（`start`/`end` 都是 **1 起始**的帧号） |

然后还要两件事，程序才知道有这个宠物：

1. 把它追加进 `pet_linker.json`（自定义宠物清单）：

```json
["C:\\Users\\<你>\\AppData\\Roaming\\WindowPet\\custom-pets\\angel_pet.json"]
```

2. 把它放进 `pets.json` 的 `app` 数组，并给一个 `id` 和 `type: "custom"`——这一步等价于在「宠物商店」里点「添加」：

```json
{ "app": [ { ...宠物对象..., "id": "a1b2c3d4-0000-0000-0000-000000000001", "type": "custom" } ] }
```

改完 `settings.json`/`pets.json` 都要**重启程序**（这些文件是启动时读的）。

### 4.3 动作状态怎么配

WindowPet 的场景里会用到这些状态：

| 状态 | 什么时候播 | 我的配置 |
| --- | --- | --- |
| `walk` | 来回走动 | 1~4（4 帧循环） |
| `idle` / `stand` | 站着不动 | 5（站立帧） |
| `jump` / `fall` / `bounce` | 跳/落 | 2（抬脚、翅膀上扬的帧，看起来像腾空） |
| `drag` | 被鼠标拖动 | 5 |
| `sit` / `crawl` / `climb` | 坐/爬/攀爬 | 5（没有专门帧就复用站立） |

> 注意：**缺失某个状态不会崩**，但如果你只配了 `idle`，场景切到 `fall`/`climb` 时动画会播不出来。保险做法是十个状态全写上，指向已有的帧。

---

## 五、调试实录：高 DPI 下桌宠"消失"了

这是这次最花时间的问题，值得完整写一遍。

### 5.1 现象

装好、配好宠物、程序也在运行（托盘有图标、进程活着、日志显示 `Pets scene loaded`），**但屏幕上什么都没有**。

一开始以为是形象包的问题，换了内置宠物——一样看不见。于是做了个对照实验：把宠物换成一张**纯品红色块**（256×256 全填充），然后调 `petScale`：

| `petScale` | 结果 |
| --- | --- |
| 0.7（默认） | 屏幕上找不到任何品红像素 |
| 2.0 | 还是没有 |
| 6.67 | 屏幕右侧边缘露出一条品红 |
| 10 | 整屏品红 |

结论：**渲染是正常的，但宠物被画在了屏幕可视区域之外**。

再往下量：我的屏幕是 2560×1600、系统缩放 **150%**（所以 PowerShell 里 `SystemInformation.VirtualScreen` 只报 1707×1067）。WindowPet 用 `window.screen.width/height` 决定"舞台"大小，拿到的是 **2560×1600**，而窗口实际可见区域只有 **1707×1067**——舞台比可见区大了 1.5 倍。

宠物在地面（舞台 y≈1600）走动，而屏幕上只能看到 y<1067，所以全都落在屏幕下方；只有爬墙的宠物偶尔会从边缘露出来一点。

### 5.2 试过但没用的办法

| 尝试 | 结果 |
| --- | --- |
| 兼容性 → 替代高 DPI 缩放行为（DPI 不感知） | 无效（进程 DPI 感知等级确实变成 0 了，仍不对） |
| 兼容性 → 系统（增强）缩放 | 无效 |
| 外部调用 `SetWindowPos` 触发 resize 事件 | 无效（Phaser 的舞台尺寸来自 `screen`，不是窗口尺寸） |

因为程序把 `window.screen` 写死在初始化里，属于程序自身的 DPI 适配 bug，不动源码改不了。

### 5.3 解决办法

把系统缩放临时改成 **100%**（设置 → 系统 → 显示 → 缩放），此时窗口可见区域 = 2560×1600 = 舞台尺寸，宠物立刻正常出现在屏幕底部。

代价是屏幕上其他 UI 变小；如果只是偶尔用，改回来再改过去就行（我最后就是保持在 100% 用的）。

> 如果你也遇到"桌宠程序在跑但看不见"，先做三件事：
> ① 托盘右键 → **显示**（Show）；
> ② 看「我的宠物」是不是空的（空的话程序会自动关掉桌宠层并弹提示）；
> ③ 用一个高对比度的测试图 + 放大 `petScale` 验证是不是被画到屏幕外了。

---

## 六、动画调优：从"左右晃"到"真走路"

装上之后第一版走路很难看，这里记录两次迭代，给后来人避坑。

### 第一版：整图位移

只有 5 帧，做法是把整张立绘按不同角度旋转 + 上下平移——本质是"晃"，脚没动，像贴纸在抖。

### 第二版：局部重绘腿

用 SD 的局部重绘（inpaint）只重画腿部，帧序是「分开 → 并拢 → 分开 → 并拢」。比第一版好，但两个问题：

- 过渡帧和接触帧几乎一样（脚的位置没变），播放起来像只有 2 帧；
- 其中一帧是用"镜像腿部"做的换脚，左右鞋子被翻了个面，走路方向感不对。

### 第三版：参考内置甘雨的 walk

拆了内置甘雨的精灵图（`Ganyu-xll.png`，128 一格，walk 在第 2 行 4 帧），量出它的节奏：脚部横向摆动幅度约帧宽的 27%，帧序「接触–过渡–接触–过渡」，身体几乎不转。

然后照这个节奏重新生成：

1. 三个关键腿型：左脚前 / 右脚前 / 并拢（**不镜像**，左右脚用不同提示词分别生成，鞋尖朝向才对）；
2. 两个过渡帧：一只脚离地向前摆（"passing pose"）；
3. 合成时**用 alpha 合成**（不是 RGB 贴图）+ 接触帧把腿的下半段向两侧撇开 13px（512 坐标）+ 接触帧身体低 1px / 过渡帧高 2px；
4. 状态：`walk: 1-4`，其余用第 5 帧站立、第 2 帧腾空。

参考帧和最终 5 帧：

![内置甘雨 walk 的 4 帧参考]

<img width="1596" height="454" alt="07-ganyu-walk-ref" src="https://github.com/user-attachments/assets/9d0fad58-1861-472b-ba3c-d2fe5c3e0860" />

![小天使最终 5 帧：接触 → 过渡 → 接触 → 过渡 → 站立]

<img width="1360" height="386" alt="05-sheet-spec" src="https://github.com/user-attachments/assets/539da034-5cbe-4bef-b837-5bf83ec9f064" />

![最终精灵图 1280×256]

<img width="1280" height="256" alt="08-angel-sheet" src="https://github.com/user-attachments/assets/ed39dd43-46e5-4d3e-abab-6bc738658efe" />

播放效果（9fps，4 帧循环）：

![走路循环 GIF]

<img width="256" height="256" alt="15-walk-loop" src="https://github.com/user-attachments/assets/45512416-5098-4c7b-93ad-66b603961ea9" />

---

## 七、备选形象：粉发白翼版

同一套流程可以批量产不同形象：我另外跑了一版 **粉发 + 白色大翅膀 + 光环 + 无帽无眼镜** 的天使，只做素材备货，没有装进 WindowPet（桌面上始终只保留戴帽版）。

![粉发白翼 + 光环的备选天使立绘]

<img width="512" height="768" alt="13-pink-angel" src="https://github.com/user-attachments/assets/24888ad9-383a-4f90-a4bf-df77de7f51ee" />

![粉发版的透明立绘 + 原始立绘对照]

<img width="572" height="424" alt="14-pink-preview" src="https://github.com/user-attachments/assets/3e00afd4-f716-4ce8-916e-c7234692bc8a" />

想换的时候：把它的精灵图放进 `assets\`，改 `custom-pets\*.json` 的 `imageSrc` 和 `states`，重启即可；或者在「宠物商店」里再添加一只。

---

## 八、清理与卸载

- **卸载**：运行 `F:\WindowPet\app\uninstall.exe`（或设置 → 应用 → WindowPet）；
- **配置**不会被自动删：删掉 `%APPDATA%\WindowPet\` 才算彻底清理（里面有自定义宠物、设置、日志）；
- **关闭开机自启**：托盘 → 设置 → 关掉开关，或删掉注册表 `HKCU\...\Run` 里的 `WindowPet` 值；
- **日志**：`%APPDATA%\WindowPet\WindowPet.log`，排查启动问题时先看它（`app started`、`Pets scene loaded`、报错都在这里）。

---

## 九、完整流程回顾

```text
照片 → [SD 生图: 绘世/WebUI] → 原始立绘
     → [抠图: 逐行估背景 + 连通域] → 透明立绘 256×256
     → [局部重绘 + 合成] → frames/01..05.png
     → [拼图脚本] → 精灵图 1280×256 + angel_pet.json
     → [WindowPet] → assets/angel_pet.png + custom-pets/angel_pet.json + pet_linker.json + pets.json
     → [重启程序] → 桌面上的小天使 🎉
```

几个关键参数备忘：

| 项目 | 值 |
| --- | --- |
| 精灵图 | 1280×256，5 帧 × 256×256，RGBA |
| 帧尺寸 `frameSize` | 256 |
| 动作 | `walk: 1-4`，`idle/stand/sit/drag/crawl/climb: 5`，`jump/fall/bounce: 2` |
| 显示缩放 | **系统 100%**（150% 下会跑到屏幕外） |
| 播放帧率 | 程序内置 9fps（约 0.44 秒一个走路循环） |

整条链路里最容易翻车的其实不是工具，而是**一致性**和**环境**：形象要一致（固定 seed、只改局部）、动作要一致（参考真实样本的帧序和幅度）、环境要一致（高 DPI 缩放、模型/tokenizer 的下载）。把这三件事按坐标系量清楚，剩下的都是体力活。
