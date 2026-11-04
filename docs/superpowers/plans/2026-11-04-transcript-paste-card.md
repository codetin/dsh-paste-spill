# Transcript Paste Card Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让已发送消息里 ≥4 KB 的段落渲染成可展开的"粘贴内容"卡片，模型输入、session、复制/导出全不变。

**Architecture:** 浏览器半只加三样东西：一个同步 user-text 规则（`nextPasteSpan` + `pasteTextRule`）、一个 keyed 槽位渲染器（`PasteCard`）、一段 `dshps-card-*` 样式。规则通过 `ctx.get("uiConversation").userText.register` 注册，渲染器注册到 stock 的 `conversation.user.text.node` 槽位（与 `@文件引用` chip 同一机制）。宿主半、session 格式、`package.json` 依赖都不动。

**Tech Stack:** 浏览器半是 `window.__ModuleLoader__` 工厂 bundle（纯 JS + `require("react")`），测试是 `node --test` + 手工 ModuleLoader/react 桩。

## Global Constraints

- 只改浏览器半 `lib/client.js`（+ 测试 + 文档），不改宿主半与 session 格式。
- 阈值复用 `FOLD_BYTES`（4000）；卡片常量 `PASTE_CARD_BYTES = FOLD_BYTES`。
- 规则 id 必须是 `"paste"`（`"text"` 保留）；候选区间必须满足 `from <= start < end <= text.length`。
- `interactive !== true` 时必须原样输出 `raw` 纯文本（队列预览、提交回显、只读视图不变）。
- 不 key 任何 stock CSS-module 哈希类名；样式类名一律 `dshps-card-*`。
- 不改变消息文本：卡片只在渲染层替换显示。
- 每个任务结束 `node --test` 必须全绿。

---

### Task 1: 段落扫描 + user-text 规则

**Files:**
- Modify: `lib/client.js`（常量区、`utf8Bytes` 附近、`exports.__internals`）
- Test: `test/bundle.test.js`（末尾追加）

**Interfaces:**
- Produces: `nextPasteSpan(text, from) -> { start, end, bytes, lines } | undefined`；`pasteTextRule = { id: "paste", order: 20, match(text, from) }`；`PASTE_CARD_BYTES`
- Consumes: 已有的 `utf8Bytes`

- [ ] **Step 1: 写失败测试**

```js
test("nextPasteSpan ignores paragraphs under 4 KB and returns the first long one", () => {
  const { nextPasteSpan } = loadBundle().exports.__internals;
  const short = "x".repeat(3999);
  const long = "y".repeat(4000);
  assert.equal(nextPasteSpan(short, 0), undefined, "3999 bytes must not card");
  const span = nextPasteSpan(long, 0);
  assert.ok(span, "4000 bytes must card");
  assert.equal(span.start, 0);
  assert.equal(span.end, 4000);
  assert.equal(span.bytes, 4000);
  assert.equal(span.lines, 1);
});

test("nextPasteSpan only cards the long paragraph and keeps its neighbours", () => {
  const { nextPasteSpan } = loadBundle().exports.__internals;
  const text = `前言两句\n\n${"z".repeat(4001)}\n\n后记`;
  const span = nextPasteSpan(text, 0);
  assert.equal(span.start, text.indexOf("z"));
  assert.equal(span.end, text.indexOf("z") + 4001);
  // A cursor inside the long paragraph must not produce a half card.
  assert.equal(nextPasteSpan(text, span.start + 10), undefined);
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `node --test test/bundle.test.js`
Expected: FAIL — `nextPasteSpan is not a function`

- [ ] **Step 3: 实现**

在 `utf8Bytes` 之后加入：

```js
/** A sent paragraph at least this big renders as a collapsed card in the transcript. */
const PASTE_CARD_BYTES = FOLD_BYTES;

/**
 * The next pasted-looking paragraph at or after `from`, measured in UTF-8 bytes.
 *
 * Paragraphs are separated by a blank line. The span excludes the paragraph's own
 * leading and trailing whitespace, so the blank lines between paragraphs stay ordinary
 * text nodes and the bubble keeps its line structure. A cursor that lands INSIDE a
 * paragraph yields nothing: a card there would swallow the tail of whatever node ended
 * at that cursor.
 */
function nextPasteSpan(text, from) {
  const source = typeof text === "string" ? text : "";
  if (source.length === 0) return undefined;
  const cursor = typeof from === "number" && from > 0 ? Math.floor(from) : 0;
  const separator = /\n\s*\n/g;
  let lineStart = 0;
  while (lineStart <= source.length) {
    separator.lastIndex = lineStart;
    const match = separator.exec(source);
    const rawEnd = match === null ? source.length : match.index;
    let begin = lineStart;
    while (begin < rawEnd && /\s/u.test(source.charAt(begin))) begin += 1;
    let end = rawEnd;
    while (end > begin && /\s/u.test(source.charAt(end - 1))) end -= 1;
    if (end > begin && begin >= cursor) {
      const slice = source.slice(begin, end);
      const bytes = utf8Bytes(slice);
      if (bytes >= PASTE_CARD_BYTES) {
        return { start: begin, end, bytes, lines: countLines(slice) };
      }
    }
    if (match === null) break;
    lineStart = match.index + match[0].length;
  }
  return undefined;
}

/** Offer a long paragraph as a card; every other range stays plain text. */
const pasteTextRule = {
  id: "paste",
  order: 20,
  match(text, from) {
    const span = nextPasteSpan(text, from);
    if (span === undefined) return undefined;
    return {
      id: "paste",
      start: span.start,
      end: span.end,
      data: { bytes: span.bytes, lines: span.lines },
    };
  },
};
```

并在 `exports.__internals` 中加入 `PASTE_CARD_BYTES, nextPasteSpan, pasteTextRule`。

- [ ] **Step 4: 运行测试确认通过**

Run: `node --test test/bundle.test.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add lib/client.js test/bundle.test.js
git commit -m "Detect long sent paragraphs for the transcript card"
```

### Task 2: 卡片渲染器与注册

**Files:**
- Modify: `lib/client.js`（locale 字典、`PasteCard` 组件、apply 内注册、样式字符串、`exports.__internals`）
- Test: `test/bundle.test.js`

**Interfaces:**
- Consumes: Task 1 的 `pasteTextRule`
- Produces: `PasteCard({ node, raw, interactive, t })`；`installTranscriptCard(ctx)`；locale keys `pastedTitle/pastedMeta/pastedShow/pastedHide/pastedCopy/pastedCopied/pastedCopyFailed`

- [ ] **Step 1: 写失败测试**

```js
test("PasteCard renders plain text when it is not interactive", () => {
  const { PasteCard } = loadBundle().exports.__internals;
  const raw = "q".repeat(4000);
  const tree = PasteCard({ node: { data: { bytes: 4000, lines: 1 } }, raw, interactive: false, t: (key) => key });
  assert.equal(tree.props.className, "dshps-card-plain");
  assert.equal(tree.children[0], raw);
});

test("PasteCard is collapsed by default and expanded when the state says so", () => {
  const collapsed = loadBundle().exports.__internals.PasteCard;
  const raw = `{\n${"a".repeat(4100)}\n}`;
  const node = { data: { bytes: 4107, lines: 3 } };
  const t = (key, params) => (params === undefined ? key : `${key}:${params.size}:${params.lines}`);
  const tree = collapsed({ node, raw, interactive: true, t });
  assert.equal(tree.props.className, "dshps-card");
  const full = findDeep(tree, "data-paste-card-full");
  assert.equal(full, null, "collapsed card must not render the full body");
  assert.ok(findDeep(tree, "data-paste-card-preview"), "collapsed card shows a preview");
  assert.ok(findDeep(tree, "data-paste-card-toggle"), "and a toggle");

  const expanded = loadBundle({ useState: () => [true, () => {}] }).exports.__internals.PasteCard({ node, raw, interactive: true, t });
  const body = findDeep(expanded, "data-paste-card-full");
  assert.ok(body, "expanded card renders the full text");
  assert.equal(body.children[0], raw);
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `node --test test/bundle.test.js`
Expected: FAIL — `PasteCard is not a function`

- [ ] **Step 3: 实现组件、样式、locale 与注册**

locale 字典补：

```js
pastedTitle: "粘贴内容",
pastedMeta: "{size} · {lines} 行",
pastedShow: "展开",
pastedHide: "收起",
pastedCopy: "复制",
pastedCopied: "已复制",
pastedCopyFailed: "复制失败",
```

英文对应 `Pasted text / {size} · {lines} lines / Show / Hide / Copy / Copied / Copy failed`。

组件（放在 `PasteFoldChip` 旁边）：

```js
/**
 * One sent paragraph rendered as a collapsed card.
 *
 * Only the transcript's durable user messages pass `interactive === true`; queue
 * previews, submission echoes and read-only views pass false, so they keep the plain
 * text they have always shown.
 */
function PasteCard({ node, raw, interactive, t }) {
  const [open, setOpen] = React.useState(false);
  const [copyState, setCopyState] = React.useState(null);
  const data = node !== null && typeof node === "object" && node.data !== null && typeof node.data === "object" ? node.data : {};
  const bytes = typeof data.bytes === "number" ? data.bytes : utf8Bytes(raw);
  const lines = typeof data.lines === "number" ? data.lines : countLines(raw);
  const say = typeof t === "function" ? t : (key) => key;
  if (interactive !== true) {
    return React.createElement("span", { className: "dshps-card-plain" }, raw);
  }
  const copy = () => {
    const fail = () => setCopyState("failed");
    try {
      const clipboard = typeof navigator === "undefined" ? undefined : navigator.clipboard;
      if (clipboard === undefined || clipboard === null || typeof clipboard.writeText !== "function") { fail(); return; }
      Promise.resolve(clipboard.writeText(raw)).then(() => {
        setCopyState("ok");
        setTimeout(() => setCopyState(null), 1500);
      }, fail);
    } catch {
      fail();
    }
  };
  const label = copyState === "ok" ? say("pastedCopied") : copyState === "failed" ? say("pastedCopyFailed") : say("pastedCopy");
  const preview = raw.split("\n").slice(0, 2).join("\n");
  return React.createElement(
    "span",
    { className: "dshps-card", "data-dshps-paste-card": "" },
    React.createElement(
      "span",
      { className: "dshps-card-head" },
      React.createElement("span", { className: "dshps-card-title" }, say("pastedTitle")),
      React.createElement("span", { className: "dshps-card-meta" }, say("pastedMeta", { size: formatFoldSize(bytes), lines })),
      React.createElement(
        "button",
        {
          type: "button",
          className: "dshps-card-action",
          "data-paste-card-toggle": "",
          "aria-expanded": open ? "true" : "false",
          onClick: () => setOpen((current) => current !== true),
        },
        open ? say("pastedHide") : say("pastedShow"),
      ),
      React.createElement(
        "button",
        { type: "button", className: "dshps-card-action", "data-paste-card-copy": "", onClick: copy },
        label,
      ),
    ),
    open
      ? React.createElement("pre", { className: "dshps-card-full", "data-paste-card-full": "" }, raw)
      : React.createElement("pre", { className: "dshps-card-preview", "data-paste-card-preview": "" }, preview),
  );
}
```

注册函数：

```js
/**
 * Register the rule and its renderer. Missing `uiConversation` (an older host) turns
 * this capability off instead of stalling the whole plugin.
 */
function installTranscriptCard(ctx) {
  const uiConversation = typeof ctx.get === "function" ? ctx.get("uiConversation") : undefined;
  if (uiConversation === undefined || uiConversation === null || uiConversation.userText === undefined) {
    diag({ transcriptCard: "no-service" });
    return;
  }
  ctx.effect(() => uiConversation.userText.register(pasteTextRule), "dsh-paste-spill: paste text rule");
  diag({ transcriptCard: "registered" });
  ctx.slots.inject("conversation.user.text.node", function* () {
    yield ctx.slots.register({ name: "conversation.user.text.node", key: "paste", priority: 100, locale: NS }, PasteCard);
  });
}
```

`apply` 里在 `registerFoldSource(ctx)` 之后调用 `installTranscriptCard(ctx);`。

样式追加到既有样式串末尾：

```js
".dshps-card{display:inline-block;box-sizing:border-box;width:100%;min-width:0;" +
"margin:2px 0;padding:8px 10px;text-align:left;border-radius:10px;" +
"border:.5px solid var(--dsw-alias-border-l2,#0000001f);" +
"background:var(--dsw-specific-input-major,#0000000a);font:inherit}" +
".dshps-card-head{display:flex;align-items:center;gap:8px;min-width:0;font-size:12px;line-height:18px}" +
".dshps-card-title{flex:none;color:var(--dsw-alias-label-primary)}" +
".dshps-card-meta{flex:1 1 auto;min-width:0;overflow:hidden;text-overflow:ellipsis;" +
"white-space:nowrap;color:var(--dsw-alias-label-tertiary)}" +
".dshps-card-action{flex:none;padding:2px 6px;border:none;border-radius:6px;background:transparent;" +
"color:var(--dsw-alias-label-secondary);font:inherit;cursor:pointer}" +
".dshps-card-action:hover{background:var(--dsw-alias-interactive-bg-hover,#0000000a);" +
"color:var(--dsw-alias-label-primary)}" +
".dshps-card-action:focus-visible{outline:2px solid var(--dsw-alias-state-business-primary,#4d6bfe);" +
"outline-offset:1px}" +
".dshps-card-preview,.dshps-card-full{margin:6px 0 0;padding:0;overflow:hidden;" +
"font-family:ui-monospace,SFMono-Regular,Menlo,monospace;font-size:12px;line-height:18px;" +
"white-space:pre-wrap;word-break:break-word;color:var(--dsw-alias-label-secondary)}" +
".dshps-card-preview{max-height:36px}" +
".dshps-card-full{max-height:320px;overflow:auto}"
```

- [ ] **Step 4: 运行测试确认通过**

Run: `node --test`
Expected: 全部 PASS

- [ ] **Step 5: 提交**

```bash
git add lib/client.js test/bundle.test.js
git commit -m "Render sent pastes as a collapsible card in the transcript"
```

### Task 3: 文档、构建标识与全量验证

**Files:**
- Modify: `lib/client.js`（`BUILD_REV`）、`README.md`、`README.zh.md`、`docs/internals.md`

- [ ] **Step 1:** `BUILD_REV` 改为 `"transcript-1"`，便于在 GUI 诊断里确认新构建。
- [ ] **Step 2:** 两份 README 在"转文件层"之后新增"## 发送后（对话内卡片）"一节，说明：≥4 KB 段落出卡、模型输入与 session 不变、队列预览不变、纯长度判定的已知行为。
- [ ] **Step 3:** `docs/internals.md` 追加一节，记录否决 sentinel 标记的理由与 `interactive` gating 的理由。
- [ ] **Step 4:** `node --test` 全绿；`git add -A && git commit -m "Document the transcript paste card"`。