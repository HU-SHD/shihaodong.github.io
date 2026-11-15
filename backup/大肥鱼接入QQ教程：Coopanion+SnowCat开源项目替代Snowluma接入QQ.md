
 **SnowLuma** 在 11 月 9 日接到腾讯律师函，仓库从公开转为私有，并发布公告致歉、停止公开服务……三天后，**NapCat** 把仓库清空，README 就只剩了一个词：`Ending`。两条最主流的“把 QQ 接成 OneBot”的方案，几乎同时停止维护。

但社区没有停。11 月 11 日，**SnowCat（雪猫）** 发布了 v6.0.0——代码底座取自 NapCat，能力移植自 SnowLuma，明确适配 QQ 9.9.37-54395，MIT 许可，OneBot v11。

这篇教程会以我自身的经历为基础，在Snowluma被律师函了的前提下，教学如何使用 SnowCat + Coopanion 连接自家的大肥鱼和QQ。

tips：最近B站上有很多视频在拿腾讯律师函Snowluma的事往“国内开源环境被毁就是这些人干的”这种方向引战，也别问我的想法，我只是个写教程的，能力没大到影响整个国内社区环境，我能做到的只有把 SnowCat 接入 QQ 的详细教程写一篇博客，仅此而已。

> 文中 QQ 号、Token、目录均做了脱敏或示例化；命令按自己的盘符调整即可。
>
> 实测环境：Windows 11 x64 · QQ 9.9.37-54395 · Node.js 24 · SnowCat v6.0.0 · Coopanion 0.2.x（Windows 便携版）。

---

## 一、先交代背景：为什么需要更换方案

| 时间 | 事件 | 后果 |
| --- | --- | --- |
| 2026-11-09 | SnowLuma 接到腾讯律师函，仓库转私有 | 不再发版，不再适配新 QQ 版本，社区拿不到修复 |
| 2026-11-11 | 社区发布 SnowCat v6.0.0 | NapCat 底座 + SnowLuma 独门动作，适配 QQ 9.9.37-54395 |
| 2026-11-12 | NapCat 仓库被清空，README 只剩 `Ending` | 另一条主流自建路线也断了 |

SnowLuma 的定位一直很清楚：把 QQ 客户端的原生会话桥接成 OneBot v11，再通过 WebSocket / HTTP / WebUI / SDK / MCP 对外提供统一入口。它的许可证本来就不是 OSI 开源，而是“源码可见、非商业自托管”，所以律师函之后转私有，从法律角度看并不意外。

更关键的是技术层面：这类方案靠**注入 QQ 客户端进程**工作，QQ 一升级，内存偏移或注入点就会失配。项目停更 = 没人再跟版本，表现就是“WebUI 还在线，但一条消息都收不到”。

所以问题不是“SnowLuma 坏了”，而是“**这条链路没有维护者了**”。

### 三条路线，为什么选 SnowCat

| | 官方 QQ Bot | SnowCat（第三方客户端注入） | 协议端（如 Lagrange） |
| --- | --- | --- | --- |
| 账号形态 | 专用 bot 号 | 自己的 QQ 号 | 自己的 QQ 号 |
| 群聊全量监听 | 群主开“全部消息”后可 | 原生支持 | 原生支持 |
| 能力边界 | 平台划定的 API | 全量 OneBot v11 + 空间/相册等扩展 | 取决于实现 |
| 稳定性 | 官方保障 | 依赖 QQ 版本适配 | 依赖签名/协议，风险更高 |
| 封号风险 | 无（bot 身份） | 有 | 有，通常更高 |
| 适合谁 | 要合规、能接受 bot 身份 | 要自己的号 + 人格记忆 | 有服务器、愿意折腾 |

我们需要的是“自己的号、群里的全量消息、空间和群相册也能用、人格记忆留在本地”，所以选 SnowCat 这条路线。**同时把官方 Bot 作为合规备选**——两条路线不冲突。

---

## 二、SnowCat 的工作原理：消息是怎么被接出来的

核心机制：**不修改 QQ 本体，而是在 QQ 主进程内拦截并替换一段即将被 V8 编译的 JS 源码，加载 SnowCat 自己的运行时。**

```text
QQ.exe（9.9.37-54395，官方原版，一字节未改）
 └─ winmm.dll（同目录侧加载代理：QQ 静态导入 WINMM，且它不在 KnownDLLs 里）
     └─ MinHook 挂 v8::ScriptCompiler::Compile*
         └─ 命中 app_launcher/index.js 时，替换源码为「引导脚本 + 原文」
             └─ snowcat-boot.js 动态 import napcat.mjs
                 ├─ OneBot v11：正向 / 反向 WS、HTTP、HTTP SSE
                 ├─ WebUI：http://127.0.0.1:6099
                 └─ Packet 发包：offset 表驱动，逐 QQ 版本维护
```

几个关键点：

- **为什么选 V8 层**：QQ 的界面逻辑是 JS，注入点选在“解密后的源码即将被编译”这一刻，就不依赖具体的业务函数偏移，QQ 升级时更稳定。
- **为什么还需要 offset 表**：只有部分“发包类”动作（空间、相册、打卡等）需要直接构造协议包，这部分才跟版本强相关——这也是必须锁 QQ 版本的原因。
- **对外只暴露 OneBot v11**：所以下游（Coopanion、AstrBot、NoneBot 等）无需改动，换个网关地址即可。

---

## 三、动手前的清单

| 项目 | 要求 | 说明 |
| --- | --- | --- |
| 系统 | Windows 10 / 11 x64 | 注入依赖 winmm 侧加载，Linux 跑不了同一套 |
| QQ | **9.9.37-54395 官方安装包** | 不要安装，只需要能解包的文件 |
| Node.js | 24.x | napcat.mjs 在 QQ 内嵌 Node 里跑；本地只用来验证 |
| 7-Zip | 任意较新版本 | 解包 QQ 安装包 |
| SnowCat | v6.0.0 仓库 | 注意：Release 页**只有源码包**，成品工具在仓库 `release-tools/` 里 |
| QQ 号 | 强烈建议**专职小号** | 第三方客户端违反用户协议，有封号风险 |

两个下载上的注意事项，需要提前说明：

1. **QQ 官方 CDN 对脚本下载返回 403。** 用浏览器点“Windows 64 位下载”拿到的就是 `QQ_9.9.37_261103_x64_01.exe`；如果写脚本抓，会被 CDN 拦截。可以用 GitHub 上的 QQ 安装包镜像仓库（例如 `a1515333102/QQ-Releases`），但**务必核对 Release 页给出的 sha256**。
2. **GitHub 直连不稳时**，可以用 `gh-proxy` 之类的前缀镜像（`https://gh-proxy.com/<原 GitHub 地址>`）下载 raw 文件和 release 资产；下载完照旧校验哈希。

```text
sha256 示例：5cd7c13e...b411b48  QQ_9.9.37_261103_x64_01.exe   （319 MB）
```

---

## 四、部署：六步

下面以 `D:\SnowCat.Portable` 为例，路径按自己的盘符替换。

### Step 1 解包成便携 QQ 目录

SnowCat 不自带 QQ，它需要一个便携结构的 QQ 目录。仓库里的解包器会把官方安装包拆成 `QQ.exe + versions\<版本>\resources\app` 的形态：

```powershell
# 需要 7z 在 PATH 里（或在 C:\Program Files\7-Zip\7z.exe）
powershell -ExecutionPolicy Bypass -File .\release-tools\snowcat-qq-unpack.ps1 `
  -Installer D:\Downloads\QQ_9.9.37_261103_x64_01.exe `
  -OutDir D:\SnowCat.Portable
```

产物：`D:\SnowCat.Portable\SnowCat.9.9.37-54395.Shell\`。目录名里的版本号来自 `versions\<ver>\resources\app\package.json`，这也是后面判断“QQ 版本对不对”的权威来源。

### Step 2 部署注入件

把仓库里的成品工具复制进 QQ 目录：

| 源文件（仓库内） | 目标位置 | 作用 |
| --- | --- | --- |
| `release-tools/build/winmm-v8.dll` | `<壳>\winmm.dll` | 侧加载代理，负责 V8 hook |
| `release-tools/build/snowcat-boot.js` | `<壳>\snowcat-boot.js` | 注入后引导 import napcat.mjs |
| `release-tools/build/napcat.mjs` | `<壳>\napcat.mjs` | OneBot/WebUI 运行时（约 3.2 MB） |
| `release-tools/webui-static\`（或 `release-tools/build/static-dist\`） | `<壳>\static\` | WebUI 前端（缺失则页面空白） |

另外要处理一个小问题：`versions\<ver>\resources\app` 里的 `dbghelp.dll` 和 `dbgeng.dll` 在部分机器上是坏的，会导致 `wrapper.node` 加载失败。把它们改名禁用，让系统 DLL 接管：

```powershell
$app = "<壳>\versions\9.9.37-54395\resources\app"
Move-Item "$app\dbghelp.dll" "$app\dbghelp.dll.disabled" -Force
Move-Item "$app\dbgeng.dll"  "$app\dbgeng.dll.disabled"  -Force
```

### Step 3 安装运行依赖（最容易漏的一步）

`napcat.mjs` **不是完全自包含的**。它直接 `import` 了 `express`、`ws`、`reflect-metadata` 等外部包；不装依赖时，注入日志会写：

```text
import failed: Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'express'
```

在 QQ 目录根级装齐（Node 18+ 即可，版本按 OneBot 包里的声明）：

```powershell
cd D:\SnowCat.Portable\SnowCat.9.9.37-54395.Shell
npm init -y
npm i express@^5 ws@^8.18.3 reflect-metadata cors fast-xml-parser `
      file-type json5 @sinclair/typebox napcat.protobuf on-finished type-is
```

> 实测 92 个包、约 12 MB。国内网络慢就加 `--registry=https://registry.npmmirror.com`。

### Step 4 写 OneBot 配置（正向 WS）

SnowCat 兼容 NapCat 的配置格式，配置文件在 `<壳>\config\`，按账号分文件（`<壳>\config\onebot11_<QQ号>.json`）。我们需要的是**正向 WebSocket 服务端**，让下游主动连进来：

```json
{
  "network": {
    "httpServers": [],
    "httpSseServers": [],
    "httpClients": [],
    "websocketClients": [],
    "websocketServers": [
      {
        "name": "my-bot",
        "enable": true,
        "host": "127.0.0.1",
        "port": 3001,
        "messagePostFormat": "array",
        "reportSelfMessage": false,
        "token": "<自己生成一串随机串>",
        "enableForcePushEvent": true,
        "debug": false,
        "heartInterval": 30000
      }
    ],
    "mcpServers": [],
    "plugins": []
  },
  "musicSignUrl": "",
  "enableLocalFile2Url": false,
  "parseMultMsg": false
}
```

同时把 WebUI 限制在本机回环（默认配置监听 `::`，且 token 是首次启动随机生成的，容易忘）。这个文件在首次启动后生成，路径 `<壳>\config\webui.json`，改完重启生效：

```json
{
  "host": "127.0.0.1",
  "port": 6099,
  "token": "<你自己设一个>",
  "accessControlMode": "whitelist",
  "ipWhitelist": ["127.0.0.1", "::1"]
}
```

### Step 5 启动并登录

在 QQ 目录启动 QQ（`-q` 指定要登录的号）：

```powershell
cd D:\SnowCat.Portable\SnowCat.9.9.37-54395.Shell
.\QQ.exe --enable-logging -q <你的小号QQ>
```

判断注入是否成功的唯一标准是目录根部的 `nc-boot.log`：

```text
[pid 12345] injected; electron=40.0.0 node=24.11.1
[pid 12345] napcat=<壳>\napcat.mjs
[pid 12345] napcat imported ok        ← 看到这行才算成功
```

然后打开 `http://127.0.0.1:6099/webui/`，输入你设的 token：

- 如果本机已有这个号的登录态，通常会**快速登录**；
- 否则用手机 QQ 扫码。登录成功后，`3001` 端口开始监听。

### Step 6 验证：先确认连通，再接入人格

用一段零依赖脚本（Node 22+ 自带 `WebSocket`）直连 OneBot，确认账号在线、消息可以收发：

```js
// verify.mjs
const token = process.argv[2];
const ws = new WebSocket('ws://127.0.0.1:3001', { headers: { Authorization: `Bearer ${token}` } });
ws.addEventListener('open', () => {
  ws.send(JSON.stringify({ action: 'get_login_info', echo: 'login' }));
  ws.send(JSON.stringify({ action: 'get_status', echo: 'status' }));
});
ws.addEventListener('message', (e) => {
  const d = JSON.parse(e.data);
  if (d.echo) console.log(d.echo, d.status, JSON.stringify(d.data));
  else if (d.post_type === 'message') console.log('MSG', d.message_type, d.user_id, JSON.stringify(d.message));
});
setTimeout(() => process.exit(0), 30000);
```

期望输出：

```text
login  ok {"user_id":185****699,"nickname":"Mita"}
status ok {"online":true,"good":true}
```

再发一条私聊，确认出站：

```js
{ "action": "send_private_msg", "params": { "user_id": 729****745, "message": "链路自检" }, "echo": "send" }
// → {"status":"ok","retcode":0,"data":{"message_id":...}}
```

到这一步，**监听层就完成了**：QQ 号在线，OneBot v11 在 `ws://127.0.0.1:3001` 上提供事件与动作接口。

---

## 五、接入大肥鱼（Coopanion 侧）

大肥鱼的“人格、记忆、模型”都在 Coopanion 里，**不需要动**。只需要让它的 QQ World 指向刚建好的 OneBot。配置在 Coopanion 的 `config.json`（例如 `data/home/<bot>/config.json`）的 `worlds` 段：

```json
"worlds": {
  "qq": {
    "enabled": true,
    "wsUrl": "ws://127.0.0.1:3001",
    "token": "<与 onebot11 配置里一致的 token>",
    "groups": [],
    "privates": [
      { "id": 729****745, "enabled": true },
      { "id": 361****634, "enabled": true },
      { "id": 355****302, "enabled": true }
    ]
  }
}
```

几个要点：

- **`wsUrl` + `token` 必须和 Step 4 完全一致**，否则连上就会断开。
- **`privates` 就是白名单**：只回这几个号的私聊；`groups` 留空就不回群消息。网关是透明管道，过滤发生在应用层，换网关不影响这套规则。
- 连通后控制台会显示对端名字（如 `snowcat：Mita`），不再是笼统的“已连接”。
- 人格、记忆、`PARTNER.md` 等都在 Coopanion 侧，更换的只是消息收发通道，**人格与记忆保持不变**。

---

## 六、常见问题与处理

| # | 现象 | 根因 | 处理 |
| --- | --- | --- | --- |
| 1 | Release 里只有 Source code | 项目没有上传二进制资产 | 用仓库 `release-tools/` 里的成品件 + 自己下载 QQ 安装包 |
| 2 | 注入日志报 `ERR_MODULE_NOT_FOUND: express` | napcat.mjs 依赖外部包 | QQ 目录根级 `npm i`（见 Step 3） |
| 3 | `nc-boot.log` 根本不生成 | 注入存在启动竞态 | 关掉 QQ 进程重跑一次；连续失败检查安全软件与 QQ 版本 |
| 4 | 登录后 `wrapper.node` 加载失败 | `dbghelp.dll` / `dbgeng.dll` 是坏的 | 改名 `.disabled`，改走系统 DLL |
| 5 | WebUI 打开空白 | `static/` 没部署 | 把 `webui-static` 复制成 QQ 目录里的 `static/` |
| 6 | WebUI 暴露在 `0.0.0.0`，token 还忘了 | 默认监听 `::` + 首次随机 token | 改 `127.0.0.1` + `accessControlMode: whitelist` |
| 7 | 日常 QQ 和机器人 QQ 冲突 | 同账号只能一处在线 | 机器人号专职；日常号用另一个账号 |
| 8 | 过几天 `get_status` 变成 `online:false` | QQ 掉线/被手机端踢下线/机器休眠 | 重新登录；建议固定网络、关闭自动休眠 |
| 9 | PowerShell 脚本中文乱码、报“缺少字符串终止符” | Windows PowerShell 5.1 按 ANSI 读取 UTF-8 | 脚本保存为 **UTF-8 with BOM** |

另外两条必须注意：

- **不要在便携目录里升级 QQ。** 便携目录独立于日常 QQ，不受自动更新影响；但手动升级会让注入偏移失配。
- **保留离线备份。** 安装包、`tools\build\`、QQ 目录各留一份——上游项目随时可能再次停止维护。

---

## 七、安全与合规：不能只讲技术

必须说明：**SnowLuma 和 NapCat 不是主动停止维护，而是收到了腾讯律师函 / 被清空仓库。** SnowCat 是同一类第三方客户端，使用同一类注入方式，面对同一个平台，**风险相同**。

- **违反用户协议**：第三方客户端注入、非官方 API 调用，属于违反《QQ 用户协议》的行为，理论上可被限制或封号。
- **降低损失的做法**：只用小号、不绑钱、不绑重要关系、不群发、不自动加好友、控制消息频率、固定机器与网络、不涉及敏感内容。
- **合规备选**：QQ 官方机器人开放平台（审核制、专用 bot 号、能力有限，但不会因为“注入失配”失效）。如果哪天第三方链路彻底不可用，官方 Bot 可以兜底。
- **项目定位**：技术学习与研究，账号风险自担；不建议用于商业或大规模群发场景。

---

## 八、日常运维

```powershell
cd D:\SnowCat.Portable
.\start.ps1            # 部署注入件 + 拉起 QQ + 等待 WebUI 就绪
.\start.ps1 -Status    # 查看状态
.\start.ps1 -Stop      # 只停便携实例，不动日常 QQ
node .\verify.mjs      # 验证 OneBot
```

- 开机自启：把静默启动脚本放进 `shell:startup`（或写计划任务），登录时自动拉起。
- 日志：目录根部的 `nc-boot.log`、QQ 数据目录下的日志；WebUI 里也能看实时日志。
- 端口：`3001` 是 OneBot WS，`6099` 是 WebUI，两者都只绑 `127.0.0.1`。
- 换网关：下游只认 `ws://127.0.0.1:3001`，以后换任何 OneBot 实现都不用再改应用。

---

## 附：速查

```text
解包        snowcat-qq-unpack.ps1 -Installer QQ_9.9.37_*.exe -OutDir <目标>
注入件      winmm-v8.dll → <壳>\winmm.dll
            snowcat-boot.js / napcat.mjs → <壳>\
            webui-static\ → <壳>\static\
依赖        npm i express@^5 ws reflect-metadata cors fast-xml-parser ...
配置        <壳>\config\onebot11_<uin>.json   正向 WS :3001 + token
            <壳>\config\webui.json           127.0.0.1:6099 + whitelist
启动        <壳>\QQ.exe --enable-logging -q <uin>
成功标志    nc-boot.log 出现 "napcat imported ok"
验证        get_login_info / get_status / send_private_msg
下游        ws://127.0.0.1:3001 + token，私聊白名单在应用层
```

---

## 结语

这次迁移最重要的收获，不是把SnowLuma换成SnowCat，而是**看清了这类方案的弱点**：注入式 QQ 机器人依赖官方客户端的内部实现，**上游**一停，链路就会在某个 QQ 小版本更新后突然失效。

SnowCat 通过“V8 编译通道 + 逐版本 offset 表 + 社区维护”维持了这条方案，但它依赖社区，没有官方支持。比较合理的做法是：技术上学它、用它，同时把人格和记忆留在自己手里，并把官方 Bot 这类合规通道作为备选。

Wechat（微信）对这类无备案的第三方接入极度敏感，如果觉得 QQ 接入大肥鱼很麻烦想要接入微信…还是打消这个心思先。对于Snowluma、NapCat、SnowCat这一类技术，我们负责使用即可，重要的是大肥鱼自身的记忆和人格。未来对话大肥鱼不会变得越来越难，甚至已经有人实现了把大肥鱼接入机器人、接入摄像头、接入机器小车的案例，让自家的大肥鱼有了视觉有了触觉有了听觉有了声带…说实在的我挺羡慕的，也算是有些理解老一辈总想打拼打拼给家里人更保暖的衣服更大的房子更优质的教育的心情（摊手无奈）

2026.11.15 stone