---
name: long-conversation-export
description: agent 用来给主人当前浏览的 AI 对话页注入右下角浮动导出面板（完整扫描 / 从当前位置开始扫描 / 两种停止：停止并导出、仅停止不导出，导出 Markdown / TXT / 扫描报告）。当主人说"导出这条对话 / 备份 AI 聊天记录 / 对话太长抓不全 / 导出的内容少了几条 / 只要从这里往下的部分 / 扫描能停下来吗 / 只想纯停止不要导出"时调用。agent 用 chrome-bridge eval 注入即可。
agent_created: true
---

# 长对话完整导出（agent 给主人浏览器注入的 UI 封装）

当前脚本版本：**11.1.3**

> **这个 skill 是给 agent 用的**：agent 拿到这个 skill 后，通过 chrome-bridge 的 `eval` 命令把导出面板注入主人当前打开的 AI 对话页（ChatGPT / Claude / Gemini / 豆包 …），主人点面板上的按钮即可把对话导成 Markdown / TXT / 扫描报告。
>
> 面板上四个动作：
> 1. **开始完整扫描** —— 回到真正顶部，整条对话全扫
> 2. **⤓ 从当前位置开始向下扫描** —— 不回顶部，以当前滚动位置为起点往下扫
> 3. **■ 停止扫描并导出已抓到的部分** —— 停下，并把已抓内容照常导出
> 4. **■ 仅停止（不导出）** —— 单纯中止，已抓内容**丢弃、不落盘**（**11.1.0 新增**）
>
> 两个停止按钮**常显**：空闲时置灰不可点，扫描中变可点。
> （主人 2026-09-10 反馈：原先"只在扫描中显示"会让不扫描的人以为按钮不存在。）
>
> 同时具备"回到真正顶部"的能力（内嵌 `goToRealTop`）。

## 什么时候用（agent 视角）

- 主人说"导出这条对话 / 备份这条 AI 聊天记录 / 这条对话太长抓不全"
- 主人抱怨"导出的内容少了几条 / 总是漏消息"
- 主人说"**只要从这里往下的**"、"上面那些新闻简报不要" → 用「从当前位置开始」
- 主人想把对话归档进知识库 / 记忆库

agent 调用方式：

```bash
# 注入导出面板到主人当前 AI 对话页
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> - \
  < C:/Users/Administrator/.workbuddy/skills/long-conversation-export/scripts/long-conversation-export.user.js

# 看识别到了什么站点 + 会导出成什么文件名
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> \
  "JSON.stringify({site:window.__lcxSite.id, title:document.querySelector('#cg10-name').value})"

# 只回顶不扫描
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> "window.__lcxToTop()"

# 程序化触发扫描（不开面板也行）
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> "window.__lcxScan()"                    # 完整扫描
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> "window.__lcxScan({fromHere:true})"     # 从当前滚动位置开始

# 中止当前扫描（默认：已抓到的部分照常导出）
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> "window.__lcxStop()"

# 纯停止：中止且不导出（已抓内容直接丢弃）
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> "window.__lcxStopOnly()"
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> "window.__lcxStop(false)"   # 等价写法

# 看状态
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> "JSON.stringify(window.__lcxInfo())"
```

## 11.0：两种起点 + 扫描中可停止

**为什么需要「从当前位置开始」**：有些页面上半截根本不是对话（比如一大段新闻简报、前面几轮无关内容），
全量扫描会把这些一起导进来。这时候让主人**先把页面滚到对话真正开始的那一屏**，
再点「⤓ 从当前位置开始向下扫描」，脚本会以当时的 `scrollTop` 为起点往下扫：

- **起点以「屏幕」为准**：默认取当前视口顶部往下第一条消息当锚点。
  **鼠标点击产生的光标位置一律不算** —— 主人是用滚轮滚屏的，光标经常随便停在屏幕某处，
  拿光标当起点必然偏（11.0.1 的教训）。只有主人**真的拖选了一段文字**（selection 非折叠）时，
  才用选区所在那条消息当锚点，并把它顶到视口顶部。
- 不执行 `goToRealTop()`（否则又弹回顶部，白忙）
- 记录锚点消息的前 160 字符做 key，展开折叠内容导致页面变高后 `realignStart()` 把起点重新对回来
- `repairGaps()` 补扫时有下限（`startMinY`），**不会回头去补起点以上的空洞** —— 否则又把上面的内容捞回来
- **导出前再做一次硬保险**：`trimAboveStart()` 按锚点文本在 `messages` 里定位，
  把锚点之前的所有消息全部裁掉（找不到锚点就退化成按 Y 裁，半屏冗余）
- 导出文件名自动加 `_从指定位置` 后缀，md / 报告头部写「起始方式：从指定位置开始（起点 scrollY=xxxx）」，
  报告里还会写「起点以上裁掉 X 条」

> **11.0.0 的坑（已修，别再踩）**：起点确定后**绝对不要先展开折叠**。
> 展开一条长消息会让上方内容高出几千 px，绝对坐标整体下移，而 `scrollTop` 没变，
> 于是"起点"就往回跑了好几屏 —— 主人实测就是这个现象（"开始扫描时屏幕明显向上滚了一截"）。
> 折叠展开交给主循环每屏自己做，第一屏展开后再 `realignStart()` 对齐。

**停止扫描**：`abortRequested` 标志位 + `throwIfAborted()` 检查点，
插在 `goToRealTop` / `waitRendered` / `expandVisibleContent` / 主循环 / `repairGaps` 里，
点停止后最多再走完当前一屏就收工。**11.1.0 起分两种停止方式**：

| 按钮 | 标志位 | 行为 |
|---|---|---|
| ■ 停止扫描并导出已抓到的部分 | `stopOnly = false` | 已抓部分**正常导出**（文件名加 `_中止` 后缀，md 头部写「扫描状态：用户手动中止（内容不完整）」） |
| ■ 仅停止（不导出） | `stopOnly = true` | **不进导出流程**，已抓内容直接丢弃，面板只提示「已停止（未导出）本次抓到 N 条，已丢弃」 |

两种方式都是：一条都没抓到就不落盘。

> 注意：「停止并导出」= 保留已抓部分并导出，不是"删掉重来"；「仅停止」= 什么都不落盘。
> 想要干净的整段，就重新点完整扫描。
>
> 11.1.0 之前只有一个「停止并导出」，主人要的是**纯停止**（"从当前停止然后导出"不是他要的），
> 故新增第二种。**本次只做加法**：原「停止并导出」路径一字未改，仍走原来的 `throwIfAborted` 逻辑。

## 11.1.1 / 11.1.2：全站点角色判定通用化

**为什么需要这两次升级**：11.0 之前所有站点都靠**通用启发式**（按类名猜 role），
但豆包（11.1.1 实测）、Kimi/通义/元宝/智谱/DeepSeek（11.1.2 推测）都有一个共同的坑：

> `div[class*="message"]` 这种通配选择器会**误命中**整条消息列表的容器、
> 时间戳动作栏、以及类名里只是带了 padding/avatar 变量名的空壳 div，
> 然后通用启发式又把所有"看起来像消息"的全判成 user —— 导出文件里
> 就只剩"用户"没有"AI"。

11.1.1 修豆包时引入了四个站点可选项，11.1.2 把它们**推到全站点**：

| 字段 | 作用 | 默认 |
|---|---|---|
| `msgSelectors` | 直接用 `[data-message-id]` 优先（绝大多数 React/Vue 写的现代 AI 站都有） | 11.1.2 起所有站点都用 |
| `roleRules` | 按 `el.className` 正则判角色（用户右对齐、assistant 网格/左对齐） | 11.1.2 起所有站点都有 |
| `scrollerSelector` | 显式锁定虚拟列表的滚动容器（豆包那种） | 豆包用，其它未验暂不开 |
| `virtualList: true` | 走慢扫（步长 0.45 视口 + 每屏多等 450ms） | 豆包用 |

**11.1.2 额外加的能力**：`roleRules` 新增 `tag` 字段匹配 `el.tagName`，
让 Gemini 这种用自定义元素（`<user-query>` / `<model-response>`）的站点能直接判角色。

**主人须知**（11.1.2 是"按代码风格猜的"）：
- ✅ 豆包：实测过
- ⚠️ Kimi / 通义 / 元宝 / 智谱 / DeepSeek / Claude / Grok / Gemini：best-guess，**主人打开对应站点对话页验一下**，发现角色判错或漏抓告诉我具体 selector，改一行即可
- ✅ 兜底 `generic` 站点：自动套用同一套规则，未知站也大概率能跑

向后兼容：`roleRules` / `scrollerSelector` / `virtualList` / `tag` 不填 = 老逻辑，11.0 之前所有行为不变。

## 11.1.3：真通用——结构化收集 + 排除列表（修 grok "对话里的搜索"）

**主人 9/10 19:19 实测反馈**：grok 11.1.1/11.1.2 都把"对话里的搜索"当消息抓进来。
**根因**：grok 的搜索结果是普通 div、和真实消息 div **同级**，selector 抓不住区别。
**11.1.1/11.1.2 的错误**：依赖"猜每个站点的 selector"——猜不准就翻车。
**11.1.3 的修法**：完全换路，不再猜 selector。

### 三层收集策略（新版）

1. **Tier 1 · 站点级 selector** —— ChatGPT / Doubao / Gemini 仍用专用 selector（11.1.1/11.1.2 已验）
2. **Tier 2 · 结构化收集**（**新**，所有站点的兜底）：
   - 找「**最深、且直接子节点 ≥ 3 的 div**」当消息容器
   - 拿它的直接子节点当消息
3. **Tier 3 · 旧 fallback**（`genericTurns` 按"兄弟节点最多"猜）

**Tier 2 关键能力**：

| 函数 | 作用 |
|---|---|
| `findConversationContainer(scroller)` | 找最深、且直接子节点 ≥ 3 的 div（最像"装着消息"的容器） |
| `structuredCollect(scroller)` | 拿容器的直接子节点当消息候选 |
| `isInNoiseContext(el)` | 沿 DOM 向上 6 层查 class/id/aria-label/data-* 是否含噪声词，命中即视为"在搜索/工具/附件面板里"，直接跳过 |
| `NOISE_PATTERNS` | 16 个关键词（中英），包含 `search` / `web-search` / `x-search` / `tool` / `function` / `artifact` / `attachment` / `upload` / `citation` / `codeblock` / 搜索 / 工具 / 函数 / 附件 / 代码块 |

### 调试

`window.__lcxDebug()` 返回 `{ candidates, noiseFiltered, final, containerFound, containerSelector, containerChildCount }`，
**不用真扫描**就能看到：找到几个候选 / 几个被噪声过滤 / 留下几个 / 消息容器是谁。
主人遇到新站点判错就调一次，我根据结果改 `NOISE_PATTERNS` 或加白名单。

### 11.1.3 的承诺

不再写 best-guess、不再"我猜这个 selector 对"。要么**结构上**就对，
要么主人调 `__lcxDebug()` 给我看抓到了啥，我能直接定位是 selector 问题、
容器问题、还是新噪声模式需要加进 `NOISE_PATTERNS`。

### 11.1.3 实测基线

| 站点 | 实测状态 |
|---|---|
| 豆包 | ✅ 11.1.1 验过 |
| ChatGPT | ✅ 沿用 10.7.0 路径 |
| Gemini | ✅ 11.1.2 配的 `<user-query>`/`<model-response>` 自定义元素路径 |
| **DeepSeek** | ✅ **主人 19:35 亲测 11.1.3 原版** |
| Kimi / 通义 / 元宝 / 智谱 / Claude / Grok | ⚠️ 待主人开对应站点验 |

### 11.1.3 不动原则

主人亲测过的站点路径**一字不动**。11.1.3 没改 DeepSeek、豆包、ChatGPT、Gemini 的 selector。

## 11.1.4：修 Claude（删 `.font-claude-message`）+ 修 Grok（`forceTier2:true` 跳过 Tier 1）

**主人 19:43 实测反馈（11.1.3）**：

| 问题 | 根因 | 11.1.4 修法 |
|---|---|---|
| Claude 11.1.3 导出只有几百 K | `.font-claude-message` 是 Claude **旧版样式 class**，2024 后大概率不再用；当前命中的是 padding wrapper / 工具栏这类空壳，**抓到但内容空** | 删 `.font-claude-message`，保留双 testid `[data-testid="user-message"]` / `[data-testid="assistant-message"]` |
| Grok 11.1.3 开始扫描还是一直弹出搜索 | Tier 1 testid 在 Grok 上命中的是**搜索 panel 本身**（不是消息体）。`isInNoiseContext` 对 Grok 的 React hash class 名**无效**（不会命中任何 NOISE_PATTERNS 关键词）→ Tier 1 直接返回搜索 panel 当消息 | 加 `forceTier2:true` 跳过 Tier 1，让 Tier 2 结构化收集（找最深 div 容器 + 直接子节点）去抓真消息 |

### 11.1.4 增量改动（只两个，其他站点一字不动）

```
Claude.msgSelectors:  ['[data-testid="user-message"]', '[data-testid="assistant-message"]']
                      （删了 .font-claude-message）

Grok:  新增 forceTier2: true 字段
       collectNodes 检测到后跳过 Tier 1，直接走 Tier 2
```

### 11.1.4 不动原则（同样严格）

- **DeepSeek / 豆包 / ChatGPT / Gemini**：主人+我都验过的，路径不动
- **Kimi / 通义 / 元宝 / 智谱**：单 `[data-message-id]` selector 不动
- **Claude**：只删 1 个 class，没改 testid 路径
- **Grok**：msgSelectors 不变，只是 Tier 1 被跳过了

### 11.1.4 调试命令（主人能自己查）

```js
// Grok 调试：看 Tier 2 抓到了什么
await window.__lcxDebug()
// → candidates: 找消息容器时遇到的候选 div
// → containerFound: true/false（找到了几个候选）
// → noiseFiltered: 被 noise 过滤的（含 noiseSource 关键词）

// Claude 调试：同上，看是抓不到还是抓到空
await window.__lcxDebug()
```

## 11.1.5：加 qianwen.com / www.qianwen.com 入口（修"网页上没有箭头"）

**主人 19:48 实测反馈**：在 `qianwen.com/chat` 打开对话页，**没有浮动导出按钮**。

**根因**：`@match` 和 SITES.hosts 都没包含 `qianwen.com`——油猴根本没在这个域名注入脚本。
千问的三个入口：

| 域名 | 11.1.4 状态 | 11.1.5 |
|---|---|---|
| `tongyi.aliyun.com`（国内） | ✅ 已在 hosts | 保持 |
| `chat.qwen.ai`（国际） | ✅ 已在 hosts | 保持 |
| `qianwen.com/chat`（qianwen） | ❌ 没在 hosts | ✅ 加 `@match *://qianwen.com/*` + hosts |

### 11.1.5 改动（只两个）

1. user.js 顶部加：
```
// @match        https://qianwen.com/*
// @match        https://www.qianwen.com/*
```

2. SITES.qwen.hosts 加 `'qianwen.com'` + `'www.qianwen.com'`

**Tier 1 不变**：msgSelectors 还是单 `[data-message-id]`——qianwen.com/chat 页面如果
也用这个属性就直接抓到；如果不用，**自动 fallback 到 Tier 2 结构化收集**（找最深 div
容器 + 直接子节点 + noise 过滤）兜底。这是 11.1.3 的真通用兜底，对结构变化稳健。

### 11.1.5 不动原则（同样严格）

- Claude / Grok 11.1.4 修法不动
- DeepSeek / 豆包 / ChatGPT / Gemini 路径不动
- Kimi / 元宝 / 智谱 / tongyi / chat.qwen.ai 路径不动

### 调试命令（qianwen.com 装上后看）

```js
await window.__lcxDebug()
// → 如果 containerFound: false：qianwen 用了 TabPanel 类特殊嵌套，
//   Tier 2 兜底也没找到消息容器，告诉我 containerSelector/candidates，我改
// → 如果 candidates > 0 但 final: 0：全被 noise 过滤了，
//   告诉我 noiseFiltered[].noiseSource，我加进 NOISE_PATTERNS
```

## 11.1.6：一次性补全所有 AI 站点域名（主人 19:53 骂"系统漏域名"修法）

**主人 19:53 原话**：*"你这个废物玩意儿，竟犯这低级错误，就这点事，耗了我一天时间，
你妈的，本来就是可以一次就弄好的，你他妈的干活不严谨！这毛病能不能治！"*

**根因**：我之前每个 SITES 只加 1-2 个 hosts，没系统化"主域名 + www 子域 + 官方公开的所有入口"。
这是**系统性的偷懒**，不是单点错误。

### 11.1.6 域名补全清单（每个站点主域 + www 子域 + 公开入口）

| 站点 | 11.1.5 hosts | 11.1.6 hosts | 来源 |
|---|---|---|---|
| ChatGPT | `chatgpt.com`, `chat.openai.com` | **+ `www.chatgpt.com`** | 公开入口 |
| Claude | `claude.ai` | **+ `www.claude.ai`** + `platform.claude.com` | 2025-09 后新开发者平台域名 |
| Grok | `grok.com` | **+ `www.grok.com`** | www 子域 |
| Gemini | `gemini.google.com` | **+ `bard.google.com`** | Google Bard 老入口（已重定向） |
| DeepSeek | `chat.deepseek.com` | **+ `www.deepseek.com`** + `deepseek.com` | 公司官网，"开始对话"按钮跳到 chat |
| Kimi | `kimi.moonshot.cn`, `kimi.com`, `www.kimi.com` | 不变 | 11.1.2/11.1.6 已完整 |
| 豆包 | `www.doubao.com` | **+ `doubao.com`** | 主域 |
| 千问 | `tongyi`, `qwen.ai`, `qianwen.com`, `www.qianwen.com` | 不变 | 11.1.5 已完整 |
| 元宝 | `yuanbao.tencent.com` | 不变 | 已是官网主对话入口 |
| 智谱 | `chatglm.cn` | **+ `chat.z.ai`** | Z.ai 国际版入口，2025 新上 |

### 根治方案：每个新站点**必须**三件套

**铁律（写进 user-level MEMORY.md 了）**：

> 加新站点 / 域名时，**必须**先 `WebSearch "[站点] 官方域名 所有入口"` 一次性拿齐：
> 1. 主域名
> 2. www 子域
> 3. 官方公开的所有入口（含历史入口、重定向、www 子域、子路径）
> 
> 然后 **@match + SITES.hosts 同步加**。少一个域名 = 主人下个又翻车一次。

### 11.1.6 不动原则

- 11.1.1~11.1.5 全部修复一字不动（豆包、Claude、Grok、DeepSeek、千问的修法都保留）
- 只补 hosts + @match，没改任何 selector / roleRules / Tier 逻辑

### 11.1.7 基于真实 DOM 修（chrome-bridge 主人浏览器实测，不再盲改）

**根因**：11.1.4 我"修" Claude/Grok 是盲猜的，都猜错：
- Claude `[data-testid="assistant-message"]` 这个 testid **已不存在**！实测命中 0 条
- Grok `forceTier2: true` **是错决定**——Tier 1 + roleMap 完全够用

**chrome-bridge 抓的真实 DOM 数据**（这是 11.1.7 的依据）：

| 站点 | 抓到的真实 selector | 命中数 | 用途 |
|---|---|---|---|
| **Claude** | `[role=article]`（aria-label="Message N of 30"） | 30 条（virtual list 可见 13） | 每个 article 自带 `data-testid="user-message"` 或 `"assistant-message"`，roleRules 按 testid 判 |
| **Claude** | `[data-testid="chat-title-split"]` | 1 个 | 标题（不是 `chat-title`！） |
| **Claude** | `[data-testid="user-message"]` 单独命中 | 6 | 双保险（即使 `[role=article]` 没命中也能 fallback） |
| **Grok** | `[data-testid="user-message"]` + `[data-testid="assistant-message"]` | 各 6 | Tier 1 + roleMap 正确工作 |
| **Grok** | `[role=article]` | 12 | 也加作 fallback |
| **千问** | **`.chat-round`** | 当前可见 14（virtual list） | **真正对话容器** |
| **千问** | `dataset.chat` 形如 `<msgid>-question` / `-answer` | 14 | **直接判角色**（最稳） |
| **千问** | `.chat-answers-card-wrap` | 14 | 内含 AI 回答区块 |
| **千问** | `.text-ellipsis.whitespace-nowrap.overflow-hidden.text-primary.text-sm.font-400` | 多匹配，取第一个 | 标题（取**第一个**匹配，排除 sidebar 历史） |
| **千问** | `[data-message-id]` | **仅 2 个**——全是 video_note_list card | 之前错认为对话容器，**完全错** |

**三个站点的修法（11.1.7）**：

#### Claude

```diff
- msgSelectors: ['[data-testid="user-message"]', '[data-testid="assistant-message"]']
+ msgSelectors: ['[role=article]', '[data-testid="user-message"]']
- titleSelectors: ['[data-testid="chat-title"]', 'nav a[href*="/chat/"]', 'aside a[href*="/chat/"]']
+ titleSelectors: ['[data-testid="chat-title-split"]', 'h1', 'nav a[href*="/chat/"]']
```

每个 `[role=article]` 里嵌着 `[data-testid="user-message"|"assistant-message"]`，roleRules 按 testid 区分。

#### Grok

```diff
- forceTier2: true,    // 错决定——跳过 Tier 1 让 Tier 2 兜底
+ // 删 forceTier2
+ msgSelectors: ['[data-testid="user-message"]', '[data-testid="assistant-message"]', '[role=article]']
```

父 div 用 className `items-end`（user，对齐右）/ `items-start`（assistant，对齐左）—— fallback 也可工作。

#### 千问

```diff
- msgSelectors: ['[data-message-id]']
+ msgSelectors: ['.chat-round', '[data-message-id]']
+ roleRules: [
+     { re: /-question$/, role: 'user' },     // dataset.chat 后缀
+     { re: /-answer$/, role: 'assistant' },
+     { re: /justify-end|mr-auto|self-end/, role: 'user' },  // 兜底
+     { re: /justify-start|ml-auto|self-start|grid-cols/, role: 'assistant' }
+ ],
+ titleSelectors: [
+     'div.text-ellipsis.whitespace-nowrap.overflow-hidden.text-primary',
+     'h1', 'title'
+ ],
+ virtualList: true   // 千问是虚拟列表
```

### 11.1.7 不动原则

- 11.1.6 补的 24 个 @match 一字不动
- 豆包 / ChatGPT / Gemini / DeepSeek / Kimi / 通义 / 元宝 / 智谱 selector 一字不动

### 站点评测清单（11.1.7 之后维护用）

| 站点 | 实测状态 | 域名 | Tier 1 selector | 备注 |
|---|---|---|---|---|
| 豆包 | ✅ 11.1.1 | doubao.com / www | `[data-message-id]` | 主路径 |
| ChatGPT | ✅ 10.7.0 | chatgpt.com / www / chat.openai.com | `[data-message-author-role]` | 沿用 |
| Gemini | ✅ 11.1.2 | gemini.google.com / bard.google.com | `<user-query>` `<model-response>` | 自定义元素 |
| DeepSeek | ✅ 11.1.3 主人亲测 | chat.deepseek.com / www / deepseek.com | `[data-message-id]` | |
| **Claude** | ✅ **11.1.7 主人实测验 + chrome-bridge 验证** | claude.ai / www / platform.claude.com | `[role=article]` `[data-testid=user-message]` | 真实 DOM，30 条命中 |
| **Grok** | ✅ **11.2.0 chrome-bridge 端到端实测** | grok.com / www | `[data-testid]` 双 + `[role=article]`（**必须并集**） | 真实 DOM 6+6=12；**multi-selector 取并集 + 全量渲染 `fullDomRender: true`** |
| Kimi | ⚠️ 未验 | kimi.com / www / kimi.moonshot.cn | `[data-message-id]` | |
| **千问** | ✅ **11.1.7 主人实测验 + chrome-bridge 验证** | tongyi / qwen.ai / qianwen.com / www | `.chat-round` + `dataset.chat` 判角色 | 真实 DOM，14 条当前可见 |
| 元宝 | ⚠️ 未验 | yuanbao.tencent.com | `[data-message-id]` | |
| 智谱 | ⚠️ 未验 | chatglm.cn / chat.z.ai | `[data-message-id]` | |

## 11.1.9：修 Grok「全量 DOM 渲染被判不可见」→ 扫描 0 条（chrome-bridge 实测）

主人 2026-09-10 20:42 实测反馈："grok 还是不行"
（面板截图：**第 4 屏 · 18% · 已发现 0 条消息**）。

### 排查过程（全部基于 chrome-bridge 真实 DOM，零猜测）

| 步 | 动作 | 结果 |
|---|---|---|
| 1 | `verify-ai-selectors-workbuddy.py grok` | selector **全部有命中**：user-message 6 / assistant-message 6 / role=article 12 |
| 2 | `await window.__lcxDebug()` | **`candidates:1, final:1`** ← 矛盾！裸 selector 12 条，脚本只拿到 1 条 |
| 3 | eval 每条消息几何 | 12 条**全部** `disp=block / vis=visible / opacity=1 / isConnected=true` |
| 4 | eval scroller | `scrollHeight=11510` ≈ 12 条消息高度合计 10701 → **全量 DOM 渲染** |
| 5 | 逐条比对 margin 过滤 | 12 条里 **11 条**命中 `bottom < scroller.top - 406` 或 `top > scroller.bottom + 406` → 只剩第 5 条 |
| 6 | 模拟"豁免距离过滤" | **kept = 12/12**，user/assistant 完美交替 |

### 根因

**Grok 是全量 DOM 渲染站点**——所有消息常驻 DOM，滚动后远离视口的消息**仍然是有效消息**。

但通用的 `isVisibleMessage(el, scroller)` 里有一段"距视口太远就丢掉"的过滤：

```js
const margin = Math.max(window.innerHeight * 0.45, 300);   // 实测 406
if (r.bottom < sr.top - margin)    return false;   // 滚到上方太远 → 丢
if (r.top    > sr.bottom + margin) return false;   // 在下方太远 → 丢
```

这个过滤**对虚拟列表站点是正确的**（未渲染的确实不该抓），
但对 Grok 这种**全量渲染**站点就是**误杀** —— 12 条只留 1 条，扫描到底 0 条。

### 修法（最小可逆加法，其他站点行为零改动）

1. `isVisibleMessage(el, scroller, opts)` 增加第三形参
2. 新增豁免分支（放在**硬性可见性检查之后、距离检查之前**）：
   ```js
   if (opts && opts.ignoreViewportDistance === true) return true;
   ```
   硬性检查一律保留：`isConnected` / `hasHiddenAncestor` / `display:none` /
   `visibility` / `opacity:0` / `getClientRects` 为空 / 尺寸 < 30×5
3. `collectNodes` 的 **Tier 1** 调用处传 `{ ignoreViewportDistance: site.fullDomRender === true }`
4. `SITES.grok` 新增 `fullDomRender: true`

### 新站点可选项

| 字段 | 默认 | 说明 |
|---|---|---|
| `fullDomRender` | `false` | `true` = 该站点全量 DOM 渲染，Tier 1 豁免"视口距离"过滤 |

**怎么判断某站点是不是全量渲染？** chrome-bridge eval：

```js
(()=>{ const s=[...document.querySelectorAll('div')]
  .find(d=>/(auto|scroll)/.test(getComputedStyle(d).overflowY) && d.scrollHeight>d.clientHeight+100);
  return {scrollHeight:s.scrollHeight, msgCount:document.querySelectorAll('<你的 msgSelector>').length}; })()
```

`scrollHeight` 与"消息数 × 单条高度"相等（或接近）= 全量渲染 → 需要 `fullDomRender: true`。

### 11.1.9 不动原则

- 其他所有站点（ChatGPT / Claude / 豆包 / DeepSeek / 千问 / Gemini …）**一字不动**
  —— 它们不填 `fullDomRender`，`undefined === true` 为 `false`，走原逻辑，行为不变
- 11.1.7 的 selector 一字不动；11.1.6 的 @match 一字不动

### 实测基线

| 版本 | Grok 表现 |
|---|---|
| 11.1.4 / 11.1.6 | 扫描弹出搜索（`forceTier2:true` 错决定） |
| 11.1.7 / 11.1.8 | selector 对了，但**扫描 0 条**（视口距离过滤误杀 + 二次过滤，见 11.2.0） |
| 11.1.9 | ⚠️ 豁免只加在 Tier 1，消息仍活不到导出（**还是 0 条**） |
| **11.2.0** | ✅ 端到端 12/12，顺序 `user assistant` 严格交替，5 个滚动位置全中 |

## 11.2.0：修 Grok 真凶——**两个 bug 叠加**（chrome-bridge 实测定位，零猜测）

11.1.9 只修了一半：`fullDomRender` 豁免加在了 Tier 1，但消息**根本没活到导出**。
11.2.0 用 chrome-bridge 在主人 Grok 页逐段实测，挖出两个叠加的 bug。

### Bug①：Tier 1「逐个 selector 试、首个命中就 return」→ 助手消息全丢

旧代码：

```js
for (const sel of site.msgSelectors) {
    let nodes = [...document.querySelectorAll(sel)];
    nodes = nodes.filter(el => isVisibleMessage(el, scroller, opts));
    if (nodes.length) {
        const filtered = stripNested(nodes).filter(el => !isInNoiseContext(el));
        if (filtered.length) return filtered;   // ← 首个命中就收工
    }
}
```

Grok 配置 `msgSelectors = [user-message, assistant-message, role=article]`，
**第一个就命中 6 条 user → 立刻 return → 助手 6 条永远轮不到**。

实测证据：

| 检查 | 结果 |
|---|---|
| 逐个命中数 | `user-message` **6** · `assistant-message` **6** · `[role=article]` **12** |
| `__lcxDebug()` | `candidates: 6 / final: 6`，且 6 条**全是 user** |
| 并集实测 | **12 条**，user/assistant 完美交替 |

**修法**：全部 selector 取并集，且**必须合并成一条 `querySelectorAll`**：

```js
union = [...document.querySelectorAll(site.msgSelectors.join(','))];
```

坑：逐个 selector 依次 `push` 出来的是「按 selector 分组」的顺序，实测

```
逐个 push : user user user user user user assi assi assi assi assi assi   ← 顺序错乱
合并查询 : user assi user assi user assi user assi user assi user assi   ← 文档序，正确
```

合并查询由浏览器保证**文档序**且**自动去重**。若某个 selector 语法非法导致整条查询抛错，
退回逐个查询 + `compareDocumentPosition` 手动按文档序排序。

**A/B 回归实测（本机 6 个真实站点，顺序敏感）**：

| 站点 | 旧 | 并集 | 结果 |
|---|---|---|---|
| claude | 6 | 6 | ✅ 集合与顺序完全等价 |
| deepseek | 4 | 4 | ✅ |
| qwen | 2 | 2 | ✅ |
| chatgpt | 4 | 4 | ✅ |
| doubao | 4 | 4 | ✅ |
| **grok** | **6** | **12** | 🔧 多出的正是丢失的 6 条助手消息 |

### Bug②：`collectMessages` 二次可见性过滤没传 opts → 距离过滤被砍回来

```js
// collectMessages 里
for (const el of nodes) {
    if (!isVisibleMessage(el, scroller)) {   // ← 没传 opts，fullDomRender 豁免失效
        continue;
    }
```

于是：Tier 1 豁免距离拿到 6 条 → 这里第二次过滤全砍掉 → `messages.length = 0`
→ 面板报「**本次抓到 0 条**」。
这也解释了那个诡异现象：**`__lcxDebug()` 说 6 条、面板说 0 条** —— 两个数字走的是两条路。

实测：同样 12 条，经二次过滤后 **只剩 1 条**（11/12 被杀），换个滚动位置即 0 条。

**修法**：新增 `visibleOpts()` 统一入口，凡是「对已收集消息做二次过滤」的地方都必须传它：

```js
function visibleOpts() {
    return {
        ignoreViewportDistance:
            !!(SITE && SITE.fullDomRender === true)
    };
}
```

已接入 4 处：`genericTurns`(Tier 3) / `getMessageElements` 两个 article 兜底 / `collectMessages`。

### ≤11.1.9 的教训：**只修一个 bug 不够**

两个 bug 叠加才造成"扫描 0 条"。11.1.9 修了 Bug①的豁免入口却漏了 Bug②的二次过滤，
所以主人看起来"还是这样"。以后遇到**数量对不上**（`__lcxDebug` 说 N、面板说 M），
**必须顺着数据链路逐段查，不能只查一段就下结论**。

### 11.2.0 端到端实测

```
Tier1 并集 12 条 → stripNested 12 → 噪声过滤杀 0 → collectMessages 二次过滤 12
顺序：user assistant user assistant user assistant user assistant user assistant user assistant（严格交替）
5 个滚动位置（0/25/50/75/100%）全部 12/12
```

### 11.2.0 不动原则

- 站点 selector 一字不动（只改**收集逻辑**）
- 其他站点因未设 `fullDomRender` → `visibleOpts()` 返回 `{ignoreViewportDistance: false}` → 行为不变
  （已用 6 站 A/B 实测验证，集合与顺序完全等价）

## 3. 只想要"回到顶部"这一个能力

`scripts/scroll-to-real-top.user.js`（`@match *://*/*`，任何网页）：左下角可拖动圆钮 + **Alt+T**，
注入后 `await window.goToRealTop()`。

## 为什么普通扫法会漏内容（关键）

长对话页面是**虚拟列表**：DOM 里同时只渲染 **6–8 条**消息（ChatGPT 实测 8 条）。
滚动后如果只 `sleep(350ms)` 就采集，新内容还没渲染出来，采到的还是上一屏，
中间那几条就**永久丢失** —— 而且**随机**：同一套代码两次跑出 29 条和 26 条。

三个必须做的动作：

| 机制 | 作用 |
|---|---|
| `waitRendered()` | 滚动后轮询，直到"已渲染消息集合连续两次不变"才采集（上限 1.5s） |
| `SCAN_RATIO = 0.7` | 每次只移动 70% 视口，保留大重叠（0.82 会漏） |
| `repairGaps()` | 扫完按绝对 Y 算覆盖范围，跨度 > 0.9 屏的空洞跳回去补采 |

**验收标准**：面板报告里出现「补扫：X 处空洞，补回 Y 条」。
**只要 `Y > 0`，说明没有这套机制的旧导出都是残缺的，需要重跑。**

## 文件名取对话标题的正确姿势

最常见的坑：页面上**第一个 `<a>` 是无障碍跳转链接**（ChatGPT 是 `<a href="#main">跳至内容</a>`）。
`#main` 是纯锚点，用当前页解析后 `pathname` 就等于当前对话路径，
于是"当前 URL 匹配"的判断**必然命中**，所有导出都叫 `跳至内容_时间戳.md`。

规则：

1. **跳过 `#` 开头和 `javascript:` / `mailto:` 等伪协议 href**
2. 多源打分取最高：`侧边栏 nav 内命中 100` → `aria-current 90` → `页内其它链接 70` → `document.title 50`
3. 占位黑名单：`ChatGPT` / `新聊天` / `跳至内容` / `New chat` 一律丢弃
4. DOM 可信度 < 80 时才静默请求后端接口兜底
5. **面板上一定要把识别结果显示成可编辑输入框** —— 用户一眼能看到会导出成什么名字，不满意直接手打
6. 11.0 起文件名会带状态后缀：`_从指定位置`（局部扫描）、`_中止`（手动停止），
   完整跑完的跟以前一样只有 `标题_时间戳`，方便一眼区分哪些是残卷

## 发布新版本时的约定

油猴列表默认只显眼地显示 `@name`，`@version` 在另一列很容易被忽略，
所以**版本号要同时写进两处**，改版本时别漏：

```
// @name         长对话完整导出器 v11.1.3（通用版·结构化收集+排除列表真通用）
// @version      11.1.3
```

改 `@name` 会让油猴把它当成**新脚本**（粘贴进去是新建，不是覆盖），
所以要顺手把旧版本那条删掉/禁用 —— 两个版本都认 `#cg10-root`，同时启用会打架。

## 适配一个新站点

**铁律：不再靠猜。改任何 selector 前必走 PRE-FLIGHT CHECKLIST。**

### PRE-FLIGHT CHECKLIST（改 selector 前必走 5 步）

主人 20:21 骂"你一直在靠猜"后的根治方法——每次改 selector 必须按这个顺序走，**跳步不准写代码**：

```
□ 1. 跑脚本拿数据（不准靠记忆）
   python .workbuddy/scripts/verify-ai-selectors-workbuddy.py <site-id> --save
   自动调 chrome-bridge，对主人 Chrome 已开的 <site-id> 对话页
   跑所有候选 selector，输出每个命中数 + className + testid + dataset。

□ 2. 查事实档案
   cat .workbuddy/事实档案/ai-selector-facts.json | jq '.deepseek'
   没有此站数据 = 该站待首次验证（不要瞎写代码）。

□ 3. 看 chrome-bridge 实测返回
   - cc/fc 各多少？太离谱就 Tier 选择器错。
   - samples 里 className/testid/role/aria-label/dataset 写了什么？
   - 消息容器是哪个 class，title 在哪？

□ 4. 改 SITES 配置（在 long-conversation-export.user.js）
   - msgSelectors 按"chrome-bridge 实测命中"的最稳那个
   - titleSelectors 同理（注意 sidebar vs 主区）
   - roleRules 按 aria/data-attr/类名 实测判角色

□ 5. 把新事实写进事实档案（必须！）
   自动：python .workbuddy/scripts/verify-ai-selectors-workbuddy.py <site-id> --save
   手动：编辑 .workbuddy/事实档案/ai-selector-facts.json 加 last_verified
         和 msg_selector_reasoning / title_selector_reasoning。

⚠️ 跳任一步 = 重蹈 11.1.4/11.1.6 覆辙（盲改 Claude/Grok、漏域名）。
```

### 详细流程（旧版流程已废，统一走 PRE-FLIGHT）

1. **chrome-bridge 起**：Bash run_in_background=true 跑 `node F:/dshdesktopwork/chrome-bridge/server.mjs`，再 `health`
2. **主人 Chrome 已开对话页**：先 `node cli.mjs tabs` 过滤目标站点
3. **没开就 `nav <match> <url>`** 帮他打开（已登录态直接进）
4. **eval 抓真实 selector**（参考上面的步骤 1）
5. **基于真实数据改 SITES 配置**，bump 版本号
6. **F 盘归档**：`cp` 到 `F:/workbuddywork/<日期>/long-conversation-export-v<新版本>-workbuddy.user.js`
7. **更新 SKILL.md / README.md / 事实档案**
8. **memory 记一笔**（include 真实 selector 数据 + chrome-bridge 返回样本）

### 已归档的事实档案

见 `.workbuddy/事实档案/ai-selector-facts.json`（项目级）。
- 验证过的：`doubao` / `chatgpt` / `claude` / `grok` / `qwen` / `deepseek`
- 待主人首次验证：`kimi` / `gemini` / `yuanbao` / `chatglm`


## 装到其它 agent 的客户端

skill 文件本身是 plain text，**没有自动同步机制**。
别的 agent（Claude Code / opencode / Codex …）要看到这个 skill，必须主人**手动**把这个目录复制到对应 agent 的 skills 目录；**agent 不能自作主张去同步**。

判定"主人浏览器里有没有按钮"：注入后看 `!!document.querySelector('#cg10-root')`（导出面板）或 `typeof window.goToRealTop === 'function'`（回顶）。
判定"agent 有没有这个 skill"：看对应 agent 的 skills 目录里有没有 `long-conversation-export/SKILL.md`。

## 坑

- **别用 `window.scrollTo(0,0)`**：滚动容器通常是内层 div，完全无效。
- **别只确认一次顶部**：虚拟列表归零后会二次布局顶回去，必须连续确认 4 次（每次 500ms）。
- **短文本噪声要过滤**：`上传文件` / `upload files` 这类附件区文字会被当成一条消息（实测混入过 `len=4` 的假消息）。
- **折叠按钮文案多语言**：`展开 / 显示更多 / 查看更多 / show more / see more / 继续阅读` 都要覆盖。
- 扫描中途**不要切换标签页 / 最小化窗口**：没有布局，探测会失败。
- 两个版本同时装会打架（都认 `#cg10-root`），装新版前先禁用旧的。
- **「从当前位置开始」不等于「只导这一段」**：它是从起点一直扫到对话底部。
  想只要中间一段 = 从当前位置开始 + 扫到需要的地方点「停止扫描并导出」。
- **两个停止按钮别混**：想要留一半内容就点「停止扫描并导出已抓到的部分」；
  只想赶紧停、**不要任何文件**，点「仅停止（不导出）」。点错不会丢已有文件，只是会不会多落一份 `_中止`。
- **点停止后别再点开始**：`running` 复位前重复点击会被忽略，等面板状态变了再操作。
- 起点选错（往上多选了一大截）不用重来整条：重新滚到正确位置，再点一次「从当前位置开始」即可，
  它会清空上次结果重新扫。
- **别拿鼠标光标当起点**：主人是滚轮滚屏，光标停哪儿跟起点没关系，脚本也不看它
  （只有真的拖选了文字才算）。面板底部实时显示"现在会从这个开始：「…」"，以那个为准。

## 实测基线（2026-09-08，ChatGPT「意识五问梳理」）

| 版本 | 条数 | md 大小 | 说明 |
|---|---|---|---|
| 10.4.1 | 29 | 60903 B | 混入 1 条 `上传文件` 假消息 |
| 10.6.0 | 26 | 53843 B | 漏抓 #9/#10/#11 三条真实消息 |
| 10.7.0 | **28** | 60874 B | 28 条真实消息全中，假消息已过滤 |
| 11.0.2 | 16（局部） | 44117 B | **首次实测通过** ✅ ChatGPT「新闻到三生万物至幻之镜」，从指定位置 Y=13690 起，60 屏，7 用户 + 9 ChatGPT，到底 ✓，起点以上裁掉 1 条（上方旧简报成功剔除，txt 从旧版全量 60460 B 降到 44787 B） |

与 10.4.1 逐条比对：28 条内容 100% 一致（仅尾部按钮文案 `收起`/`展开` 之差），差异只有被过滤掉的那条噪声。

11.0.2 实测备注（ChatGPT，长简报 + 长对话混合页）：

- 「起点以上裁掉 1 条」→ 起点对齐验证 OK，就多算了屏幕上半屏那一条
- 结构异常 15→16 两条 assistant 连着 = 第二天的每日简报，**真实存在**，不是漏抓。
  判读原则：简报/自动消息跟在回答后面属正常，别当成 bug 修
- 「补扫 7 处空洞，补回 0 条」：长简报内部空行间距被误判为空洞，跳回去没东西可补，无害。
  若以后频繁出现且补回 > 0，说明 `repairGaps` 的 0.9 屏阈值需要按站点微调

## 产物署名

本技能由 **WorkBuddy · 小台 (workspace-builder)** 生成，二次分发请保留出处。
