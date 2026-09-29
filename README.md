# AI长对话完整导出器

> 把任意 AI 长对话**一次性抓全**，导出为 Markdown / TXT / 扫描报告。
> 专治长对话「导出不全、漏抓、文件名错、没法只导其中一段」等痛点。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Userscript](https://img.shields.io/badge/Tampermonkey-ready-green.svg)](scripts/long-conversation-export.user.js)

## 为什么需要它

- 绝大多数 AI 站点**没有「导出单条对话」**，只有「导出全部数据」——成百上千条对话混在一个 JSON 里，根本没法用。
- 长对话是**虚拟列表**，DOM 同时只渲染少量消息。普通滚动采集会**随机漏抓**：同一段对话两次导出可能一次 29 条、一次 26 条。
- 导出文件名常常取错（比如取到无障碍跳转链接，所有文件都叫「跳至内容」）。

## 核心能力

- **真实顶部**：反复压 `scrollTop=0` 并连续确认多次，解决 Home 键到不了顶的问题。
- **全量抓取**：逐屏扫描 + 渲染稳定检测 + 滚动到底后回头补扫空洞，确保一条不漏。
- **折叠展开**：自动点开「展开 / 显示更多 / show more / 继续阅读」等多语言折叠。
- **三重去重**：稳定 ID + 文本哈希 + 绝对 Y 位置，避免重复或错位。
- **正确命名**：多源打分取侧边栏当前对话名，文件名在面板上可见、可改。
- **局部扫描**：从当前屏幕最上面那条消息开始向下扫，只导需要的那段。
- **随时停止**：两个停止按钮常显——「停止并导出已抓部分」/「仅停止（不导出）」。
- **独立回顶**：「回到真正顶部」按钮，不触发扫描，也可被其它脚本调用。
- **全站点通用**：结构化收集 + 排除列表，覆盖主流 AI 站点；新站点可参考 `references/site-profiles.md` 适配。

## 支持的 AI 站点

ChatGPT · Claude · Gemini · DeepSeek · Kimi · 豆包 · 通义千言 · 元宝 · 智谱清言 ·
以及其它自动走通用启发式算法的站点。

## 安装

需要浏览器装 **Tampermonkey（篡改猴）** 等油猴管理器。

1. 打开脚本管理器 → 新建脚本
2. 全选删掉默认内容，粘贴 `scripts/long-conversation-export.user.js` 的全部内容
3. 保存（Ctrl+S）

> 只想用「回到顶部」的话，可单独装 `scripts/scroll-to-real-top.user.js`（任意网页通用，左下角可拖动圆钮 + `Alt+T`）。

## 使用

1. 打开要导出的那条对话。
2. 右下角浮窗 → 确认「导出文件名」是否正确（可直接改）。
3. 选择起点：
   - 整条都要 → 点「开始完整扫描」（会先回到真正顶部）。
   - 只要其中一段 → 用滚轮把页面滚到对话真正开始的那一屏，让起点那条消息处在**屏幕最上面**，点「⤓ 从当前位置开始向下扫描」。
4. 扫描中随时可停止（两个按钮始终显示，空闲时置灰）。
5. 等它跑完（长对话可能几分钟，中途别切标签页）。
6. 扫完看报告里的「补扫：X 处空洞，补回 Y 条」——`Y > 0` 说明补回了之前可能漏掉的内容。

## 给 Agent / 开发者

注入后脚本在 `window` 上暴露以下接口，可供其它脚本或 Agent 驱动：

```js
window.__lcxToTop()                  // 只回顶部，不扫描，返回 Promise<boolean>
window.__lcxScan()                   // 完整扫描（Promise）
window.__lcxScan({ fromHere: true }) // 从当前滚动位置开始向下扫描
window.__lcxStop()                   // 中止并导出已抓到的部分
window.__lcxStopOnly()               // 纯停止：中止且不导出
window.__lcxInfo()                   // { version, running, site, messages }
```

## 双形态

- **油猴脚本**：`scripts/long-conversation-export.user.js`，装 Tampermonkey 即可用。
- **WorkBuddy 技能**：同一能力封装为 WorkBuddy 技能，在开放平台市场可搜到，由 Agent 通过 `chrome-bridge` 注入使用。

## 许可证

[MIT](LICENSE)
