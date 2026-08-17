# R2 · Lynx 主线程（MTS）能力边界

> 一手来源：`@lynx-js/types@4.1.0` 类型定义（`types/main-thread/`、`types/common/`）、`packages/react/runtime/src/`（worklet/线程通信实现）、`packages/motion/src/env_types/papi.d.ts`（motion 自声明的 PAPI 子集）、`examples/` 实证用法、`packages/motion/src/mini/` 实证用法。
> 标注：**[确认]** = 源码/类型直接证实；**[推断]** = 合理外推；**[实证]** = 仓内代码实际在用的用法。
> 版本注：lynxjs.org 文档站本次不可达（网络限制），版本号均以类型定义的 `@since` 标注为准。

---

## 0. 一句话结论

**MTS 具备「同步写元素 + 同步读部分属性 + rAF 帧驱动 + 全事件面」的动画地基，但跨线程通信是公开层面上的纯消息式（双向 Promise/事件），没有 runOnUISync 式的锁同步调用；布局主动测量是异步的（`invoke`），只有 `layoutchange` 事件是被动的同步布局快照来源。**

---

## 1. Element API 面（MTS 侧，typed）

来源：`@lynx-js/types/types/main-thread/element.d.ts`。**[确认]** 全部 `@since Lynx 2.14`，`animate` 例外（3.4）：

| API | 同步性 | 说明 |
|---|---|---|
| `setAttribute(name, value)` | 同步 | 写属性 |
| `getAttribute(name)` / `getAttributeNames()` | 同步 | 读属性 |
| `setStyleProperty(name, value)` / `setStyleProperties(styles)` | 同步 | 写样式（kebab-case）——motion `styleEffect` 的落点 |
| `querySelector(sel)` / `querySelectorAll(sel)` | 同步 | 子树选择器 |
| `invoke(methodName, params): Promise<any>` | **异步** | 调 UI 方法（测量走这里） |
| `animate(keyframes, options): Animation` | 同步调用、引擎驱动 | **WAAPI 风格，`@since Lynx 3.4`**；返回的 `Animation` 有 `play/pause/cancel` |

### 对应 PAPI 层

来源：`packages/motion/src/env_types/papi.d.ts`（motion 为跑通双平台自声明的子集）。**[确认]**：

- 写：`__SetAttribute` / `__AddInlineStyle` / `__FlushElementTree(element?)`（手动触发渲染提交）
- 读：`__GetAttributeByName` / `__GetAttributeNames` / **`__GetComputedStyleByKey(e, key): string`（同步返回）**
- 选择：`__QuerySelector(All)` / `__GetPageElement`
- UI 方法：`__InvokeUIMethod(e, method, params, callback)` —— **callback 式，异步**
- 动画：**`__ElementAnimate(element, [operation, name, keyframes, options?])`** —— operation 枚举 START/PLAY/PAUSE/CANCEL；options 支持 duration/delay/iterationCount/fillMode/timingFunction/direction。这正是 R1 发现的「已声明但全仓未调用」的原生动画 PAPI（R1 结论：接通它是最现实的性能演进路径）
- 其他：`__LoadLepusChunk`（动态加载 MTS chunk）

## 2. 布局测量：主动异步、被动同步

- **主动测量 = 异步**：`element.invoke('boundingClientRect', …)` → `__InvokeUIMethod` → Promise/callback。**[确认]**（类型签名）MTS 侧没有同步的 `getBoundingClientRect`。`examples/motion` 的 iOSSlider 用 `invoke` 做手动测量 **[实证]**。
- **唯一同步的样式读取**是 `__GetComputedStyleByKey`——但 R1 已证它是「computed-style 解析」，对 `auto`/未设置的布局值给不出真实像素，**不能充当 FLIP 测量的真源**。
- **被动布局快照 = 同步随事件到手**：`layoutchange` 事件的 `detail` 直接携带 `{width, height, left, top, right, bottom}`（`types/common/events.d.ts:419+`，全平台 Android/iOS/Harmony/PC）**[确认]**。这是布局观测的关键事实，R4 深挖。

> **对布局动画的直接推论**：FLIP 式「布局前/后各测一次」在 Lynx 上没有同步双测通道；可行路径是 ① 用 `layoutchange` 事件快照（引擎免费给的 after 值）+ 动画系统自记的 before 值，或 ② 接受 `invoke` 的一帧异步延迟。

## 3. 事件模型：`main-thread:bind*` 全事件面

类型机制（`types/main-thread/events.d.ts`）**[确认]**：`LynxEventPropsBase` 的每个事件 key 都映射出 `main-thread:<lowercase-key>` prop；MTS 事件对象的 `target`/`currentTarget` 是 MTS `Element`。覆盖：**touch / mouse / wheel / key / animation / transition / layoutchange / UIAppearance（appear/disappear）**。

仓内实证绑定名（`examples/` grep）**[实证]**：`main-thread:bindtap`、`main-thread:bindtouchstart/move/end`、`main-thread:ref`、`main-thread:gesture`。

**[推断]** `main-thread:bindlayoutchange` 按同一映射机制成立（类型层面保证），但仓内无实际用例——R4 需验证其真实触发行为。

## 4. 帧回调与定时器

- **`requestAnimationFrame(cb)` / `cancelAnimationFrame(id)` 是全局函数，`@since Lynx 3.0`**（以下版本用 `lynx.requestAnimationFrame`）（`types/common/global.d.ts:34-46`）**[确认]**。MTS 可用性有实证：motion mini 的动画循环直接调全局 `requestAnimationFrame`（`packages/motion/src/mini/core/animate.ts:200,204`）**[实证]**。
- **定时器**：`setTimeout/setInterval/clearTimeout/clearInterval` 有 ambient 全局声明（`global.d.ts:51-54`），但 MTS 版 `lynx` 接口里这四个被**注释掉并标注 "Currently is internal"**（`main-thread/lynx.d.ts:12-19`）**[确认]**。**[推断]** MTS 里定时器不可靠或不推荐；mini 用 `Date.now()` 而非定时器计时与此吻合。动画循环应以 rAF 为准。
- `lynx.performance`（`@since 3.0`）在 MTS 可用 **[确认]**。

## 5. 线程通信：纯消息式，无公开同步调用

**BTS → MTS**（`packages/react/runtime/src/core/thread-function-call/main-thread.ts`）**[确认]**：
`runOnMainThread(fn)` 把 worklet 经 `lynx.getCoreContext().dispatchEvent({type: runWorkletCtx, data})` 发向主线程；返回值通过 `resolveId` + `FunctionCallRet` 事件回传，**调用方拿到 Promise**。要求 Lynx SDK ≥ 2.14。

**MTS → BTS**（`packages/react/runtime/src/core/background-function/run-on-background.ts`）**[确认]**：
`runOnBackground(f)` 同样模式——`lynx.getJSContext().dispatchEvent({type: runOnBackground, …})`，后台侧从 `BackgroundFunctionExecMap` 找到函数执行，回值走 `FunctionCallRet` 事件。**Promise 式异步**。要求 ≥ 2.16。

**同步调用**：`main-thread/lynx.d.ts:21-22` 存在被注释掉的 `triggerLepusBridgeSync`（标注 internal）**[确认]**——引擎内部有同步桥，但**不公开**。公开 API 面 = 纯消息式。

**时延特征 [推断]**：跨线程 = 对端事件循环的一次 dispatch + 回值再一个来回；不适合每帧多次往返。动画系统的设计约束因此是：**每帧状态推进必须整条链都在 MTS 内闭环**，BTS 只参与低频的控制面（启动/停止/参数变更）。

## 6. 「Reanimated worklet 需求 ↔ Lynx MTS 供给」gap 分析

| Reanimated 需求（R3 结论） | Lynx MTS 供给 | 判定 |
|---|---|---|
| worklet 序列化 + UI runtime 重建 | `'main thread'` 指令 + SWC 插件（`swc_plugin_worklet_post_process`）+ `_wkltId` + `runOnMainThread` 调度链 | ✅ 已有雏形（事件/手势在用）；缺通用闭包捕获与每-runtime 缓存基建 **[推断]** |
| UI 线程每帧驱动（vsync→rAF） | 全局 `requestAnimationFrame`（3.0+），MTS 实证可用 | ✅ |
| 每帧同步样式/属性写 | `setStyleProperty(ies)` / `__SetAttribute` / `__AddInlineStyle` + `__FlushElementTree` | ✅ 同步 |
| SharedValue JS 侧**同步读**（runOnUISync 持锁内联） | 无公开同步跨线程调用 | ❌ **最大缺口**：只能「本地缓存 + 脏标记 + 异步拉取」近似，BTS 读 MTS 值是 eventually-consistent |
| 布局同步双测（FLIP 前提） | 主动测量仅异步 `invoke`；`layoutchange` 事件带 rect 快照 | ⚠️ 半可行：事件快照 + 自记 before 值 |
| 引擎级动画 offload（WAAPI 式） | `__ElementAnimate` / `Element.animate`（3.4+） | ✅ 未接通（R1 发现），潜力路径 |
| 手势事件直达 UI 线程 | `main-thread:bind*` + `main-thread:gesture`（gesture-runtime，R5 详查） | ✅ |
| worklet 内可用 JS 子集 | 无 DOM/TurboModule 等价物；`performance`、`console`、全局 rAF；定时器存疑 | ⚠️ 比 RN UI runtime 更瘦 |

## 7. 对 G1 架构选型的输入

1. **Worklet 化路线在 Lynx 上有真实地基**：序列化、调度、rAF、同步写、手势直达全部在位——(b) 方案（Worklet 重做）不是空中楼阁。
2. **SharedValue 语义必须降级设计**：Reanimated 的「JS 线程阻塞式同步读」不可照搬；Lynx 版 SharedValue 的 BTS 读只能是缓存值（脏标记驱动异步同步）。API 设计要直面这一点（例如显式 `getSync` 不可得、`runOnMainThread` 取值为唯一真源）。
3. **`__ElementAnimate` 是第三条路**：引擎驱动动画（不进 JS 帧循环）可能覆盖一大批「可声明」的动画场景，JS 帧循环（rAF + setStyleProperty）留给弹簧/手势联动等必须逐帧计算的场景。混合架构（声明式走引擎 PAPI、命令式走 JS 帧循环）值得在 G1 认真评估。
4. **布局动画的测量约束**（§2 推论）会传导进 G3 的设计：entering/exiting 可以靠事件与挂载钩子，layout transition 的 before 值需要动画系统自存快照。

## 附：关键源码/类型索引

- MTS Element 类型：`@lynx-js/types/types/main-thread/element.d.ts`
- MTS 事件类型：`@lynx-js/types/types/main-thread/events.d.ts`（`LynxWorkletEventProps`）
- 全局 rAF/定时器：`@lynx-js/types/types/common/global.d.ts:25-54`
- `layoutchange` detail：`@lynx-js/types/types/common/events.d.ts:419+`
- PAPI 子集声明：`packages/motion/src/env_types/papi.d.ts`
- BTS→MTS：`packages/react/runtime/src/core/thread-function-call/main-thread.ts`
- MTS→BTS：`packages/react/runtime/src/core/background-function/run-on-background.ts`
- MTS rAF 实证：`packages/motion/src/mini/core/animate.ts:200`
