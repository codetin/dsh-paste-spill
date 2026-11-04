# 已发送粘贴的"对话内折叠卡"设计

> 日期：2026-11-04
> 状态：已批准，待实施
> 作用面：**仅浏览器半**（`lib/client.js`），宿主半与 session 格式零改动
> 前置：折叠层（4–50 KB → 输入框 chip）与转文件层（≥50 KB → 附件）已存在

---

## 1. 要解决的问题

折叠层把大段粘贴收进输入框里的 chip，但**发送之后它就退场了**：消息正文是那段原文本身，对话里得到一个 `white-space: pre-wrap`、可以上千像素高的气泡。DSH stock 没有任何"长用户消息折叠"。

目标：**同一条消息在对话里显示为一张可展开的卡片**，而模型收到的内容、session 里存的内容、复制/导出的内容一个字都不变。

## 2. 为什么不给消息加标记

最直觉的做法是让 chip 序列化成 `<!--dshsp:…-->原文`，再在渲染时匹配标记。**否决**，因为它把"展示层的事"写进了传输层：

- 标记会进 prompt（模型看见噪音）；
- 标记会进 session log、导出、复制结果；
- 想消除它就得在宿主半改写消息文本，那是"改变会话内容"的语义变更，代价远高于收益。

结论：**只能靠内容自身判断**。这条约束直接决定了下面"按段落 + 字节阈值"的规则，也决定了它的已知误伤（长散文同样会被折叠）。

## 3. 扩展点（已在 0.2.x 客户端 bundle 中核对）

`@deepseek-ai/dsh-client-ui-conversation` 把已发送用户文本交给一组**同步规则 + keyed 槽位**渲染，stock 的 `@文件引用` chip 正是走这条路：

| 环节 | 符号 | 说明 |
|---|---|---|
| 规则注册表 | `ctx.uiConversation.userText.register(rule)` | `match(text, from, context)` 同步；返回 `{ id, start, end, data }` |
| 文档求值 | `UserTextDocument.reconcile` | 逐个 cursor 取"起点最早"的候选；未被任何规则覆盖的区间自动成为 `text` 节点 |
| 节点渲染 | `conversation.user.text.node`（keyed、inline、session 作用域） | `{ node, raw, interactive }`，`raw = text.slice(start, end)` |
| 宿主位置 | `UserStyleBubble → renderFactorySlot("conversation.user.text", { text, facts, interactive, ellipsis })` | 正式消息 `interactive: true`；队列预览/提交回显 `interactive: false` |

`uiConversation` 是 cordis `Service`（名字即 `"uiConversation"`），用 `ctx.get("uiConversation")` 取；**不**加进 `exports.inject`，这样老宿主上服务缺失时自动关闭本能力，而不是让整个插件 stall。

## 4. 识别规则

纯函数 `nextPasteSpan(text, from)`，从 `from` 找**下一段 ≥ 4 KB** 的段落：

1. 段落按空行（`\n\s*\n`）切分。候选区间 = 段落**去掉首尾空白**后的 `[start, end)`：`start` 是段内第一个非空白字符，`end` 是最后一个非空白字符 + 1。首尾空白留在区间外，因此段落之间的空行仍由普通 `text` 节点承载，气泡里的换行结构不变。空白段落不命中。
2. 若 `start < from`（调用发生在段中间），返回 `undefined`——绝不切半段，避免把上一枚 chip 的尾巴包进卡片。
3. 区间 **UTF-8 字节数** ≥ `PASTE_CARD_BYTES`（= `FOLD_BYTES` = 4000，同一常量，两个阈值不会漂移）→ 命中。
4. **单换行也要能切开指令**：真实消息最常见的形状是"指令 + 单个换行 + 粘贴"（`看一下这段日志：\n{…}`）。若段落首行很短（≤120 字符）且以冒号（`:` / `：`）结尾，把这一行**剥在卡外**——它是指令，不是数据；数据行（`"note": "x"`、`k: 1,`）不会以裸冒号收尾，因此不受影响。
5. `data = { bytes, lines }`，由规则一次算好，渲染时不再重算；`lines` 按 `\n` 计数，`bytes` 按 UTF-8。
6. 未命中返回 `undefined`，求值器继续用文本/空白规则兜底。

规则 id 用 `"paste"`（`"text"` 是保留 id，注册时会抛错），`order: 20`（早于 stock 的 `@引用` 30 与空白规则 1000+）：长段内部即使含 `@路径`，也整段进卡片，不再单独出 chip——这是可接受的取舍。

规则失败（抛错）由框架捕获：整条消息退回原文并记录失败，不会吞文本。这条是 stock 行为，不需要我们兜底。

## 5. 卡片 UI（`PasteCard`）

props：`{ node, raw, interactive }` + locale 绑定。

- **`interactive !== true` → 直接输出 `raw` 纯文本。** 这一个判断同时挡住队列预览（它带 `ellipsis` 但 `interactive:false`）、提交回显气泡、只读/子会话视图，保证这些位置外观完全不变。
- 折叠态（默认）：头部一行 `粘贴内容 · 5.9 KB · 128 行`，右侧 `展开`；下方两行预览（等宽字体，`-webkit-line-clamp: 2`）。
- 展开态：卡内 `<pre>` 全文，`max-height: 320px`、可滚动；按钮变 `收起`。
- `复制` 按钮：`navigator.clipboard.writeText(raw)`，成功后 1.5s 内显示 `已复制`；失败显示 `复制失败`，不影响其它交互。
- 展开状态是组件本地 `useState`，不持久化，重载回到折叠态。
- 无障碍：`button[aria-expanded]`；头部文本带 `title`（原文前 120 字符）。
- 样式：沿用本仓库既有的 `data-plugin-css` 注入方式，全部 `dshps-card-*` 类名；**不 key 任何 stock CSS-module 哈希**（仓库已被此坑咬过一次）。
- 不替换气泡背景、不套外层容器：卡片就画在气泡里，保持 stock 的用户消息对齐与间距。

## 6. 不变量

| 不变量 | 保证方式 |
|---|---|
| 模型输入不变 | 规则只在渲染层生效；消息文本从未被修改 |
| session / 导出 / 复制不变 | 同上；复制按钮复制的是 `raw`，即原文 |
| 历史消息同样出卡 | 纯展示层，按会话重放时重新求值 |
| 队列预览 / 回显不变 | `interactive !== true` 直接降级为纯文本 |
| 阈值与 chip 一致 | 共用 `FOLD_BYTES` |
| 老宿主不崩 | `ctx.get("uiConversation")` 缺失即跳过注册 |

## 7. 已知限制

- 判据只有文本本身，**分不清"粘贴"和"打出来的长文本"**：≥4 KB 的段落一律折叠（用户已确认按纯长度判定）。
- 卡片不显示"这段是粘贴来的"这类元信息——没有可靠来源，不编造。
- 段落内部折行/缩进按原样进 `pre`，不做语法高亮或 JSON 树。
- 折叠/展开状态不跨重载保留。

## 8. 测试

浏览器半测试沿用现有 `test/bundle.test.js` 的 ModuleLoader 桩（`react` 用可序列化的元素树 + 可注入的 `useState`）：

**规则（`nextPasteSpan`）**
1. 3 KB 段落不出卡；4 KB 段落出卡（边界：3999 / 4000 / 4001 字节）
2. 多段落：只命中长段，长段前后的短文本仍是 `text` 节点
3. `from` 落在长段中间 → `undefined`
4. CRLF 段落分隔（`\r\n\r\n`）同样识别
5. 中文内容按 UTF-8 字节计（2 个汉字 ≠ 2 字节）
6. 尾部空行不把段尾撑大：段尾空白归属由规则确定并固定下来

**卡片（`PasteCard`）**
7. `interactive:false` → 输出纯文本，不出现按钮
8. 默认折叠：头部含 `KB`/行数，`pre` 只显示两行预览
9. `useState` 注入 `[true, …]`（已展开）→ 输出全量 `raw` 与 `收起`
10. 组装后（apply）在 stub ctx 上注册了规则、`exports.inject` 未新增必填服务

## 9. 交付物

- `lib/client.js`：`PASTE_CARD_BYTES`、`nextPasteSpan`、`pasteTextRule`、`PasteCard`、`installTranscriptCard(ctx)`、样式注入、`__internals` 导出
- `test/bundle.test.js`：上述 10 条
- `README.md` / `README.zh.md`：新增"发送后"一节
- `docs/internals.md`：记录"为什么不用标记 / 为什么 gate 在 `interactive`"
- `BUILD_REV` 递增，便于在 GUI 里确认新构建真的加载了