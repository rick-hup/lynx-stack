# R1 · `@lynx-js/motion` 现状盘点

> 研究 issue：rick-hup/lynx-stack#2（Part of #1）
> 研究对象：本仓 `packages/motion/`（版本 0.0.6，依赖上游 `motion` / `motion-dom` / `motion-utils` v13.0.0）
> 方法：以本仓源码、README、`__tests__/`、`examples/motion/` 为一手来源；每条结论标注【确认】（有源码/测试佐证）或【推断】。

## TL;DR

`@lynx-js/motion` 不是 framer-motion 的移植，而是**薄封装层（thin wrapper）**：标准入口直接把上游 `motion` v13 的 `animate` 等函数包一层「Lynx 主线程元素 → 伪 DOM 元素」的适配后转发；mini 入口是一个 ~200 行的自研数值动画循环。**README 对比表中标准版「Layout Animations: Supported」「Gesture Handlers: Full suite」在本包中没有任何对应实现**——`src/` 全仓 grep 不到 layout / drag / hover / pan / 手势组件的任何代码（【确认】）。真实可用的手势模式是用 Lynx 自带的 `main-thread:bindtouch*` 事件手写（见 `examples/motion/src/iOSSlider`）。

## 架构：两层入口，两种实现策略

```
@lynx-js/motion        → src/index.ts → src/animation/index.ts   （封装上游 motion v13）
@lynx-js/motion/mini   → src/mini/index.ts → src/mini/core/*.ts  （自研 mini 内核）
```

### 标准版 = 上游 motion + 主线程适配

`src/animation/index.ts` 的每个导出都是「`'main thread'` 指令 + 转发上游」：

- `animate` / `stagger` 转发自 `motion`（npm 包）；`spring` / `springValue` / `styleEffect` / `mapValue` / `mix` / `transformValue` 转发自 `motion-dom`【确认】。
- 唯一的增值逻辑：`animate` / `styleEffect` 先过 `elementOrSelector2Dom()`（`src/utils/elementHelper.ts`），把 CSS 选择器（走 `lynx.querySelectorAll`）或 `MainThread.Element` 包成 `ElementCompt` 再交给上游【确认】。
- `import ... with { runtime: 'shared' }` 是 Lynx worklet 的运行时共享语法，保证主线程/后台线程复用同一份模块实例。

### 适配层 = 把 Lynx 主线程元素伪装成 DOM

`src/polyfill/shim.ts`（仅 `__MAIN_THREAD__` 时执行）+ `src/polyfill/element.ts`：

| 上游 motion-dom 期望的浏览器环境                                      | Lynx 主线程上的替身                                                                       |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `window` / `document`                                                 | 空对象 + `document.querySelector(All) = lynx.querySelector(All)`                          |
| `HTMLElement` / `Element` / `EventTarget` / `NodeList` / `SVGElement` | 空 class 或 `ElementCompt`（Lynx for Web 平台下直接换成 `ElementCompt`）                  |
| `getComputedStyle(el)`                                                | `__GetComputedStyleByKey(el.element, key)`（Lynx PAPI，经 Proxy 逐 key 取）               |
| `el.style.setProperty(k, v)`                                          | `MainThread.Element.setStyleProperty(k, v)`（`transform: none` 特判改写为 `scale(1, 1)`） |
| `el.getBoundingClientRect()`                                          | **假实现**：parse computed style 的 width/height/left/top，`auto`/未设置一律 `            |
| `performance.now()` / `queueMicrotask`                                | `Date.now()` 兜底 / `lynx.queueMicrotask` 或 Promise 兜底                                 |

要点：`ElementCompt` **没有实现 `element.animate()`（WAAPI）**，所以 motion-dom 的 WAAPI 加速路径不可用，所有元素动画都走 JS 逐帧写 `setStyleProperty`【确认，基于 `src/polyfill/element.ts` 全文无 `animate` 方法】。另外 `src/env_types/papi.d.ts` 声明了原生动画 PAPI `__ElementAnimate`（keyframes + timing + play/pause/cancel），但 `src/` 中**无任何调用**——声明了没接，是潜在的加速路径【确认】。

### Mini 版 = 自研数值动画循环

`src/mini/core/`（`dependencies.test.ts` 强制 mini 不得依赖标准版 `animation/index.ts`）：

- `MotionValue.ts`：自研 `createMotionValue`，接口含 `get/set/jump/getVelocity/onChange/stop/isAnimating/destroy/attach`；速度是 `set()` 时按 `Date.now()` 差分的瞬时值【确认】。
- `animate.ts`：`requestAnimationFrame` + `Date.now()` 的手写循环；**只接受 number**（MotionValue\<number\> / number / setter 函数），**单目标值**；`type: 'keyframes' | 'decay'` 直接 `throw`（`animate.ts:84-86`）；二选一：spring（默认）或 duration+ease 的 tween【确认】。
- `spring.ts` / `easings.ts`：仍然复用 `motion-dom` / `motion-utils` 的 spring 解算器与缓动函数，通过 `src/utils/registeredFunction.ts` 的 `registerCallable` / `globalThis.runOnRegistered` 句柄机制跨 worklet 上下文调用（规避闭包捕获限制）【确认】。

## 能力矩阵

### 标准版（`@lynx-js/motion`）

| 能力                                                            | 状态               | 依据                                                                                                                                                                                                        |
| --------------------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `animate`（元素/选择器/对象/MotionValue/序列 sequence）         | 可用               | 转发 `motion.animate`，重载签名齐全（`src/animation/index.ts:46-156`）；`__tests__/wrapper/animate.test.tsx` 验证选择器/元素/数组/MotionValue 的委托                                                        |
| keyframes（关键帧数组、`AnimationSequence`、per-value options） | 可用               | 类型签名含 `DOMKeyframesDefinition` / `AnimationSequence`；`examples/motion/src/Stagger` 用 `{ y: [50, 0] }`。注：测试只验证委托参数，未端到端跑关键帧插值【部分推断】                                      |
| spring（弹簧）                                                  | 可用               | `spring` / `springValue` 转发 `motion-dom`；`Spring` 示例 + `wrapper/animate.test.tsx`                                                                                                                      |
| 颜色/字符串/百分比插值                                          | 可用               | 上游 motion-dom 能力经 `ElementCompt.style` 代理落地；`ColorInterception` / `BasicPercent` 示例【确认示例存在，真机效果推断】                                                                               |
| `MotionValue`（`motionValue()`）                                | 可用               | 转发 `motion-dom`，`toJSON` 被改写为 `{}` 以避免主线程→后台线程的冗余同步（`src/polyfill/MotionValue.ts`）                                                                                                  |
| `styleEffect`                                                   | 可用               | 转发 `motion-dom`；`iOSSlider` 示例实战使用（`{ y, scaleX, scaleY }` 绑定）                                                                                                                                 |
| `mapValue` / `transformValue` / `mix` / `clamp` / `progress`    | 可用               | 纯函数转发；`wrapper/animate.test.tsx` 有数值断言                                                                                                                                                           |
| `stagger`                                                       | 可用               | 转发 `motion`；`Stagger` 示例                                                                                                                                                                               |
| React 集成 hooks                                                | 部分可用（实验性） | `useMotionValueRef` / `useMotionValueRefEvent` 源码标注 `@experimental ... highly experimental, subject to change`（`src/hooks/`）；`hooks.test.tsx` 覆盖创建/监听/卸载                                     |
| 手势 handlers（drag / pan / hover / press / whileTap…）         | **缺失**           | `src/` 无任何手势代码【确认】。README 表格的 "Full suite" 描述的是上游 framer-motion 而非本包。替代模式：Lynx `main-thread:bindtouchstart/move/end` + `MotionValue` 手写（`examples/motion/src/iOSSlider`） |
| 布局动画（layout animation / FLIP）                             | **缺失**           | `src/` 无 layout 任何代码【确认】；详见下节                                                                                                                                                                 |
| `motion` 组件（`motion.div`、声明式 props）                     | 缺失               | 无 `createMotionComponent`，包是纯命令式 API；README 也只展示命令式用法【确认】                                                                                                                             |
| `AnimatePresence` / 出场退场编排                                | 缺失               | 无相关代码【确认】                                                                                                                                                                                          |
| WAAPI 硬件加速                                                  | 缺失（未接）       | `ElementCompt` 无 `animate()`；`__ElementAnimate` PAPI 已声明未调用【确认】                                                                                                                                 |

### Mini 版（`@lynx-js/motion/mini`）

| 能力                                           | 状态               | 依据                                                                                      |
| ---------------------------------------------- | ------------------ | ----------------------------------------------------------------------------------------- |
| `createMotionValue`（数值）                    | 可用               | 自研实现，`__tests__/MotionValue.test.tsx` 516 行覆盖全方法                               |
| `animate`（number tween，duration+ease）       | 可用               | `mini/core/animate.ts`；`animate.test.tsx` 覆盖 onUpdate/onComplete/stop/then             |
| `animate`（spring）                            | 可用               | 默认即 spring；复用 motion-dom 解算器；`animate.test.tsx` spring 用例                     |
| keyframes（多关键帧）                          | **缺失**           | `type: 'keyframes'` 直接 throw；只支持 起点→终点 两段式【确认】                           |
| decay（惯性衰减）                              | **缺失**           | 同上 throw【确认】                                                                        |
| 字符串/颜色/对象动画                           | 缺失               | 只处理 number；README 亦称 "Numbers only (mostly)"【确认】                                |
| 直接驱动元素样式                               | 缺失（设计上如此） | mini `animate` 只产出数值，需自己 `onUpdate` 里写样式或配合标准版 `styleEffect`【确认】   |
| 缓动函数套件                                   | 可用               | easeIn/Out/InOut、circ、back、anticipate、linear（转发 motion-utils）；`easings.test.tsx` |
| `useMotionValueRef` / `useMotionValueRefEvent` | 可用（实验性）     | 复用 `hooks/useMotionValueRefCore`，注入 mini 的 `createMotionValue`                      |

## 布局动画：README 声称 vs 现实

README 对比表（`packages/motion/README.md:71`）写标准版 **Layout Animations: Supported**。**证伪**，三层证据：

1. **无代码**：`packages/motion/src/` 全量 grep `layout|drag|hover|pan|gesture|whileTap|whileHover|AnimatePresence|motion\.div|createMotionComponent` 零命中（排除注释）【确认】。
2. **无 API 面**：标准入口导出表（`src/index.ts`）只有 12 个命令式函数 + 2 个实验 hook，没有 `motion` 组件——而 framer-motion 的布局动画恰恰挂在 `motion` 组件的 `layout` prop 上，没有组件就无处附着【确认】。
3. **测量基建是假的**：framer-motion 布局动画依赖布局前后各测一次真实包围盒（FLIP）。本包的 `getBoundingClientRect()`（`src/polyfill/element.ts:164-196`）是从 computed style 解析 width/height/left/top 凑出来的——flex 自动定位、`auto`、未设置的值全部落 `0`，布局变化后根本测不出位移【确认】。

**结论**：该表格行的「Standard Motion」列是在描述上游 framer-motion 的特性集（移植目标画像），不是本包已实现的能力；对本包而言布局动画是**缺失**而非「支持」。

**缺口与已有积木**（学习要点）：

- Lynx 主线程**有**真实测量能力：`element.invoke('boundingClientRect')`（`iOSSlider` 示例里 `measureSlider()` 手写调用，`examples/motion/src/iOSSlider/index.tsx:35-42`）——但它是异步 Promise，且没有「布局变更前后自动各测一次」的管线。
- 要做 FLIP，需要自建：布局前快照 → 触发布局 → 布局后快照（同步测量是最大障碍）→ invert 写 transform → play 用现有 `animate`。本包目前一步都没有。
- `__ElementAnimate`（声明于 `src/env_types/papi.d.ts:94`，未被调用）暗示 Lynx 引擎层有原生 keyframe 动画 PAPI，若接通可绕开 JS 逐帧写样式，是性能上最现实的演进路径【推断】。

## `__tests__/` 反映的已验证能力

测试基于 `@lynx-js/react/testing-library`（vitest 插件），在模拟 Lynx 双线程的环境里跑：

| 测试文件                                                                                     | 验证内容                                                                                                                 |
| -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `index.test.ts`                                                                              | 标准入口 13 个导出的存在性（导出面契约）                                                                                 |
| `wrapper/animate.test.tsx`                                                                   | 标准版各函数**委托上游**的参数正确性（mock `motion`/`motion-dom` 断言入参）——验证的是「接线」，不是真动画                |
| `animate.test.tsx`                                                                           | mini `animate`：MotionValue/函数 setter/裸 number 三种目标、onUpdate/onComplete/stop/then、spring 模式、新动画顶掉旧动画 |
| `MotionValue.test.tsx`                                                                       | mini MotionValue 全方法（get/set/velocity/jump/onChange/attach/stop/destroy/toJSON/边界）                                |
| `hooks.test.tsx`                                                                             | 两个实验 hook 的创建、change 回调真实触发、卸载清理、多实例                                                              |
| `element.test.tsx`                                                                           | `ElementCompt` 的 style 代理、getComputedStyle、getBoundingClientRect 解析逻辑                                           |
| `shim.test.ts` / `mini-polyfill.test.tsx`                                                    | 全局 shim 的幂等与分支（`__MAIN_THREAD__`、Lynx for Web）                                                                |
| `dependencies.test.ts`                                                                       | 架构守卫：mini 依赖图不得触及标准版 `animation/index.ts`                                                                 |
| `spring.test.tsx` / `easings.test.tsx` / `utilities.test.ts` / `polyfillMotionValue.test.ts` | mini spring 句柄注册、缓动转发、小工具、toJSON 改写                                                                      |

注意：标准版的测试只断言「参数正确转发给了 motion/motion-dom」，**没有端到端验证 Lynx 元素上真实跑动画**；mini 版才有真实动画行为断言。即「已验证能力」≈ mini 内核 + 适配层接线，标准版的真机表现主要靠 examples 人工验证【确认测试内容，结论为推断】。

## 关键源码路径索引

| 主题                                                  | 路径                                                                                              |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 标准入口导出面                                        | `packages/motion/src/index.ts`                                                                    |
| 标准版封装核心（委托上游）                            | `packages/motion/src/animation/index.ts`                                                          |
| 主线程元素→伪 DOM 适配                                | `packages/motion/src/utils/elementHelper.ts`、`src/utils/isMainThreadElement.ts`                  |
| 伪 DOM 元素（含假 getBoundingClientRect）             | `packages/motion/src/polyfill/element.ts`                                                         |
| 浏览器全局 shim                                       | `packages/motion/src/polyfill/shim.ts`                                                            |
| MotionValue 跨线程优化（toJSON 改写）                 | `packages/motion/src/polyfill/MotionValue.ts`                                                     |
| 实验 hooks                                            | `packages/motion/src/hooks/useMotionValueRefCore.ts`、`useMotionValueRef.ts`、`useMotionEvent.ts` |
| mini 动画循环                                         | `packages/motion/src/mini/core/animate.ts`                                                        |
| mini MotionValue                                      | `packages/motion/src/mini/core/MotionValue.ts`                                                    |
| mini spring/缓动（句柄转发上游解算器）                | `packages/motion/src/mini/core/spring.ts`、`easings.ts`、`src/utils/registeredFunction.ts`        |
| 未使用的原生动画 PAPI 声明                            | `packages/motion/src/env_types/papi.d.ts`（`__ElementAnimate`）                                   |
| README 失实表格                                       | `packages/motion/README.md:67-74`                                                                 |
| 手势 DIY 范式（真实测量 + styleEffect + spring 回弹） | `examples/motion/src/iOSSlider/index.tsx`                                                         |
| 标准版示例集                                          | `examples/motion/src/{Basic,BasicPercent,BasicSelector,ColorInterception,Spring,Stagger,Text}`    |
| mini 示例                                             | `examples/motion/src/Mini`                                                                        |
