
# 软件测试：基于 Selenium + Mocha + Mochawesome 的 Web 自动化测试与结果可视化

最近在推进软件测试的实验课项目，要求针对一个本地部署的 Web 系统构建完整的自动化功能测试流。老样子，找AI询问了一些在2000~5000行代码的网页项目，最终选择了名为 “Tienda de Donas” 的甜甜圈购买平台。随后鬼脑发力，想着学都学了整点好活，于是对甜甜圈购买平台进行二次开发，制作成了“哈基米南北绿豆购买系统”。在TdD的基础上增加了Selenium工具进行自动化测试，原有的Html/Css/Java框架不变，涵盖了商品渲染、购物车状态管理、金额计算以及原生弹窗交互等经典的前端业务逻辑。（简单来说套了个皮，嘻嘻）

而在技术栈的选择上，我放弃了 Python + unittest 或 Java + TestNG，直接接入 Node.js 生态，选用 **Selenium WebDriver** 负责浏览器控制，**Mocha** 作为测试组织框架，以及 **Mochawesome** 负责最终的可视化报告生成。TsetNG框架其实也很不错，对于软件测试来说是一个成熟的应用级框架，我选择的Mocha框架事实上也可以替换为TestNG框架，但原有的甜甜圈购买平台使用 Selenium + Mocha 的结构因此为了便于对比和修复，我并没有修改框架而是直接沿用了原有的框架。软件测试事实上也是我第一次在独立做，从一个开发人员的角度独立承担开发、测试、修复、优化的工作，遇到的问题也不少，比如网页搭建时想要实时展示页面修改后的外观从而发现的缓存机制、想要增加新功能从而尝试对框架和结构进行修改从而开始学习Mocha框架、想要使用Selenium WebDriver却发现插件商店没有进而到官网看更新公告下载桌面版……还是很有意思的，不光是学习各种技术，也有把自己喜欢的东西真的通过知识的学习呈现到大家面前的兴奋感。

---

## 问题一：前端缓存机制的延迟更新

终端运行被测系统，确保本地服务器端口正常开启后我还有需要更新的png图片或css格式，手动调整后想要快速验证静态资源是否修改完毕，随后就发现web界面与代码并不同步：修改了商品列表的 JavaScript 源码后，刷新浏览器界面依然渲染旧数据。（经典问题）

排查了代码问题、服务器启动问题、web页面甚至是插件后依旧找不到问题，事已至此求助AI了，随后才知道是 HTTP 强缓存（HTTP Strong Caching）的问题：`http-server` 默认会为静态资源注入 `Cache-Control: max-age=3600` 的响应头，导致浏览器在有效期内强制读取本地磁盘缓存，阻断了与服务器的网络请求。

**工程化解决方案**：
在 `package.json` 的启动脚本中，通过附加 `-c-1` 参数彻底禁用服务端的缓存策略：
```json
"start": "npx http-server -p 5500 -c-1"
```
这一改动不仅修复了开发时的视觉延迟，更重要的是保证了后续 Selenium 脚本执行时的环境一致性，避免测试断言读取到过期缓存导致假阴/假阳性 ————大肥鱼如是说道。

---

## 问题二：旧工具版本的组合和Selenium的内置驱动

环境准备就绪后，第一个 Selenium 脚本不出意外地出意外了，上来就报 `Timeout of 30000ms exceeded` 错误。查看了一下日志，日志追踪指向了 ChromeDriver 的会话建立阶段。

这个是驱动与浏览器版本不匹配了。本地 Chrome 浏览器已静默升级至 141.x，而项目依赖中锁定的依然是 115.x 的驱动版本。TdD项目还是太早了，Selenium和Chrome的版本依旧是旧组合。

**技术演进的红利**：
好在 Selenium 4.x 引入了全新的 **Selenium Manager**。这是一个内置的驱动管理 CLI，它在底层会自动探测本地浏览器的版本，并动态拉取匹配的驱动二进制文件。
我直接移除了 `package.json` 中冗余的 `chromedriver` 依赖，将 `selenium-webdriver` 升级至 `4.50.0`。通过 `Builder().forBrowser('chrome').build()` 的实例化过程，Manager 接管了底层复杂的版本映射，实现了真正的“零配置”驱动调度。

---

## 问题三：测试脚本设计中的异步等待与上下文切换

为了偷懒，我们的项目规格说明书我就写了7条：商品展示、加购累加、侧边栏可见性、总价计算、Alert接管和登录鉴权。然后测试用例也就写了7条，一般来说是要在2n+1甚至更多的测试用例上进行测试的，但为了偷懒嘛~对于实验课来说7个需求7条用例就够了。

哪怕只有7条用例，进行代码编写和测试执行的时候依旧遇到了错误（你看，这说明用例经典啊！）：

### 1. API 迭代导致的元素等待异常
在 TC-02 中，需求是验证购物车角标由 0 更新为 1。初版代码使用了：
`await driver.wait(until.elementTextIs(By.id('contador-carrito'), '1'), 5000);`
直接抛出 `TypeError: element.getText is not a function`。这是因为在 Selenium 4 的底层实现中，`elementTextIs` 方法期望接收一个已经解析的 `WebElement` 对象，而非单纯的 `By` 定位器。
**重构策略**：先执行 `findElement` 获取实体对象，再将其作为断言条件传入。

### 2. DOM 树存在与元素可见性的区别
在测试 TC-03（验证购物车侧边栏）时，调用 `until.elementLocated()` 拿到元素后直接读取文本，结果返回空字符串。由于被测系统使用了 Bootstrap 的 Offcanvas 组件，侧边栏在未展开时，其节点是存在于 DOM 树中的，但表现为 `visibility: hidden`。
**重构策略**：必须使用 `until.elementIsVisible()` 显式等待条件，确保元素的渲染状态（Rendered State）符合用户视觉预期，避免因动画延迟导致断言失效。

### 3. 原生弹窗的上下文接管
TC-05 模拟点击发送 WhatsApp 按钮，并验证原生 Alert。由于浏览器原生弹窗不属于 HTML DOM 树的一部分，常规的 `By.css` 无法定位，且弹窗会阻塞主线程。
**重构策略**：采用 `driver.switchTo().alert()` 将控制权交由 WebDriver 的 Alert 接口，读取文本后，务必执行 `alert.accept()` 释放阻塞状态，否则后续用例将全线崩溃。

> **简单的Selenium桌面版展示**
>
在Selenium项目官网下载

<img width="1969" height="1076" alt="Image" src="https://github.com/user-attachments/assets/c348bb97-a48d-4b60-870d-311c7e526e90" />
下载完毕最新版（保证是4.x版本才有桌面版）后创建项目即可。界面如下：
<img width="1920" height="1032" alt="Image" src="https://github.com/user-attachments/assets/db4516ec-9fdc-4d51-9532-8db09b103fed" />
后续进行测试就是点击左上角的开始小三角开始录制，点击小三角（事实上应该是小方形Pause）暂停录制，点击暂停小红点结束录制。
<img width="1920" height="1032" alt="Image" src="https://github.com/user-attachments/assets/33beceba-bb70-4885-909f-349b63294c95" />

---

## 阶段四：测试闭环与报告渲染

当所有的交互逻辑和显式等待（Explicit Wait）被正确编排后，在终端输入 `npm test`，Mocha 会按照生命周期钩子（`before` / `beforeEach` / `after`）自动隔离每次测试的上下文。

测试完成后，**Mochawesome** 展现了其作为定制报告框架的强大之处。它直接解析 Mocha 运行器产生的 JSON 日志，通过前端渲染引擎生成了静态的 HTML 报告。在报告中，我们可以直观地审视通过率（Pass Percentage）饼图、嵌套的用例树、执行耗时，以及异常发生时的堆栈追踪（Stack Trace）。这种数据可视化的能力，对于缺陷定位和质量度量极具价值。

<img width="1279" height="722" alt="Image" src="https://github.com/user-attachments/assets/19730c1a-7353-47dd-9645-4797012aa93e" />

---

## 写在最后的一点点小感想

这次实验是以开源社区的“Tienda de Donas甜甜圈购买系统”为基础进一步扩展的。很开心的一次实验，我觉得学技术的过程中能够做点自己喜欢的、感兴趣的东西，比如这次听起来就很不正经的“哈基米南北绿豆专卖店购买系统”。

<img width="1280" height="728" alt="Image" src="https://github.com/user-attachments/assets/deffffa6-8eb2-4d01-b36e-fd7b4bdc67f8" />

我觉得很开心，我学的技术能够让我把我喜欢的东西把我感兴趣的东西实实在在地呈现在我的面前，按照我的想法，从无到有实现了一个哈基米的网站。我很开心。
而在这个过程中其实也学到了很多。除了Selenium自动测试工具的使用，我认为收获最大的应当是了解了Mochawesome和Mocha测试框架。Mochawesome能够自动根据Mocha框架生成测试结果的html报告界面，支持对于测试结果的统计、展示、分类、计时等等功能。这是一个很好用的框架，代替了阅读终端界面输出的繁杂步骤。原有的日志文件和Json文件阅读起来相当不直观，我认为Mocha框架很好的弥补了这个缺点。当然，该有的report.json文件和日志记录也都在mochawesome-report目录下。
