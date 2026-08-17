# R4 · Lynx 布局观测能力

> 一手来源：`@lynx-js/types@4.1.0`、`packages/react/runtime/src/snapshot/`（patch 生成与应用）、`packages/react/runtime/src/worklet-runtime/`（MTS 侧 runtime）、`packages/web-platform/web-elements/`（web 侧 layoutchange 实现，作参照）。
> 建立在 R2（主线程能力边界）结论之上。标注：**[确认]** / **[推断]** / **[实证]** 同 R2 约定。

---

## 0. 一句话结论

**Lynx 有一个 Reanimated 式「挂载事务拦截」的 JS 侧等价物：ReactLynx 的 MTS patch 管线（`updateMainThread` → `snapshotPatchApply` → `__FlushElementTree`）。全部挂载/卸载/属性变更以 opcode 流形式在 MTS 的 JS 里过一遍，且 flush 前存在同步窗口——entering/exiting/transition 三件套的拦截点可以落在这一层，不需要引擎改动。布局观测本身 = `layoutchange` 事件（布局后通知，detail 自带 rect）+ 动画系统自存 before 快照。**

---

## 1. 布局变化观测：`layoutchange` 事件

- **语义 = 布局后通知**：web 平台实现（`web-elements/.../CommonEventsAndMethods.ts:23-58`）基于 **ResizeObserver**——元素尺寸变化后回调，`detail` 用 `getBoundingClientRect()` 现取 `{width, height, left, top, right, bottom, id}` **[确认（web）]**。**[推断]** 原生引擎同语义（类型定义标注全平台：Android/iOS/Harmony/PC）——即「布局完成之后」的事件，不是布局前。
- **线程归属**：事件本身可绑 `main-thread:` 前缀（R2 已确认类型层面成立，`main-thread:bindlayoutchange`），即可**直达 MTS**，无需 BTS 中转。**[推断]** 仓内无 `main-thread:bindlayoutchange` 实证用例，真机行为待验证（G3 的验证切片之一）。
- **时序结论**：`layoutchange` 给你的是 **after 值**（布局已完成、rect 已新）。FLIP 的 **before 值没有引擎通道**——动画系统必须自存每个被观测元素的最近一次 rect（首次观测时播种，之后每个 layoutchange 滚动更新）。

## 2. 主动测量

R2 已结：`element.invoke('boundingClientRect')`（→ `__InvokeUIMethod`，callback/Promise）**异步**，MTS/BTS 两侧都只有这一条主动测量路。结论同 R2 §2：主动测量不适合做每帧 FLIP 的真源，只用于低频一次性查询。

## 3. 挂载/卸载的可见性 ★（本 ticket 核心）

### 3.1 ReactLynx 的更新管线（BTS → MTS）

**[确认]**（`packages/react/runtime/src/snapshot/lifecycle/patch/`）：

```
BTS：React render → diff → 生成 SnapshotPatch（opcode 流：
     CreateElement / InsertBefore / RemoveChild / SetAttribute(s) / nodesRef*）
     │  （经 LifecycleConstant.patchUpdate 全局函数送 MTS）
     ▼
MTS：updateMainThread(data, patchOptions)
     ① JSON.parse → patchList
     ② snapshotPatchApply(patch)：逐条解释 opcode，操作 SnapshotInstance 树
        —— RemoveChild / InsertBefore / CreateElement 全部以 JS 函数调用经过这里
     ③ applyRefQueue()：worklet ref 更新（见 §3.2）
     ④ 执行延迟的 runOnMainThread 任务
     ⑤ __FlushElementTree(__page, flushOptions) —— 一次性提交原生渲染
```

**关键性质**：
- 这整条链是 **ReactLynx 运行时的普通 JS 代码，跑在 MTS**（`snapshotPatchApply.ts` 模块注释：「applying them to the DOM in the main thread」）。
- 引擎侧注入点只有一个：`globalThis[LifecycleConstant.patchUpdate] = updateMainThread`（`injectUpdateMainThread`）。
- **flush 前有同步窗口**：opcode 全部应用完（内存中的元素树已改）到 `__FlushElementTree` 真正提交原生之间，是同一个同步 JS 执行流。

### 3.2 卸载在 MTS 的可见性

- **ref 层**：`updateWorkletRef(handle, element)` 在卸载时把 `main-thread:ref` 的 current 置为 **null**（`worklet-runtime/workletRef.ts:87-94`），并有 `removeValueFromWorkletRefMap` 清理 **[确认]**。即 MTS 能感知「某元素没了」——但这是在 patch 应用阶段、flush 提交之前。
- **opcode 层**：`RemoveChild(parentId, childId)` 是显式 opcode（`snapshotPatchApply.ts:53-64`），任何 wrap `snapshotPatchApply` 的代码都能看见「谁要被删、从哪删」，且可以选择**暂扣不执行**（Reanimated proxy 暂扣 Remove mutation 的完全同构）。

### 3.3 BTS 侧可见性（对照）

BTS 有完整 React 生命周期（函数组件 effect / class 生命周期）+ patch 生成逻辑。framer-motion 的 `AnimatePresence` 式方案（组件保持挂载直到离场动画完成）**纯 BTS 就能做**，不需要 MTS 拦截。

## 4. 三件套在 Lynx 上的实现位置推论

| 件 | 推荐拦截层 | 机制 | 引擎改动？ |
|---|---|---|---|
| **entering** | MTS patch 管线（或 BTS React 层） | 见 `CreateElement`+`InsertBefore` → 首帧设初始样式（opacity-0 类），flush 后用 rAF 驱动到终态 | 不需要 |
| **exiting** | **MTS patch 管线**（首选）/ BTS AnimatePresence 式（备选） | 暂扣 `RemoveChild`，保留元素播离场动画，完成后补 RemoveChild + flush | 不需要 **[推断——暂扣后引擎侧无不一致，因未 flush]** |
| **layout transition** | MTS：`layoutchange` 事件 + 自存 before 快照 | 元素 layoutchange 触发 → 与自存 before 值做 FLIP 差分 → rAF 驱动 transform/尺寸插值 | 不需要；**若要做「布局前拦截」（避免一帧跳变）则需引擎支持** |

**与 Reanimated 的对照**：RN 的拦截点在 C++（`MountingOverrideDelegate.pullTransaction`），JS 不可达；**Lynx 的等价物（snapshotPatchApply）是纯 JS、模块级、可 wrap**——这是 Lynx 侧反而更友好的地方。**[推断]** 风险在于 wrap 内部函数依赖 ReactLynx 内部 API 稳定性（非公开 API，版本升级可能破坏）；更稳的做法是推动 ReactLynx 暴露正式的 patch 生命周期钩子（上游化议题）。

## 5. 给 G3（布局动画设计）的输入

1. **exiting 的两种路线需在 G3 权衡**：MTS patch 拦截（性能最优、全程 MTS、贴 Reanimated 哲学）vs BTS AnimatePresence 式（实现最简单、但动画帧在 BTS 会掉帧——除非配合 worklet 把动画体下沉 MTS）。
2. **layoutchange 的 MTS 实证**（`main-thread:bindlayoutchange` 真机行为）应列入验证切片。
3. **自存 rect 注册表**（元素 → 最近 rect）是布局动画子系统的基础设施，随 layoutchange 滚动更新。
4. wrap `snapshotPatchApply` 是原型期可行路径；spec 应记录「需要 ReactLynx 正式钩子」的上游诉求。

## 附：关键源码索引

- MTS patch 应用：`packages/react/runtime/src/snapshot/lifecycle/patch/snapshotPatchApply.ts`（opcode 解释器）
- MTS 更新入口：`packages/react/runtime/src/snapshot/lifecycle/patch/updateMainThread.ts`（`updateMainThread`，含 `__FlushElementTree` 调用点）
- opcode 定义：`packages/react/runtime/src/snapshot/lifecycle/patch/snapshotPatch.ts`（`SnapshotOperation`）
- MTS worklet ref：`packages/react/runtime/src/worklet-runtime/workletRef.ts`（卸载置 null：`updateWorkletRef`）
- web 侧 layoutchange：`packages/web-platform/web-elements/src/elements/common/CommonEventsAndMethods.ts`（ResizeObserver 实现）
- 事件跨线程构造：`packages/web-platform/web-core/ts/client/mainthread/elementAPIs/createCrossThreadEvent.ts:111+`
