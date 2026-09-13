# 站点适配手册

脚本顶部 `SITES` 数组决定"去哪找消息 / 怎么判角色 / 去哪找标题"。
内置配置文件（ChatGPT 已实测，其余为**选择器最佳实践**，用到时请现场核对）：

| id | 域名 | msgSelectors | roleAttr | scroller |
|---|---|---|---|---|
| chatgpt | chatgpt.com | `[data-message-author-role]` | `data-message-author-role` | `right` |
| grok | grok.com | `[data-testid="user-message"]` / `[data-testid="assistant-message"]` | `data-testid` | generic |
| claude | claude.ai | `[data-testid="user-message"]` / `[data-testid="assistant-message"]` | `data-testid` | generic |
| gemini | gemini.google.com | `user-query` / `model-response` | — | generic |
| deepseek | chat.deepseek.com | `[data-message-id]` | — | generic |
| kimi | kimi.com | `div[class*="chat-item"]` | — | generic |
| doubao | doubao.com | `div[class*="message"]` | — | generic |
| qwen | tongyi.aliyun.com / chat.qwen.ai | `div[class*="message"]` | — | generic |
| yuanbao | yuanbao.tencent.com | `div[class*="message"]` | — | generic |
| chatglm | chatglm.cn | `div[class*="message"]` | — | generic |
| generic | 兜底 | 无（走启发式） | — | generic |

## 探测一个新站点（chrome-bridge 一条命令搞定）

```bash
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> "
(function(){
  var out = {};
  // 1) 滚动容器
  var cx=innerWidth/2, cy=innerHeight/2, best=null, br=0;
  document.querySelectorAll('*').forEach(function(el){
    var s=getComputedStyle(el);
    if(!/auto|scroll|overlay/.test(s.overflowY)) return;
    var g=el.scrollHeight-el.clientHeight; if(g<=50) return;
    var r=el.getBoundingClientRect();
    if(r.width<200||r.height<200) return;
    if(cx<r.left||cx>r.right||cy<r.top||cy>r.bottom) return;
    if(g>br){br=g;best=el}
  });
  out.scroller = best ? (best.tagName+'.'+String(best.className).slice(0,60)) : 'window';
  out.scrollRange = br;
  // 2) 带角色属性的节点
  ['data-message-author-role','data-role','data-author','data-testid','data-message-id'].forEach(function(a){
    var n=document.querySelectorAll('['+a+']').length;
    if(n) out[a]=n;
  });
  // 3) 候选消息容器：同父下文本>30字符的兄弟节点最多的一组
  var cands=[].slice.call(document.querySelectorAll('div,li,article'))
    .filter(function(el){return (el.innerText||'').trim().length>30});
  var byParent=new Map();
  cands.forEach(function(el){var p=el.parentElement; if(!p) return;
    var a=byParent.get(p)||[]; a.push(el); byParent.set(p,a)});
  var bestArr=[], bs=0;
  byParent.forEach(function(a,p){ if(a.length<2) return;
    var d=0,q=p; while(q){d++;q=q.parentElement}
    var sc=a.length*100+d; if(sc>bs){bs=sc;bestArr=a}});
  out.turns = bestArr.length;
  out.turnSample = bestArr.slice(0,2).map(function(e){
    return e.tagName+'.'+String(e.className).slice(0,60)});
  out.turnTextSample = bestArr.slice(0,2).map(function(e){
    return (e.innerText||'').trim().slice(0,40)});
  // 4) 标题候选
  out.title = document.title;
  out.activeLink = (document.querySelector('nav a[aria-current], [aria-current=\"page\"]')||{}).innerText || null;
  return JSON.stringify(out,null,1);
})()"
```

拿到结果后填 `SITES`：

- `msgSelectors`：优先用探测到的**带角色属性的**（第 2 步里数量合适的那个）；没有就用第 3 步的 `turnSample` 类名
- `roleAttr` + `roleMap`：有角色属性就填；没有留空，靠 `roleFromText()` 和交替兜底
- `titleSelectors`：侧边栏当前对话的元素（通常 `nav a[href*="/chat/"]` 这类）
- `scroller`：探测到的容器能覆盖视口中心就用 `generic`（默认）；只有右侧独立滚动条才用 `right`

## 通用启发式（没有配置时）

1. 找出所有**可见**且**文本 > 30 字符**的 `div / li / article / section`
2. 按父元素分组，取 `兄弟数量 × 100 + DOM 深度` 分数最高的一组 → 认为是一轮轮对话
3. 角色判定顺序：站点 `roleAttr` → `data-role / data-author / data-testid / aria-label` 关键词 → `className` 关键词 → 容器内查子元素标记 → `unknown`
4. `unknown` 时按 DOM 顺序**一问一答交替**（偶数 user、奇数 assistant）

## 实测踩坑（2026-09-08 grok 实测，通用站点同样适用）

在 grok（28 条长对话）上跑通时踩到 4 个坑，都是**只会在非 ChatGPT 站点暴露**的问题，
新增站点适配时务必对照检查：

### 1. `data-testid` 不一定是唯一 ID —— 会把 28 条压成 2 条

grok 所有用户消息的 `data-testid` 都是 `user-message`、助手都是 `assistant-message`，
是**类别名**而不是唯一 ID。若只用 `stableId` 去重，整条对话会被合并成 2 条。

判定办法：`stableId` 相同但**文本内容不同且绝对位置差很远**（> 一屏的 0.85）
→ 说明是 ID 冲突，必须当作两条消息。（同一条消息流式变长时位置不变，仍复用。）

### 2. 虚拟列表会复用 DOM 元素 —— 会把 28 条压成"一屏的条数"

grok / Claude 的虚拟列表滚动时**复用同一批 DOM 节点**，元素还是那个元素、
内容已经换成另一条消息。若仅用 `WeakMap(元素 → 消息)` 判定"采集过"，
就只会覆盖文本而不新增，最终只剩一屏能显示的条数（grok 实测只剩 6 条）。

判定办法：命中已映射元素后，再检查 **内容变了 且 位置差很远** → 走新建流程。

### 3. 别硬编码 ChatGPT 的属性做"渲染稳定"判断

`waitRendered()` 原本只查 `[data-message-author-role]`（ChatGPT 专有）。
在其他站点上恒为空集合 → 判断永远不成立 → 白等满 1500ms 再返回 false，
10.7 这个关键修复在通用站点上等于完全失效。
现已改为用当前站点的 `msgSelectors` 取指纹，并用 `textContent` 代替 `innerText`
（后者会触发强制重排）。

### 4. 大 DOM 上别对每个元素调 `getComputedStyle` / `innerText`

这类调用会**强制同步重排**。grok 有近 300 个按钮 + 上万节点：

- `findGenericScroller()`：先用便宜的 `scrollHeight/clientHeight` 过滤，再算样式
- `isExpandableButton()`：先做便宜的文本匹配，命中了才做可见性检查
  （`getComputedStyle` + `getClientRects` + `getBoundingClientRect` 三个都触发重排）

另外注意：**裸 `'more'` 不能进"展开"关键词白名单** —— 它会误命中各站点的
`more actions / 更多操作` 菜单按钮，点了只会弹菜单而不是展开内容。

### 5. 扫描前确认标签页在前台！

Chrome 会节流后台标签页：`setTimeout` 被拉长到 ≥1 秒、`requestAnimationFrame` 直接暂停。
脚本大量依赖 `await sleep()`，虚拟列表渲染也依赖 rAF —— 后台运行时会慢 10~50 倍
（grok 实测：前台每屏数秒，后台每屏 3 分钟）。

排查命令：

```bash
node F:/dshdesktopwork/chrome-bridge/cli.mjs eval <match> \
  "(()=>JSON.stringify({visibility:document.visibilityState,hasFocus:document.hasFocus()}))()"
```

看到 `hidden` / `false` 就先把标签页切到前台再跑。

## 校验清单（改完必做）

- [ ] 面板「导出文件名」显示的确实是左侧对话列表里的名字
- [ ] 点「回到真正顶部」后状态显示 `scrollTop = 0`
- [ ] 扫完看报告里的「补扫：X 处空洞，补回 Y 条」
- [ ] 导出的 md 条数 = 页面上肉眼能数到的消息条数
- [ ] 没有 `上传文件` / `upload files` 这类 4 个字的假消息
