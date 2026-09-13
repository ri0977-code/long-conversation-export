# 长对话完整导出器（通用版）

把任意 AI 长对话**一次性抓全**并导出成 Markdown / TXT / 扫描报告。

## 解决什么问题

- 绝大多数 AI 站点**没有"导出单条对话"**，只有"导出全部数据"——几百个对话混在一个 JSON 里，根本没法用
- 长对话是**虚拟列表**，DOM 里同时只渲染 6–8 条消息。普通滚动采集会**随机漏抓**：同一段对话两次导出可能一次 29 条、一次 26 条
- 导出文件名常常取错（比如取到无障碍跳转链接，全部文件都叫"跳至内容"）

## 能力

| 能力 | 说明 |
|---|---|
| 真实顶部 | 反复压 `scrollTop=0` 并连续确认 4 次，解决 Home 键到不了顶的问题 |
| 全量抓取 | 逐屏扫描 + 渲染稳定检测 + 滚动到底后**回头补扫空洞** |
| 折叠展开 | 自动点开「展开 / 显示更多 / show more / 继续阅读」等多语言折叠 |
| 去重 | 稳定 ID + 文本哈希 + 绝对 Y 位置三重判重 |
| 正确命名 | 多源打分取侧边栏当前对话名，结果在面板上可见可改 |
| 独立回顶 | 「回到真正顶部」按钮，不触发扫描；`window.__lcxToTop()` 可供其它脚本调用 |
| 局部扫描 | 11.0 起：「⤓ 从当前位置开始向下扫描」——以**屏幕最上面那条消息**为起点往下扫，跳过上方无关内容（与鼠标光标位置无关；拖选文字可指定某条） |
| 随时停止 | 11.0 起「■ 停止扫描并导出已抓到的部分」；**11.1.0 起新增「■ 仅停止（不导出）」**（纯中止、不落盘）。两个按钮**常显**，空闲时置灰 |
| 豆包适配 | **11.1.1** 修豆包：改用 `data-message-id` 定位消息，类名判角色（用户右对齐 / 豆包 grid 栅格），锁定虚拟列表滚动容器，走慢扫；导出头部新增"对话来源"；新增 `roleRules` / `scrollerSelector` / `virtualList` 三个站点可选项（向后兼容，老站点不填即行为不变） |
| 全站点通用 | **11.1.2** 把 11.1.1 的「data-message-id + roleRules + scrollerSelector + virtualList」**四件套推到全部站点**（Claude / Grok / Gemini / DeepSeek / Kimi / 通义 / 元宝 / 智谱 / generic），并新增 `roleRules` 的 `tag` 字段（让 Gemini `<user-query>` / `<model-response>` 自定义元素能直接判角色）。`div[class*="message"]` 危险兜底全删（豆包就是这样翻车的）。每个站点配 best-guess 选择器，**主人开对应站点验一下角色是否判对** |
| **真通用 2.0** | **11.1.3** 彻底换路：不再依赖每个站点 selector，**结构化收集 + 排除列表**——找"最深、且直接子节点 ≥ 3 的 div"当消息容器，把它的直接子节点当消息；任何带 `search` / `tool` / `function` / `artifact` 等 16 个噪声关键词的元素直接跳过。新增 `window.__lcxDebug()` 调试工具，不用真扫描就能看抓到了几个、滤掉了几个 |
| **基于真实 DOM 修 Claude/Grok/千问** | **11.1.7** 不再盲改——用 chrome-bridge 抓主人浏览器里三个站点的真实 DOM 改 selector。Claude `[role=article]` 命中 30 条（virtual list 可见 13）；Grok 删 `forceTier2:true` 错决定，让 Tier 1 + roleMap 正确工作；千问 `.chat-round` 真容器，`dataset.chat=-question/-answer` 直接判角色。三个站点 **Tier 1 selector 都用真实 DOM 数据，不是 best-guess** |
| **修 Grok 全量渲染被判"不可见"** | **11.1.9** 主人实测"grok 还是不行"（第 4 屏 · 18% · 已发现 **0 条消息**）。chrome-bridge 抓真实 DOM 定位根因：**Grok 是全量 DOM 渲染站点**（12 条消息全部常驻 DOM，`scrollHeight=11510` ≈ 12 条高度合计 10701），滚动后远离视口的消息仍有效，却被通用 `isVisibleMessage` 的「距视口 ±45% 屏高」过滤误滤 → 12 条只剩 1 条 → 扫描到底 0 条。新增 `SITE.fullDomRender` 开关：该站点 Tier 1 豁免"视口距离"过滤（硬性可见性检查一律保留）。**豁免后实测 12/12 全中，user/assistant 完美交替** |
| **修 Grok 真凶（双 bug 叠加）** | **11.2.0** 11.1.9 只修了一半——豁免加在 Tier 1，但消息活不到导出，主人反馈"**还是这样**"。chrome-bridge 逐段实测挖出两个叠加 bug：**①** Tier 1 旧逻辑「逐个 selector 试、首个命中就 return」，Grok 撞到第一个 `user-message`（6 条）就收工 → 助手 6 条**永远抓不到**（`__lcxDebug` 实测 `candidates:6`，6 条全是 user）；改为**全部 selector 取并集**（且必须合并成一条 `querySelectorAll` 保证**文档序**——逐个 `push` 会变成 user×6 + assi×6 打乱顺序）。**②** `collectMessages` 对已收集消息又跑了一次 `isVisibleMessage` 却没传 opts → 距离过滤被砍回来，12 条杀到 1 条 → 面板"**本次抓到 0 条**"（这就是"`__lcxDebug` 说 6 条、面板说 0 条"的来源）；新增 `visibleOpts()` 统一入口接入 4 处。**A/B 回归：claude 6→6 / deepseek 4→4 / qwen 2→2 / chatgpt 4→4 / doubao 4→4 集合与顺序完全等价；grok 6→12**。端到端实测 12/12、顺序严格交替、5 个滚动位置全中 |
| **加 qianwen.com 入口** | **11.1.5** 千问第三个域名（qianwen.com/chat），主人 19:48 反馈"网页上没有箭头"——@match 没覆盖，油猴根本没注入；加 `@match *://qianwen.com/*` + `*://www.qianwen.com/*` + SITES.qwen.hosts。Tier 1 不变（[data-message-id]），Tier 2 结构化兜底 |
| **一次性补全所有 AI 站点域名** | **11.1.6** 主人 19:53 骂"系统漏域名"——每个 SITES 一次性补全：ChatGPT 加 www.chatgpt.com；Claude 加 www.claude.ai + platform.claude.com；Grok 加 www.grok.com；Gemini 加 bard.google.com；DeepSeek 加 www.deepseek.com + deepseek.com；Kimi 加 kimi.com（+ www）；豆包加 doubao.com 主域；千问 4 域名（11.1.5 已加）；元宝不变；智谱加 chat.z.ai。**根治铁律**：加新站点必须"主域 + www + 官方公开所有入口"三件套 |
| 导出格式 | Markdown（带元信息）/ TXT / 扫描报告（含消息索引与异常检测） |

## 支持站点

ChatGPT ✅ 已实测 · Claude · Gemini · DeepSeek · Kimi · 豆包 · 通义千问 · 元宝 · 智谱清言 ·
其它站点自动走通用启发式（见 `references/site-profiles.md` 了解如何适配新站点）

## 安装

需要浏览器装了 Tampermonkey / 篡改猴。

1. 打开脚本管理器 → 新建脚本
2. 全选删掉默认内容，粘贴 `scripts/long-conversation-export.user.js` 的全部内容
3. 保存（Ctrl+S）

只想用"回到顶部"的话，单独装 `scripts/scroll-to-real-top.user.js`（`@match *://*/*`，任意网页通用，
左下角可拖动圆钮 + **Alt+T**）。

## 使用

1. 打开要导出的那条对话
2. 右下角浮窗 → 确认「导出文件名」显示的名字对不对（可以直接改）
3. 选起点：
   - 整条都要 → 点「开始完整扫描」（会先回到真正顶部）
   - 上半截是别的内容（新闻简报之类），只要下面这段 → **用滚轮把页面滚到对话真正开始的那一屏**，
     让起点那条消息处在**屏幕最上面**，点「⤓ 从当前位置开始向下扫描」。
     面板底部会实时显示「现在会从这个开始：『…』」，照着它对齐即可。
     起点**只看屏幕，不看鼠标光标停在哪**；想指定别的一条就拖选它里面的几个字。
4. 扫描中随时可以点停止（两个按钮**始终显示**，不扫描时置灰）：
   - **■ 停止扫描并导出已抓到的部分** —— 停在这里，已抓内容照常导出（想要中间一段就用这个）
   - **■ 仅停止（不导出）** —— 单纯中止，**不生成任何文件**（11.1.0 新增）
5. 等它跑完（长对话可能几分钟，中途别切标签页）
6. 扫完看报告里的「补扫：X 处空洞，补回 Y 条」——`Y > 0` 说明这次补回来了，也意味着以前的导出可能是残缺的

导出文件名会自动带状态后缀：`_从指定位置`（局部扫描）、`_中止`（手动停止）；完整扫描无后缀。
局部扫描的扫描报告里会写「起点以上裁掉 X 条」——X 应该是 0 或个位数（只多出屏幕上半屏那点），
如果 X 有十几条，说明起点往上偏了，重新对齐屏幕再扫一次。

## 实测数据

ChatGPT「意识五问梳理」长对话（可滚动总高 86142 px）：

| 版本 | 条数 | 大小 |
|---|---|---|
| 修复前（10.6.0） | 26 | 53843 B |
| 修复后（10.7.0） | 28（真实消息全中） | 60874 B |
| 11.0.0 | 待测 | 待测 |

修复后与人工核对逐条一致，且过滤掉了混入的 `上传文件` 假消息。
11.0.0 只改了「起点选择 + 可停止」，抓取判重逻辑与 10.7.0 一致。

**Grok「人生量变质变辩证法则」（12 条消息 · 全量 DOM 渲染站点）**：

| 版本 | 抓到 | 说明 |
|---|---|---|
| 11.1.7 / 11.1.8 | 0 条 | 视口距离过滤误杀（12 条只剩 1 条） |
| 11.1.9 | **仍然 0 条** | 豁免只加在 Tier 1，被 `collectMessages` 的二次过滤砍回来 |
| **11.2.0** | **12 / 12**（6 user + 6 assistant，顺序严格交替） | ① 多 selector 取并集（合并查询，文档序）② `visibleOpts()` 贯穿所有二次过滤；5 个滚动位置（0/25/50/75/100%）全中 |
11.1.0 只加了「纯停止（不导出）」和停止按钮常显，**抓取与导出主流程一字未改**。
11.1.1 只动了**豆包**（修 `data-message-id` 定位 / 类名判角色 / 锁定虚拟列表容器 / 慢扫 / 抬头加"对话来源"），其它站点行为不变。
11.1.2 把 11.1.1 的四件套**推到全部站点**（豆包不动），并给 `roleRules` 加 `tag` 字段（让 Gemini 自定义元素能判角色）。
11.1.7 用 chrome-bridge 抓真实 DOM 修 Claude / Grok / 千问 selector（不再盲改）。
11.1.9 修 **Grok 全量 DOM 渲染被判"不可见"**：新增 `SITE.fullDomRender`，Grok 豁免"距视口 ±45% 屏高"过滤。
chrome-bridge 实测：改动前 selector 裸命中 12 条但 `__lcxDebug()` `candidates=1`、`final=1`；
改动后 **raw 12 → 可见性 12 → stripNested 12**，角色完美交替。

## 给 Agent / 其它脚本调用

注入后暴露：

```js
window.__lcxSite              // 当前站点配置
window.__lcxToTop()           // 只回顶部，不扫描，返回 Promise<boolean>
window.__cg10Debug()          // 打印全部标题候选，排查文件名取错时使用
window.__lcxScan()            // 完整扫描（Promise）
window.__lcxScan({fromHere:true})  // 从当前滚动位置开始向下扫描
window.__lcxStop()            // 中止当前扫描，已抓到的部分照常导出
window.__lcxStop(false)       // 等价于「仅停止（不导出）」
window.__lcxStopOnly()        // 纯停止：中止且不导出（11.1.0）
window.__lcxInfo()            // { version, running, site, messages }
```

## 出处

由 **WorkBuddy · 小台 (workspace-builder)** 生成。
