# 可配置阈值（4 KB / 50 KB）设计

日期：2026-11-04
状态：已与用户确认方案（入口 = 设置 → 插件 tab；边界 = 只允许合法区间；单位 = 整数 KB）

## 1. 问题

两个决定插件行为的阈值写死在浏览器半：

```js
const FOLD_BYTES = 4000;    // 4–50 KB 折叠成 chip 的下界
const SPILL_BYTES = 50000;  // ≥ 该值转真附件的下界
```

用户想按自己的工作习惯调整（例如日志类粘贴阈值调高、代码类调低），并在设置界面里改。

## 2. 目标与非目标

**目标**

- 两个阈值可在 DSH 设置界面里修改，改动**立即生效**（只影响之后的粘贴）；
- 值持久化在 profile 配置里，手改配置文件同样有效；
- 非法组合在保存时被拒绝，不静默改写；
- 老宿主 / 没有 settings 服务的部署上，插件行为与今天完全一致（默认值）。

**非目标**

- 不提供"关掉某一层"的开关（用户已明确选 A：只允许合法区间）；
- 不为"发送后卡片"单独开阈值——它继续等于折叠阈值；
- 不做按会话/按工作区的差异化配置（YAGNI）。

## 3. 扩展点事实（已在本机核对）

| 事实 | 出处 |
| --- | --- |
| 插件 Config 的 `.volatile()` 字段才会进 `settings.describe`，且写入**不 remount** 插件（`applies: "live"`） | `@deepseek-ai/dsh-settings/lib/index.js`：`volatileForm`、`describe` |
| 表单按 **profile entry id** 寻址（`entry.options.id`） | 同上 `describe()` 返回 `ns: entry.options.id` |
| schema 取自插件模块的 `Config` 导出 | 同上 `schema(entry)`：`entry.fiber.runtime.Config` |
| 客户端服务 `configForms.get(entryId)` → `{ getSnapshot().value, subscribe, set(key, value) }` | `@deepseek-ai/dsh-client-ui-settings/lib/client.js`：`ConfigForms` |
| `configForms.whileServed(namespaces, register)` 在宿主服务该命名空间时才注册 | 同上 |
| 设置页 tab 槽位是一个 list：`settings.plugins.tab` | `@deepseek-ai/dsh-client-ui-settings-plugins/lib/client.js` |
| userText 规则文档缓存按 **rules 数组同一性** 与 `definition` 同一性失效 | `@deepseek-ai/dsh-client-ui-conversation/lib/client.js`：`UserTextDocument.prepare` |
| 我们的 profile entry id 是 `paste-spill` | 本仓库 `cordis.patch.yml` |

`volatile()` 在 schemastery 上是 `Schema.prototype.volatile()`（重复调用会抛 `volatile schema is already wrapped`）；老宿主没有该方法时需按 `dsh-image-gen` 的写法做条件 shim。

## 4. 设计

### 4.1 宿主半：声明 Config

`lib/index.js` 新增：

```js
import z from "@deepseek-ai/schemastery";

/** 老宿主（schemastery 无 .volatile()）上退化为普通 schema：不够可配置，但不会崩。 */
const live = (schema) => (typeof schema.volatile === "function" ? schema.volatile() : schema);

export const Config = z.object({
  foldBytes: live(z.number().min(1000).max(1_000_000).default(4000)
    .description("文本 ≥ 该字节数折叠成 chip")),
  spillBytes: live(z.number().min(2000).max(10_000_000).default(50_000)
    .description("文本 ≥ 该字节数转成真附件")),
});
```

- 存**字节**（代码内部一律按字节），界面换算成 KB；
- 宿主 `apply` **不读取**这两个值：折叠/转文件都是浏览器侧的判定，宿主半只负责声明 schema 供设置页读写；
- `@deepseek-ai/schemastery` 加进 `peerDependencies`（与 `@deepseek-ai/cordis` 一样标 optional）。

### 4.2 客户端半：阈值 store

- 常量改名降级为默认值：`DEFAULT_FOLD_BYTES = 4000`、`DEFAULT_SPILL_BYTES = 50_000`；
- 模块级 `let thresholds = { foldBytes, spillBytes }`，由 `normalizeThresholds(value)` 写入；
- 所有运行时判定点从 `thresholds` 读：`decidePaste`、`reactToDraft` 的 spill 判断与其后两处 `run.length < SPILL_BYTES` 比较、折叠外扩时的 `FOLD_BYTES` 判断、`PASTE_CARD_BYTES`；
- `installThresholds(ctx)`：
  1. `const configForms = ctx.get("configForms")`；缺失 → `diag({ thresholds: "no-service" })`，保持默认值，返回；
  2. 用容错匹配找命名空间（见 4.3）→ `configForms.get(ns)`；
  3. 立即读一次 `getSnapshot().value` 并归一化；
  4. `ctx.effect(() => form.subscribe(() => applyFormValue(form.getSnapshot().value)), ...)`，后续每次变化都重新归一化；
  5. 每次阈值**真的改变**时（浅比较）调用 `refreshTranscriptRule(ctx)`。

### 4.3 entry id 的容错匹配

我们的 bundle patch 用 `id: paste-spill`，但 dsh-market 安装时 entry id 可能被改写。做法：

```js
const wanted = (ns) => ns === ENTRY_ID || ns.endsWith(`-${ENTRY_ID}`) || ns.endsWith(`/${ENTRY_ID}`);
```

从 `configForms.describe().getSnapshot().view.namespaces` 里取第一个满足 `wanted(ns)` 的命名空间；一个都没有 → 视为"未被服务"，保持默认值并 `diag({ thresholds: "not-served" })`。

选 `describe()` 而不是 `get(ENTRY_ID)` 的直接原因是：`get()` 对不存在的命名空间也会返回一个空 form（第一次 `ensure()` 后才知道有没有值），靠 `describe()` 的 namespaces 列表才能区分"没配置"和"配置服务不在"。

### 4.4 归一化与兜底

`normalizeThresholds(value)` 的规则，按序：

1. 非对象 → 全默认；
2. `foldBytes`：`Math.round`，clamp 到 `[1000, 1_000_000]`，非有限数 → 默认；
3. `spillBytes`：`Math.round`，clamp 到 `[2000, 10_000_000]`，非有限数 → 默认；
4. **顺序兜底**：若 `spillBytes <= foldBytes`，则 `spillBytes = foldBytes + 1`，并 `diag({ thresholdsClamped: true })`。

第 4 条只对手改配置这类"界面拦不住"的输入生效——设置页自身在保存前就拒绝这种组合，不会走到这里。

### 4.5 规则缓存失效

发送后卡片的阈值来自 `thresholds.foldBytes`，但 userText 的文档缓存只在 **rules 数组换新** 或 `definition` 对象换新时才重算（见 §3 末行）。因此：

`refreshTranscriptRule(ctx)` = 释放上一次的 disposer，再用当前阈值闭包出一个**新的**规则对象注册一次。注册会让 registry 的 `revision++` 并重建 `rules` 数组，缓存随之失效；渲染器的 slot key（`paste`）不变，所以视觉上无感。

阈值没变时（订阅回调里浅比较相等）不重注册，避免每次无关的设置写入都打断缓存。

### 4.6 设置界面

注册进 `settings.plugins.tab`，由 `configForms.whileServed([ns], ...)` 包裹，命名空间不被服务的部署上看不到任何痕迹。

卡片内容（`PasteSpillSettings`）：

- 标题「粘贴折叠 / Paste folding」；
- 两个 `input[type=number]`（step=1，单位 KB）：
  - 折叠阈值，默认显示 `4`；
  - 转文件阈值，默认显示 `50`；
- 一行摘要（随输入实时更新）：「≥4 KB 折叠成 chip · ≥50 KB 转成真附件」；
- 小字：「只影响之后的粘贴；已经折叠的 chip 不变。」与「1 KB = 1000 字节」；
- 校验：任一为空/非数字/越范围/`spill <= fold` → 行内红字 + 「保存」禁用；
- 「保存」：`Math.round(kb * 1000)` 后**一次 `form.mutate([...两个 set 操作])`** 原子提交（`set()` 只是 `mutate` 的单字段包装，两次 `set` 会各自带 revision 竞争，所以用一次 mutate）；返回 `Promise<boolean>`：true → 「已保存」1.5 秒后消失，false → 行内错误并保留输入；
- 「恢复默认」：写回 4000 / 50000。

选择显式"保存"而不是失焦即存：`ConfigForm` 的写入是**串行队列 + revision CAS + 失败后自动重读**的异步操作（`ConfigFormController.mutate`），逐字符保存会挤满队列并与 CAS 打架，且用户无法看清自己提交了什么。

### 4.7 数据流

```
设置页输入 KB ──写──> profile cordis patch（config-editor）
                          │
                dsh-settings describe（volatile 字段）
                          │
        configForms.get(ns).subscribe  ──归一化──> thresholds
                          │
        decidePaste / spill 判定 / PASTE_CARD_BYTES / 规则重注册
```

## 5. 错误处理

| 情况 | 行为 |
| --- | --- |
| 无 `configForms` 服务（老宿主） | 默认值，`diag({ thresholds: "no-service" })`，不注册设置页 |
| 命名空间未被服务 | 默认值，`diag({ thresholds: "not-served" })`，不注册设置页 |
| 写入被拒（revision 冲突 / 校验失败） | 设置页显示错误，保留用户输入，可重试 |
| 手改配置文件写出 `spill <= fold` | 归一化兜底 `spill = fold + 1` + diag |
| 阈值改变时输入框里有已折叠 chip | 不受影响：volatile 不 remount；chip 只依赖 `foldTextByRef` |

## 6. 测试

宿主半（`test/plugin.test.js`）：

1. `Config` 存在、默认值 4000/50000、字段带 volatile 标记（老宿主 shim 下无标记也不抛错）。

浏览器半（`test/bundle.test.js`）：

2. `normalizeThresholds`：默认、取整、上下界、`spill <= fold` 兜底、非数字；
3. `decidePaste` 跟随 `thresholds`（例如 fold=2000 时 2.5 KB 粘贴折叠）；
4. 发送后卡片阈值跟随 fold（fold 调到 8000 后，6 KB 消息不再出卡）；
5. 阈值变更后规则被重新注册（disposer 被调用、registry revision 增加）；
6. 阈值未变时不重注册；
7. `installThresholds`：无服务 / 未服务 → 默认值且不注册设置页；有服务 → 读到值并订阅；
8. 设置卡组件：渲染当前值、非法组合禁用保存并显示错误、保存写入正确字节数、恢复默认。

## 7. 兼容性

- `engines.dsh` 不变（`>=0.1.7-rc.2 <0.3.0-0`）；**没有 settings 服务**的宿主上等于今天的默认行为（值取默认、设置页不出现），因为协议是"服务可选、阈值有默认"；
- `@deepseek-ai/schemastery` 是 dsh 运行时自带的包，按 `dsh-image-gen` 的做法静态 import + 声明为 optional peer。它**不是**可选行为：schema 必须在模块求值时存在，所以这个包缺失时宿主半启动失败——与 `dsh-image-gen`、stock 插件一致；settings 服务缺失才是被容忍的那种"老宿主"。

## 8. 验收

- 设置 → 插件 → 「粘贴折叠」出现；
- 把折叠阈值改成 8 KB：8 KB 以下粘贴不再折叠；6 KB 消息在对话里不再出卡；
- 把转文件阈值改成 6 KB：8 KB 粘贴直接变附件；
- 设 `spill <= fold`：保存被拒并提示；
- 改完 `cordis.patch.yml`（profile）里能看到这两个值；手改成非法值后插件仍正常工作（兜底）；
- 全程不需要重启或刷新页面。