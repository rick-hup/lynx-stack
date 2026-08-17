# R5 · gesture-runtime 与滚动模型

> 一手来源：`packages/lynx/gesture-runtime/src/` 全部源码、`packages/react/runtime/src/snapshot/`（手势/事件接入）、`@lynx-js/types@4.1.0`、lynxjs.org 官方文档（主线程脚本 / 最佳实践 / scroll-view，经搜索摘要确认）。
> 标注：**[确认]** / **[推断]** / **[实证]** / **[官方文档]**（lynxjs.org，本次经搜索摘要而非直接抓取确认）。

---

## 0. 一句话结论

**手势与滚动两条地基都已就位且都是「主线程直达」**：`@lynx-js/gesture-runtime` 提供了完整的 RNGH 式识别器体系（引擎识别 + MTS worklet 回调 + 状态控制 PAPI）；滚动事件官方支持 `main-thread:bindscroll`（含全局版 `main-thread:global-bindscroll`），是官方最佳实践推荐的滚动联动路径。**Reanimated 的 `useAnimatedScrollHandler` 与手势驱动动画在 Lynx 上的等价前提全部成立。**

---

## 1. gesture-runtime 现状

### 1.1 识别器与组合 **[确认]**

源码（`src/`，16 个文件，约 1200 行）：

- **识别器**：`PanGesture`（连续，minDistance 配置，事件载荷 `changeX/changeY`）、`TapGesture`、`LongPressGesture`、`FlingGesture`、`NativeGesture`、`DefaultScrollGesture`（tapSlop 配置）
- **组合**：`ComposedGesture` / `SimultaneousGesture` / `ExclusiveGesture` / `RaceGesture`（`composition.ts`）；单手势层面的关系算子 `externalWaitFor` / `externalSimultaneous` / `externalContinueWith`（对应 RNGH 的 `requireToFail` / `simultaneousHandlers`）
- **回调生命周期**：`onBegin/onStart/onEnd` + 连续手势的 `onUpdate` + 触点级 `onTouchesDown/Move/Up/Cancel`

### 1.2 线程模型与机制 **[确认]**

```
BTS：useGesture(PanGesture).onUpdate(cb) —— 构建手势对象
     │  cb 必须是 worklet（'main thread' 指令），否则 wrapCallback 直接 throw
     │  （baseGesture.ts:30-34）
     │  手势对象 serialize（config/id/type/关系/callbacks）→ main-thread:gesture={...} 绑定
     ▼
引擎：识别手势（原生侧），回调 → MTS worklet
MTS：cb(event, stateManager)
     │  stateManager.active()/fail()/end()          → __SetGestureState PAPI
     │  stateManager.consumeGesture()/interceptGesture() → __ConsumeGesture PAPI（手势竞争仲裁）
     ▼
事件载荷：GestureChangeEvent（如 pan 的 changeX/changeY），currentTarget 为 MTS Element
```

- **识别在引擎**、**回调在 MTS**、**状态仲裁经 PAPI 回引擎**——与 RNGH + Reanimated 的分层完全同构。
- ReactLynx 接入点：`packages/react/runtime/src/snapshot/gesture/processGesture.ts`（`main-thread:gesture` prop 的处理）。

### 1.3 与 motion 的结合点（G4 的输入）

R1 已确认 motion 现状无手势 handlers；`examples/motion` 的 iOSSlider 是**手写** `main-thread:bindtouch*` + MotionValue + spring 回弹——说明：
- **可行路径 A（直接依赖）**：motion 声明对 `@lynx-js/gesture-runtime` 的 peer 依赖，手势回调里驱动 SharedValue/MotionValue。成本最低，语义对齐 RNGH。
- **路径 B（适配层）**：motion 定义自己的手势回调签名（对齐 Reanimated 的 `onStart/onUpdate/onEnd` 事件形状），底层委托 gesture-runtime。
- **路径 C（内置识别器）**：不推荐——引擎已做识别，重复造轮子无收益。
- **[推断]** 倾向 A/B：G4 决策。注意 gesture-runtime 的包名/版本独立性（v2.1.3，独立发布节奏）。

## 2. 滚动事件模型

### 2.1 主线程滚动事件 **[官方文档]**

- `<scroll-view main-thread:bindscroll={onScroll}>` —— 官方主线程脚本文档的招牌示例；handler 内 `event.detail.scrollTop` + `setStyleProperty` 直接驱动样式（官方「最佳实践」明确对比：❌ 后台线程处理滚动联动 vs ✅ main-thread 处理，后者无掉帧）。
- **`main-thread:global-bindscroll`** —— 全局滚动事件版本（最佳实践文档），无需绑在 scroller 元素上。
- `<list>` 同样支持（官方示例与社区用例：`bindscroll` + `main-thread:bindscroll` 并存）。
- `scrollend` 也有 main-thread 版（Lynx UI 的 `useBounces` 用 `main-thread:bindscroll` / `main-thread:bindscrollend` 实现自定义回弹）。
- 类型缺口：`@lynx-js/types` 的 `LynxEventPropsBase` 未列 scroll（`bindscroll` 是 scroll-view/list 的元素级 prop），故 `main-thread:bindscroll` 的类型可能缺失——**[推断]** 类型层面待补，机制层面成立（`updateWorkletEvent` → `__AddEvent` 对 eventType/eventName 是通用的，`workletEvent.ts:63-69` **[确认]**）。

### 2.2 事件载荷与控制 **[确认]**

`ScrollEvent.detail = BaseScrollInfo { scrollTop, scrollLeft, scrollHeight, ... }`（`types/common/element/common.d.ts:6-35`）；`bindscrolltoupper/lower/end/contentsizechanged` 同载荷族。滚动控制方法（`scrollTo` 等）走 `invoke`（异步，R2 结论）。

### 2.3 对 Reanimated 等价前提的判定

| Reanimated 能力 | Lynx 等价前提 | 判定 |
|---|---|---|
| `useAnimatedScrollHandler`（滚动同步驱动 UI 线程动画） | `main-thread:bindscroll` + MTS 同步 `setStyleProperty` | ✅ 官方背书的路径 |
| 滚动 + 手势组合（如下拉刷新交互） | `DefaultScrollGesture` / `NativeGesture` + 组合算子 | ✅ 识别器在 |
| 滚动位置读取（`sv.value = event.contentOffset.y` 式） | `event.detail.scrollTop/scrollLeft` 随事件到手 | ✅ |
| 惯性/减速动画接管 | `scrollend` 事件 + MTS spring | ✅ 可拼（`useBounces` 即先例） |

## 3. 给 G4（手势/滚动驱动设计）的输入

1. **手势**：在「直接依赖 gesture-runtime」与「薄适配层」之间决策（§1.3）；无论哪条，手势 → 动画值的数据通路都是「MTS worklet 回调里写 SharedValue」，与 G1 的架构选型直接耦合。
2. **滚动**：`useAnimatedScrollHandler` 等价物 = `main-thread:bindscroll` 的 hooks 封装 + SharedValue 写入；`global-bindscroll` 支撑「任意元素响应页面滚动」的场景。
3. **类型补齐**（`main-thread:bindscroll` 等）是一个小而确定的上游贡献点/本地 patch 点。

## 附：关键索引

- gesture-runtime 源码：`packages/lynx/gesture-runtime/src/`（`baseGesture.ts` / `composition.ts` / `gestureInterface.ts`）
- worklet 强制： `baseGesture.ts:30-34`（非 MTS 函数直接 throw）
- 状态 PAPI：`__SetGestureState` / `__ConsumeGesture`（`baseGesture.ts:62-105`）
- ReactLynx 手势接入：`packages/react/runtime/src/snapshot/gesture/processGesture.ts`
- MTS 事件注册机制：`packages/react/runtime/src/snapshot/snapshot/workletEvent.ts:63-69`
- 官方文档：主线程脚本 <https://lynxjs.org/zh/react/main-thread-script.html>；最佳实践 <https://lynxjs.org/zh/react/best-practices.html>；scroll-view <https://lynxjs.org/zh/api/elements/built-in/scroll-view.html>
