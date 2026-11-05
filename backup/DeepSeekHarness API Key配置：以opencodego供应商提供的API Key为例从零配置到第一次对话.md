# DeepSeekHarness API Key配置：以opencodego供应商提供的API Key为例从零配置到第一次对话

DSH（DeepSeek Harness）本身不带模型额度。它把「模型」交给一个个**提供商（Provider）**：每个提供商对应一个 API 地址、一套鉴权方式和一组模型。你要做的，就是把某个提供商的 API Key 交给 DSH。

本文用 **OpenCode Go** 完整走一遍流程，最后再给出一节通用写法，换成任何 OpenAI / Anthropic 兼容的服务都能套用。

> **关于截图**：文中所有截图均为脱敏示意图，API Key、账号、邮箱等信息已打码或替换为占位符。你按图操作时，请务必也养成「先打码再截图」的习惯——原因见文末的安全清单。

---

## 0. 先理解：DSH 里的模型是从哪来的

打开 **设置 → 模型**，你会看到这样一页：

![DSH 设置 → 模型]

<img width="3200" height="2000" alt="Image" src="https://github.com/user-attachments/assets/cf88e167-4a02-47c6-b50f-d112101c26c5" />


- **每一行 = 一个提供商**。行首的小圆点是状态灯：**绿色 = API 密钥已配置**，**红色 = 密钥缺失**。
- `DeepSeek 账号` 是内置的登录态（不需要 Key，显示「已登录」）。
- `DeepSeek` 这类内置目录提供商，只要填 API Key 就能用。
- 想接入别的服务，点最下方的 **添加模型提供商**。

所以整件事可以概括成三步：**拿到 Key → 添加提供商并填入 Key → 选模型开始对话**。

---

## 1. 获取 API Key（以 OpenCode Go 为例）

OpenCode 的 API Key 在它的 Console 里创建。如果你还没有账号，先注册并订阅 Go（$10/月起），或者直接用按量计费的 Console 余额。

1. 打开 **<https://opencode.ai/console>** 并登录。
2. 点右上角账号菜单，进入 **Account → Personal API keys**。
3. 点 **Add API key**，依次选择：
   - **Workspace**：`Personal`（个人工作区）
   - **Key name**：起个能认出来的名字，例如 `DSH-Desktop`
   - **Expiration**：建议 `30 days`，而不是 `Never`
   - **Permissions**：`Restricted`，并且只勾 **Models → Request**（能列模型、能调模型即可）
4. 点 **Create**，然后**立刻复制**生成的 Key。

![在 OpenCode Console 创建并复制 API Key]

<img width="3200" height="2000" alt="Image" src="https://github.com/user-attachments/assets/686346a4-c05f-445d-a02a-a4b29a52af45" />


Key 的格式形如 `oc_sk_xxxxxxxx...`。

> ⚠️ **完整密钥只在创建时显示一次**。关掉弹窗后，Console 只保留前缀。没复制到就只能撤销重建。

**这一步的隐私要点**：复制完就切走，不要让它停留在剪贴板里；更不要把带完整 Key 的屏幕截进任何文档。

---

## 2. 在 DSH 里添加提供商

回到 DSH：

1. 打开侧边栏底部的 **设置**。
2. 在左侧导航选择 **模型**。
3. 点 **添加模型提供商**。
4. 添加方式选 **第三方模型提供商**（内置目录），然后在 **提供商** 下拉里选 `opencode-go`。

![添加模型提供商：选择第三方目录里的 opencode-go]

<img width="3200" height="2000" alt="Image" src="https://github.com/user-attachments/assets/5ef5fbed-0e77-4e2d-adb1-bc4017669d33" />


> 下拉框里显示的是**目录标识符**（如 `opencode-go`、`openai`、`anthropic`、`zai`），不是产品名。`opencode-go` 就对应 OpenCode 的 Go 订阅。

---

## 3. 填入 API Key 并保存

选中提供商后，卡片里会出现 **API 密钥** 输入框：

![填入 API 密钥]

<img width="3200" height="2000" alt="Image" src="https://github.com/user-attachments/assets/9c70e08b-d4ce-4354-af38-8d92c7ca69d8" />


几个细节：

- 输入框右侧的**眼睛按钮**可以临时显示/隐藏你正在输入的内容；改动或清空后会重新遮住。
- **已保存的密钥永远不会被读回输入框**。留空表示「保持当前已保存的凭据」。
- 如果你更习惯用环境变量，可以留空——DSH 会按顺序回退到：**启动环境 → `$cwd/.env` → `$DSH_HOME/.env`**。OpenCode Go 对应的变量名是 `OPENCODE_API_KEY`。
- 卡片里的「自定义设置」可以改 `API 地址`、模型目录等进阶项；内置目录提供商一般不需要动。

填好后点 **保存**。保存成功后，页面顶部会出现绿色提示，提供商行首的圆点也会变绿：

![保存成功，OpenCode Go 行显示绿色圆点]

<img width="3200" height="2000" alt="Image" src="https://github.com/user-attachments/assets/7de34e10-4cf5-456c-8e1e-3c18e5329b99" />

> 密钥是以**只写**方式存进 DSH 的凭据存储的（默认在 `~/.dsh/.credentials.yaml`），**不会**写进项目里的 `cordis.patch.yml` 之类的设置文件，所以不会被误提交到 Git。

---

## 4. 选择模型，开始对话

配置完成后，回到对话输入框，点下方的模型按钮（图中显示 `Kimi K3`），会弹出模型选择器：

![在模型选择器里挑一个 OpenCode Go 的模型]

<img width="3200" height="2000" alt="Image" src="https://github.com/user-attachments/assets/1fdc7a91-f503-416f-b077-d7a4d98a2ae2" />


- 列表按提供商分组，`OpenCode Go` 下就是 Go 订阅覆盖的模型（Kimi K3、GLM-5.3、DeepSeek V4.1 Flash、Claude Haiku 5.5……）。
- 支持「搜索模型…」按名字过滤。
- 选中的模型只对当前会话生效，可以随时切换。

到这里就完成了：DSH 会带着你的 API Key 去请求对应提供商的接口，第一次回复出现就说明配置成功。

---

## 5. 通用写法：换成任何 OpenAI / Anthropic 兼容服务

上面是「内置目录」的路径。如果你用的服务不在目录里（中转站、自部署、公司网关），或者你更喜欢环境变量 / 配置文件，可以看这一节。

### 5.1 内置目录提供商：只填 Key

DSH 的目录里已经预置了常见提供商，添加方式就是第 2、3 步，**只需要填 API Key**，地址和协议由目录决定。常见对应关系：

| 提供商 | 目录标识符 | 常用环境变量 |
| --- | --- | --- |
| OpenCode Go | `opencode-go` | `OPENCODE_API_KEY` |
| DeepSeek | `deepseek` | `DEEPSEEK_API_KEY` |
| OpenAI | `openai` | `OPENAI_API_KEY` |
| Anthropic | `anthropic` | `ANTHROPIC_API_KEY` |
| Moonshot / Kimi | `moonshotai` | `MOONSHOT_API_KEY` |
| Z.ai | `zai` | `ZAI_API_KEY` |

> 环境变量名以卡片里的提示为准：输入框下方的帮助文字会直接告诉你当前提供商支持哪个变量。

### 5.2 自定义模型 API：手填地址和协议

在「添加模型提供商」里选 **自定义模型 API**，需要填四样东西：

1. **Provider ID**：唯一标识，起一个不会和目录冲突的名字，例如 `my-gateway`。
2. **API 地址**：完整的基础 URL，必须是可解析的 `http` / `https` 地址（localhost、IP、自定义端口都可以）。
3. **API 协议**：三选一
   - `OpenAI Chat Completions`
   - `OpenAI Responses`
   - `Anthropic Messages`
4. **模型**：至少一个模型 ID。可以点 **获取可用模型** 让 DSH 去问端点要模型列表，再勾选；也可以手动添加。

以 OpenCode Go 为例，它文档里给出的基础地址是：

```
OpenAI 兼容：https://opencode.ai/inference/go/openai/v1
Anthropic 兼容：https://opencode.ai/inference/go/anthropic
```

（用内置的 `opencode-go` 提供商时，这些地址已经配好，不用手填。）

### 5.3 用环境变量代替手填 Key

不想把 Key 存进 DSH，也可以只放环境变量。DSH 的查找顺序是：

1. 启动 DSH 时的**进程环境**
2. 当前工作目录下的 **`.env`**（`$cwd/.env`）
3. **`$DSH_HOME/.env`**

在编辑卡片里把 **API 密钥** 留空即可。卡片会显示当前是否检测到环境变量；没检测到时会有黄色提示，并列出应该配置的变量名。

```bash
# 示例：只写变量名，值请自己保管
export OPENCODE_API_KEY="oc_sk_你的密钥"
```

### 5.4 密钥到底存在哪

| 位置 | 内容 |
| --- | --- |
| DSH 凭据存储（`~/.dsh/.credentials.yaml`） | 你在界面里填过的密钥，只写保存 |
| `cordis.patch.yml` / 设置文档 | **只有提供商配置和变量引用名，没有密钥值** |
| `.env` / 启动环境 | 你自己放的环境变量 |

理解这一点很重要：**密钥和配置是分开的**。你可以放心分享 `cordis.patch.yml`，但 `.credentials.yaml`、`.env` 和终端历史绝对不能外发。

---

## 6. 隐私与安全清单

这一节比前面所有步骤都重要，如果你看到这里还没有配置好，想要求助AI。那么这节是最重要的，有些内容不能发给AI否则会有很大可能泄露。简单来说，就是别人能从你口袋里拿钱了。

**截图前**

- 打码清单：完整或部分 API Key、账号邮箱、Workspace 名称、账单金额、用量统计、请求日志。
- 只露「前缀 + 省略号」也不一定安全，最稳妥是整段涂掉。
- 浏览器地址栏、标签页标题也可能带信息，截图时一起裁掉。

**使用时**

- 给 Key 设**过期时间**（30 天足够），并只给**最小权限**（能列模型、能调模型）。
- 一个用途一个 Key（DSH 一个、CI 一个），泄露时只需撤销一个。
- 不要把 Key 写进代码、README、博客、Issue 或聊天记录。
- 不要把 `~/.dsh/.credentials.yaml`、`.env`、shell 历史提交到仓库。

**怀疑泄露时**

1. 立刻到提供商控制台 **Revoke** 该 Key。
2. 新建一个 Key，更新 DSH（设置 → 模型 → 编辑 → 填入新 Key → 保存）。
3. 检查用量/账单，确认没有异常调用。
4. 如果 Key 曾进过 Git 历史，仅删除文件不够——要重写历史或直接换 Key。

---

## 7. 常见问题

**Q：保存后报 401 / 403？**
Key 无效、已过期、被撤销，或者权限不足。OpenCode 的 Key 如果只给了 `Read only`，能列模型但不能调用，就会 403。

**Q：保存成功但模型列表是空的？**
先确认该 Key 对应的订阅/余额是否覆盖你要用的模型（OpenCode Go 只包含 Go 列表里的模型）。自定义提供商则检查「模型」里是否至少加了一个 ID。

**Q：环境变量明明设了，卡片还说没检测到？**
检查变量名是否和卡片提示一致；确认它是在**启动 DSH 的那个环境**里设置的（不是另一个终端窗口），或者写进 `$cwd/.env` / `$DSH_HOME/.env`。

**Q：想换回 DeepSeek 官方 Key？**
添加 / 编辑 `DeepSeek` 提供商，填入 `DEEPSEEK_API_KEY` 即可；地址默认是 `https://api.deepseek.com/anthropic`，也可以用 `DEEPSEEK_BASE_URL` 覆盖。

**Q：圆点颜色分别代表什么？**
绿色实心 = 已确认密钥配置；红色实心 = 已确认缺失；没有圆点 = 该提供商不需要密钥（例如账号登录）。

---

## 附：速查

```text
获取 Key      opencode.ai/console → Account → Personal API keys → Add API key
配置 DSH      设置 → 模型 → 添加模型提供商 → 第三方模型提供商 → opencode-go
填入 Key      编辑卡片 → API 密钥 → 保存（留空则用已保存凭据 / 环境变量）
使用          输入框下方模型按钮 → 选择本会话使用的模型
凭据位置      ~/.dsh/.credentials.yaml（只写保存，不进设置文件）
环境变量      启动环境 → $cwd/.env → $DSH_HOME/.env
```

配置本身只有三分钟，真正花时间的是**养成不泄露密钥的习惯**，不仅是API Key，其他的隐私数据比如账号密码，工具私钥都不要上传给AI。AI Enhance是大流，我们能做的只是好好待在船上别被海浪冲下去了。
