tips：最近因为某hs某性别的用户把梁圣的算力都给干崩溃了，左手国家电网右手A股股市的梁天帝都被斩于马下，被迫将Deepseek网页版限额且限制7天不活跃对话停止继续……为了活命不多说了，我是个贪生怕死一贫如洗长得又丑还没人喜欢的肥宅，请不要把我挂到小某书上去……

先交代背景：在 WorkBuddy 里配 OpenCode Go——填了 Base URL、贴了 API Key、模型写 `deepseek-v4.1-flash`，点「测试连接」。

第一版提示**连接失败**。补上 `/chat/completions` 再试，还是失败。中间还踩了个纯拼写坑——`completions` 手滑写成了 `complement`。等地址真的改对，提示变成了**「模型服务拒绝了测试请求」**。

有意思的是：**每次报错都长得很像，但三次失败根本不在同一层。**

这篇短博客借这个案例，把「服务器鉴权」和「服务器兼容性」拆开讲清楚，再看两个正面例子：OpenCode 怎么把契约写成文档，DSH（DeepSeek Harness）怎么把契约写成配置。

---

## 一、先把两个概念分开

| | 服务器鉴权（Authentication / Authorization） | 服务器兼容性（Compatibility） |
| --- | --- | --- |
| 回答的问题 | 你是谁？你配不配用？ | 你说的话我听不听得懂？ |
| 典型返回 | `401`、`403` | `400`、`404`、`415`、业务错误码 |
| 出错原因 | Key 错、过期、被撤销、套餐不含该模型、地区限制 | 路径不对、请求体字段方言不同、少了必需 header、协议选错 |
| 谁能修 | 换 Key / 提权限 | 改客户端或换端点 |

一句话：**鉴权是查身份证，兼容性是对暗号。**

大多数「连不上模型」的求助，卡点在于这两件事被混成了一个词——「连接失败」。客户端只给一句含糊提示，用户就只能在界面上反复换 URL、开关功能，猜哪一格填错了。

所以排查的第一步不是改配置，是**让服务器把话说清楚**。

---

## 二、用 curl 把边界试出来

WorkBuddy 能不能连上先放一边，直接问 OpenCode Go 的网关。发三种请求，看服务器怎么回答（实测，已脱敏）：

**① 不带会话头**

```bash
curl -sS https://opencode.ai/inference/go/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v4.1-flash","messages":[{"role":"user","content":"ping"}],"max_tokens":8}'
```

```json
{"type":"error","error":{"type":"MissingSessionID",
 "message":"Request is missing x-opencode-session and cannot be routed efficiently. ..."}}
```

`HTTP 400`。注意：**连 Key 都还没校验**。网关先判断请求能不能被路由，缺 `x-opencode-session` 直接打回。

**② 带上会话头，但不带 Key**

```bash
curl -sS ... \
  -H "x-opencode-session: session-123" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

```json
{"error":{"type":"authentication_error","message":"OpenCode Go requires an API key"}}
```

`HTTP 401`。兼容性这一关过了，现在才轮到鉴权。

**③ 带会话头，Key 是假的**

```json
{"error":{"type":"authentication_error","message":"Invalid credential"}}
```

`HTTP 401`。服务器认出了「这是一次带凭据的请求」，只是凭据无效。

把三种回答放在一起，边界非常干净：

- **400 / `MissingSessionID` → 兼容性问题**：请求形态不符合网关约定；
- **401 / `requires an API key` → 鉴权缺失**：没带凭据；
- **401 / `Invalid credential` → 鉴权失败**：凭据不对；
- **200 → 通了**，此时任何失败都只可能出在客户端。

顺带一个容易被忽略的事实：`GET .../v1/models` **不带 Key 也返回 200**。列模型是公开的，鉴权只发生在真正生成的那一刻。所以「模型列表能拉、但对话报错」并不矛盾。

---

## 三、WorkBuddy 到底卡在哪

回头看那道表单：**接口地址 + API Key + 模型名**，再加几个功能开关（工具调用、图片输入、自定义协议）。

它能表达的东西就这么多。而 OpenCode Go 这类网关的契约至少有三层：

1. **路由层**：请求必须携带稳定的会话标识，网关才能做路由和 prompt 缓存——对没有原生头的通用客户端来说，就是 `x-opencode-session`；
2. **协议层**：每个模型有自己的「母语」端点。文档里写得很清楚——`gpt-6-luna` 走 `/openai/v1/responses`，`claude-haiku-5-5` 走 `/anthropic/v1/messages`，`gemini-4-argon` 走 `/google/v1beta/...`，而 `deepseek-v4.1-flash` 这类走 `/openai/v1/chat/completions`；
3. **请求体方言层**：即便都叫「OpenAI 兼容」，`store`、`developer` 角色、`max_tokens` 还是 `max_completion_tokens`、思考字段怎么传，各家仍有差异。

一个通用客户端，表单里没有「我发哪种协议」「我会不会补会话头」的位置；如果它默认不发 `x-opencode-session`，请求就永远停在第一层，用户看到的却只是一句「连接失败」。

**「OpenAI 兼容」只等于大方向对，不等于字段级对。** 这是整件事最值得记住的一句。

---

## 四、正面例子一：OpenCode 把契约写成了文档

同一个服务，换个角度看，OpenCode 的做法几乎是把答案摊开给你：

- **Endpoints 表**逐模型列出 Model ID、native 端点、对应的 AI SDK 包名；
- **Base URLs** 全列：`/openai/v1`、`/openai/v1/responses`、`/anthropic/v1/messages`、`/google/v1beta`；
- **客户端要求**写得直白：用自己的 User-Agent 表明身份，并为每段对话发送稳定的 session ID；
- 最狠的是两张表：**Validated clients** 和 **Known problematic clients**。哪个客户端、哪个版本、缺什么、对应哪个 PR/issue，全部公开。

其中一行写着 DeepSeek Harness：会话信息在部分模型路径上有、部分没有；服务端认它的原生头，剩下的适配仍在推进，附了讨论链接。

鉴权这一侧同样干净：错误体直接按 `type` 分开——`MissingSessionID` 是路由/兼容性问题，`authentication_error` 才是凭据问题。**服务器主动把「哪一层错了」告诉客户端**，这本身就是一种兼容性设计。

这件事的价值在于：**兼容性变成了可声明、可追踪、可复现的状态**，而不是让用户自己猜。文档敢把自己产品的「已知问题」列出来，恰恰说明这是被当作工程问题管理的。

---

## 五、正面例子二：DSH 把契约写成了配置项

DSH 这边是客户端视角。它的做法不是「写死一套 OpenAI 逻辑」，而是把差异建模成数据：

**每个提供商声明自己的鉴权方式。** 目录里 OpenCode Go 这一条大致是：

```js
createProvider({
  id: "opencode-go",
  name: "OpenCode Go",
  auth: { apiKey: envApiKeyAuth("OpenCode API key", ["OPENCODE_API_KEY"]) },
  models: OPENCODE_GO_MODELS,
  api: { /* 三种协议各一个适配器 */ },
})
```

于是「用哪个环境变量、密钥存哪」是元数据，不是散落的 if；密钥只写保存，不会回显。

**每个模型自带协议和基地址。** 目录数据长这样：

```json
"chat:deepseek-v4.1-flash": {
  "api": "openai-completions",
  "baseUrl": "https://opencode.ai/zen/go/v1",
  ...
},
"chat:minimax-m3": {
  "api": "anthropic-messages",
  "baseUrl": "https://opencode.ai/zen/go",
  ...
}
```

同一个提供商下，不同模型走不同协议、不同基地址——这是数据，不是代码分支。DSH 界面上「自定义模型 API」里让你三选一协议（OpenAI Chat Completions / OpenAI Responses / Anthropic Messages），同一套抽象也照顾到了自建网关。

**兼容性差异有细粒度开关。** 上面那个 DeepSeek 模型还带一组 `compat`：不支持 `store`、不支持 `developer` 角色、`max_tokens` 走哪个字段、思考格式叫什么……都是逐模型的显式声明。

**会缺的 header 由适配器补。** DSH 专门有一段 OpenCode 适配：

```js
const OPENCODE_SESSION_HEADER = "x-opencode-session";
// 在 API 分发前，为每次会话补上路由所需的会话头
```

这正是这类通用客户端最容易缺的一层。

也要说句公道话：OpenCode 文档里 DSH 仍挂在 Known problematic 一栏，说明这件事没有「一次做完」，需要持续跟进。**但能被写进文档、能定位到具体适配器，就已经是可控的工程问题了。**

---

## 六、照着做的排查顺序

1. **先 curl，别先改界面。** 用可复现的最小请求问服务器，拿到真实状态码和错误 `type`。
2. **按状态码分层。** `400/404/415` 看兼容性；`401/403` 看鉴权；`200` 才轮到怀疑客户端。
3. **对照文档选端点。** 别默认所有模型都走 `/chat/completions`，先查 model → native endpoint 表。
4. **客户端填不下的，用代理补。** 缺 header 这类问题，本机一个小代理转发并补头，往往比换客户端快。
5. **把结论写回去。** 提 issue、补文档。兼容性问题不写下来，下一个人还要重走一遍。

---

## 结语

WorkBuddy 的那个案例，表面是「URL 少写了一段」，本质是**客户端和网关之间不兼容**：网关要求的东西（会话头、协议、字段方言），客户端的表单里没有对应的表达位置，只能失败在某一层，然后吐一句「连接失败」。

OpenCode 把契约定在文档和错误码里，DSH 把契约定在 provider 配置和适配器里。它们不靠猜，靠声明。

所以「兼容」这两个字，从来不是一句营销词。它是一份双方都要签的接口协议——**签了，才叫兼容；没签，只能叫运气好。**

---

*参考资料：OpenCode Go 文档（<https://opencode.ai/docs/console/go/>）——Endpoints、Validated clients、Known problematic clients 均出自该页；文中 400/401 返回可自行用 curl 复现，模型清单见 `https://opencode.ai/inference/go/v1/models`。DSH 侧实现细节取自本机安装包中的 provider 目录与 OpenCode 适配器源码。*
