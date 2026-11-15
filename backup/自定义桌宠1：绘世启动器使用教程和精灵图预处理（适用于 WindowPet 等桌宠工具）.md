这个教程会有至少两篇，内容主要是围绕如何自定义一个独属于自己、独一无二的桌宠。从形象设计、图片生成、多帧率精灵图制作到开源桌宠工具、WindowPet自定义配置全覆盖，或许还会有模型增强、高帧率精灵图制作教学等内容（画大饼）。当然，为了省事，博客内容也使用AI进行辅助编写。

我答应了要做一个独一无二的桌宠，一个天使的桌宠，这样每天都能感觉她在陪我（嘻嘻）。

从3D建模到live2D，折腾了一圈之后发现：**主流图片类桌宠（比如 WindowPet）根本不需要这些，它要的只是一张「精灵图」**，负责把若干张动作帧横向排开的一张 PNG图片就够了。

所以制作过程就变成三步了：

1. 用 **绘世启动器**（A绘世整合包）跑 Stable Diffusion，生成一张 Q 版天使立绘；
2. 抠成透明 PNG，再拆成/补成一组动作帧；
3. 拼成一张精灵图，交给桌宠程序。

这篇讲第 1、2 步，以及精灵图的规格与预处理细节。桌宠那边的安装配置在另一篇。

![绘世启动器主界面：左侧是分类入口，中间是文件夹快捷入口，右下角是「一键启动」，底部显示启动器与 WebUI 版本]

<img width="768" height="525" alt="01-huishi-launcher" src="https://github.com/user-attachments/assets/9f206b55-c1dd-4b79-bbfb-7325f885ef57" />

---

## 一、先搞清楚桌宠要的是什么

WindowPet 这类工具读取的是「精灵图（sprite sheet）+ 动作帧范围」：

- **精灵图**：一张 PNG，横向排列 N 个小方格，每格一帧，每格是正方形（本例 256×256）；
- **动作帧范围**：告诉程序「idle 用第 1~2 帧、walk 用第 3~4 帧、jump 用第 5 帧」。

也就是说，你要给的不是一段视频、不是 3D 模型，而是**同一角色的若干张姿态图**。这正好是 AI 生图 + 简单图像处理能覆盖的范围。

所以整条链路是：**文生图 → 选图 → 抠图 → 拆帧/补帧 → 拼图**。

---

## 二、绘世启动器：它到底替你做了什么

绘世启动器（A绘世，秋叶整合包的图形壳）本质上是个「Stable Diffusion WebUI 启动器 + 环境管理器」，对新手最有用的是三件事：

1. **一键启动**：不用自己配 Python、torch、CUDA，点一下就把 WebUI 跑起来；
2. **文件夹直达**：根目录、扩展（extensions）、模型、文生图输出目录都能一键打开，省得自己翻路径；
3. **高级选项 / 疑难解答 / 版本管理 / 模型管理**：卡住的时候能从这里看启动参数、换版本、下模型。

界面见开头的截图：左侧是一键启动按钮和分类，右侧是「文件夹」快捷入口（根目录、扩展文件夹、临时文件夹、文生图(网格)、文生图(单图)、图生图……）。启动之后浏览器会自动打开 `http://127.0.0.1:7860`，这就是 WebUI 的界面。

> 小提示：整合包目录里常见的坑是模型放错位置。SD1.5 模型放 `models\Stable-diffusion\`，VAE 放 `models\VAE\`，LoRA 放 `models\Lora\`。绘世启动器的「模型管理」页面就是干这个的。

---

## 三、第一次启动会踩的坑：CLIP tokenizer 拉不下来

我这次是自己用命令行起的 API 服务而不是点「一键启动」，结果模型加载直接失败，日志里反复刷：

```
OSError: Can't load tokenizer for 'openai/clip-vit-large-patch14'.
...
creating model quickly: OSError
loading stable diffusion model: OSError
```

看日志其实是两件事叠加：

1. WebUI 需要 HuggingFace 上的 `openai/clip-vit-large-patch14` tokenizer；
2. 这台机器 `huggingface.co` 是连不上的（`curl` 直接超时），于是它不断重试、进度条卡在 0%，ETA 显示 4800 多秒。

**解法**：把 HuggingFace 端点指到镜像站再启动，tokenizer 会自动下载到本地缓存，之后就不用联网了。

```bat
set HF_ENDPOINT=https://hf-mirror.com
set HF_HUB_DISABLE_TELEMETRY=1
python\python.exe launch.py --api --nowebui --xformers --port 7860 --skip-version-check --no-hashing
```

加上这两个环境变量后，启动时间从「卡死」变成 94 秒左右，模型 16 秒加载完，日志里能看到：

```
Model loaded in 16.3s (load weights from disk: 0.1s, create model: 11.8s,
apply weights to model: 3.2s, load textual inversion embeddings: 0.2s, ...)
Startup time: 94.5s
```

顺带一提：`--nowebui` 只开 API 不开界面，适合脚本化；想用图形界面就把这个参数去掉，用绘世启动器的「一键启动」也行。

---

## 四、在 WebUI 里生成天使立绘

界面如下（中文界面，深色主题，模型已经加载成 `anything-v5`）：

![WebUI 文生图界面：左上选模型（anything-v5），中间是正向/反向提示词——图里就是这次用的那组，提示词下方是自动解析出的词条标签]

<img width="1570" height="505" alt="02-webui-txt2img" src="https://github.com/user-attachments/assets/ef855f83-aad2-4e13-b153-fa638b44539a" />

我用的参数（这组参数对 SD1.5 动漫模型比较稳）：

| 参数 | 值 |
| --- | --- |
| 模型 | `anything-v5`（SD1.5，二次元向） |
| 采样器 | `DPM++ 2M Karras` |
| 迭代步数 | 28~30 |
| CFG Scale | 7 |
| 尺寸 | 512×768（竖构图，留出全身） |
| 批量 | 4（一次出 4 张挑一张） |
| 种子 | -1（随机） |

**正向提示词**（描述角色 + 画风 + 背景，方便后面抠图）：

```
masterpiece, best quality, 1girl, solo, chibi, full body, standing, front view,
angel, white feathered wings, glowing halo, long brown hair, black round glasses,
pink baseball cap, white dress, cute smile,
pure white background, simple background, no shadow, cel shading, clean lineart, pastel colors
```

**反向提示词**：

```
(worst quality, low quality, normal quality:1.4), EasyNegative, badhandv4,
(bad hands, bad anatomy:1.2), extra digits, multiple views, multiple girls,
text, watermark, signature, blurry, jpeg artifacts, realistic, photo, 3d,
cropped, out of frame, grey background, gradient background, shadow, floor, scenery, dark background
```

几个写提示词的经验：

- **一定要写 `pure white background` + `no shadow`**：纯色背景后面抠图几乎零成本，有渐变/阴影就要上更复杂的算法；
- **配饰要写死**：`black round glasses`、`pink baseball cap`，不然模型会自由发挥，几批图之间角色对不上；
- **`chibi` + `full body` + `front view`** 三件套决定构图，缺了 `full body` 容易出大头像，缺了 `front view` 会出侧身（桌宠左右走时会翻面，正面最稳）；
- 负向里加 `grey background / shadow / floor` 能明显提高背景干净度。

出一批 4 张，挑一张最像的：

![挑中的原始立绘：粉白棒球帽 + 黑框眼镜 + 棕发 + 白裙 + 小翅膀]

<img width="512" height="768" alt="03-angel-raw" src="https://github.com/user-attachments/assets/bf354c14-484d-47f2-93b5-19751eed31c8" />

---

## 五、抠图：白底 → 透明 PNG

这一步是关键，**精灵图必须是带透明通道的 RGBA PNG**，否则桌宠周围会带一块白底/黑块。

手工做法（GIMP，和大部分人习惯一致）：

1. GIMP 打开图片 → 菜单「图层 → 透明 → 添加 Alpha 通道」；
2. 用「按颜色选择」（Select by Color）点背景，容差 30~40；
3. 反选，或者直接 `编辑 → 清除`；
4. 边缘用「选择 → 羽化 1px」再清除一次，避免锯齿；
5. 导出 PNG（记得勾选「保存色彩空间/透明」）。

我实际用脚本做（可复现、能批处理），思路比"全图按颜色选中白色"稳一点：
**逐行取图片左右边缘 12 列的中位色当作该行背景色**（这样即使背景是渐变也能估准），
再只把「与画布边框连通」的候选背景删掉——这样角色身上的白裙子、白翅膀不会被误伤。

```python
# 简化版核心逻辑
edge = np.concatenate([rgb[:, :12, :], rgb[:, -12:, :]], axis=1)
bg_row = np.median(edge, axis=1)                  # 每行的背景色
diff = np.abs(rgb - bg_row[:, None, :]).max(axis=2)
cand = (diff < tol).astype(np.uint8)              # 候选背景
n, lab = cv2.connectedComponents(cand, 8)         # 只保留和边框连通的
bg = np.isin(lab, border_labels).astype(np.uint8)
alpha = (1 - bg) * 255
```

抠完是这样的（棋盘格背景是为了显示透明区）：

![抠图结果：透明背景的 256×256 立绘]

<img width="256" height="256" alt="04-angel-cutout" src="https://github.com/user-attachments/assets/d39fc6a8-4949-4c24-885e-c3dbd352ca1a" />

> **两个细节**：① 边缘会有 1px 的半透明白边，做精灵图时可以用「白底去除」（`C' = (C - (1-a)·255)/a`）再清一遍；② 阴影别留——桌宠是贴在桌面上的，自带阴影会显得脏。

---

## 六、从一张图到 5 帧动作

一张静态图也能当桌宠，但走起来就是"平移"。要做走路，就得有**同一角色的多个姿态帧**。

### 6.1 动作参考：拆开内置甘雨的 walk

WindowPet 内置的甘雨就是个标准样本。它的精灵图是 1664×1152、128 一格，walk 是第 2 行的 4 帧：

![内置甘雨 walk 的 4 帧：接触 → 过渡 → 接触 → 过渡]

<img width="1596" height="454" alt="07-ganyu-walk-ref" src="https://github.com/user-attachments/assets/3d2931ae-4358-4a12-84bc-322499545235" />

我顺手量了一下每帧脚的位置（脚部像素的 x 范围）：`51~86 → 58~83 → 54~85 → 51~86`，也就是说帧宽 128 的图里，**两只脚横向摆动的幅度大约是帧宽的 27%**；帧序是「两脚分开 → 并拢(一只脚在前) → 分开 → 最开」，即标准的 **接触–过渡–接触–过渡**。身体几乎不旋转，行走感来自腿的前后交替 + 轻微上下起伏。

### 6.2 保持形象不变的做法：只做局部重绘

整图重画一定会变成"另一个人"。所以我把**腿部**和**翅膀**各圈一个 mask，用 SD 的 **inpaint（局部重绘）** 只改这两个区域，脸、帽子、眼镜、头发、裙子一个像素都不动。

关键是把「动作要点」写进提示词：

| 帧 | 提示词（在通用前缀后追加） |
| --- | --- |
| ① 接触（左脚前） | `walking, left leg forward, right leg back, legs slightly apart, stride` |
| ② 过渡 | `walking, passing pose, right foot lifted off the ground, left foot planted, legs close together` |
| ③ 接触（右脚前） | `walking, right leg forward, left leg back, legs slightly apart, stride` |
| ④ 过渡 | `walking, passing pose, left foot lifted off the ground, right foot planted, legs close together` |
| ⑤ 站立 | `standing, both feet together, neutral pose` |

通用前缀（保持和原图一致）：

```
masterpiece, best quality, chibi, 1girl, angel girl, white dress, front view,
clean lineart, cel shading, flat colors, white ankle socks, white shoes, …
```

参数：`512×512、28 steps、CFG 7、DPM++ 2M Karras、mask blur 6、denoise 0.52`。
**seed 固定成同一个值**（我这里用 `20241115`），几帧之间的鞋子、腿型风格才不会打架。

### 6.3 合成时的三个小技巧

1. **用 alpha 合成，不要用 RGB 贴图**：把重绘结果的 alpha 和羽化 mask 相乘再 `alpha_composite`，否则羽化边缘会把白背景混进来，成品一圈白毛边；
2. **接触帧把腿的下半段向两侧撇开**（512 坐标下约 13px，锚点在裙摆处），过渡帧不撇——这样才有"跨步"的横向幅度；
3. **起伏**：接触帧整体下移 1px、过渡帧上移 2px，模拟走路时重心的上下。

拼出来 5 帧：

![5 帧：接触 → 过渡 → 接触 → 过渡 → 站立]

<img width="1340" height="326" alt="06-angel-5frames" src="https://github.com/user-attachments/assets/b6e7a6a0-c17e-4114-9ee2-8da08e6bad56" />

用 9fps 播放（约 0.44 秒一个循环），效果大致是这样：

![走路循环 GIF（4 帧循环）]

<img width="256" height="256" alt="15-walk-loop" src="https://github.com/user-attachments/assets/59ea1e4f-c110-444d-b3e2-bad9dda7efd7" />

---

## 七、拼成精灵图：规格与规范

### 7.1 规格

WindowPet 的规则很简单：

- 精灵图**横向**排列，帧号从 **1** 开始，从左往右；
- 每帧是**正方形**，边长 = `frameSize`；
- 整图宽 = `帧数 × frameSize`，高 = `frameSize`；
- 必须是 **RGBA（带透明通道）PNG**。

我这张是 **1280×256 = 5 帧 × 256×256**：

![精灵图规格：1280×256，红线是每帧边界，帧号从左到右]

<img width="1360" height="376" alt="05-sheet-spec" src="https://github.com/user-attachments/assets/70cb5050-1387-472e-8394-6eb06eb67025" />

成品（透明背景）：

![最终精灵图 1280×256]

<img width="1280" height="256" alt="08-angel-sheet" src="https://github.com/user-attachments/assets/f1f0a195-1878-43e4-8748-85ea2604dbe0" />

### 7.2 一帧一个文件

做图阶段建议**一帧存一个文件**（`01.png`、`02.png`……），好处是：改某一帧只重跑那一张，重新拼图一句脚本就行。文件目录长这样：

```
F:\WindowPet\天使形象制作\
├─ frames\        ← 一帧一个文件：01.png 02.png 03.png 04.png 05.png
├─ 成品\          ← 原图 / 透明立绘 / 精灵图 / 预览
└─ 素材\          ← 抠图、局部重绘、拼帧的脚本
```

### 7.3 拼图脚本的要点

写个几十行的 Python 就能把「帧文件夹 → 精灵图」自动化，几个必做的校验：

- 每帧统一缩放/居中到 `frameSize` 正方形（不同尺寸的图不能直接拼）；
- 检查每帧是否真的带 alpha，没有就报警（否则成品会有白底）；
- 输出前断言 `宽 == 帧数 × 高`；
- 顺便生成一份 JSON 配置（帧尺寸、图路径、每个动作的起止帧），桌宠程序直接读。

这一步产出的两个文件，就是下一篇要装进 WindowPet 的东西：

- `小天使_精灵图_1280x256.png`
- `angel_pet.json`（`frameSize: 256`、`walk: 1-4` 之类）

---

## 八、踩过的坑 & 经验

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 模型死活加载不了，ETA 几千秒 | CLIP tokenizer 要连 HuggingFace，被墙 | `HF_ENDPOINT=https://hf-mirror.com` |
| 成品边缘一圈白毛 | 抠图时羽化边缘混进白底；或贴图用 RGB 而非 alpha | 白底去除公式 + alpha 合成 |
| 几帧之间鞋子/腿型不一样 | 每帧单独抽卡、seed 不同 | 固定 seed；只重绘局部 |
| 走路像"左右晃" | 只做了整图旋转/位移，脚没动 | 重绘腿部 + 接触帧撇腿 + 身体起伏 |
| 走路左右脚"反了" | 用镜像换脚，鞋子朝向被翻 | 不镜像，用提示词分别生成左右脚在前的帧 |
| 精灵图拼出来是歪的 | 各帧尺寸不一致 / 没居中 | 统一缩放居中到 frameSize |
| 桌宠显示区域不对 | 系统缩放不是 100%（高 DPI） | 见下一篇的调试章节 |

---

## 小结

这条流水线其实只有四个动作：**生图（绘世/WebUI）→ 抠图（透明 PNG）→ 补帧（局部重绘 + 合成）→ 拼图（横排 + JSON）**。

难点不在工具，而在两个"一致性"：**角色的形象要一致**（固定 seed、只改局部），**动作的节奏要参考真实样本**（这次是拆了内置甘雨的 walk 来量幅度）。

有了这张精灵图和 JSON，剩下的就是把它装进桌宠程序——下一篇：
**《WindowPet 安装配置和自定义桌宠的完整流程》**。

## **这篇博客的所有图尤其是天使的图仅用作博客展示，请不要将我的天使图片用于自己的桌宠。这是对我的尊重，谢谢**