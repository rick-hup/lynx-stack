# R3 · Reanimated 架构拆解（Worklet Runtime / SharedValue / LayoutAnimations / Frame Callback）

> 研究对象：`software-mansion/react-native-reanimated` monorepo，`main` 分支（v4.6.0-main，commit `8c1bcf8`，2026-08 浅克隆）。
> v4 起 worklet 机制独立为 `packages/react-native-worklets`，Reanimated（`packages/react-native-reanimated`）只是其消费者。本文所有 `packages/...` 路径均相对该仓库根。
> 标注约定：**[确认]** = 源码/官方文档直接证实；**[推断]** = 基于源码结构的合理外推；**[初判]** = 对 Lynx 的可移植性判断，待 R2/R4 的事实校准。

---

## 0. 总览架构图

```
┌─────────────────────────── JS 线程（RN Runtime，唯一）──────────────────────────┐
│  React / 业务代码                                                                │
│      │                                                                          │
│      │ Babel 插件（构建期）: 'worklet' 指令 → 函数字符串 + __closure + __workletHash │
│      ▼                                                                          │
│  WorkletFunction { __closure, __workletHash, __initData:{code|bytecode} }        │
│      │                                                                          │
│      │ createSerializable / runOnUI（JSI: __workletsModuleProxy）                │
└──────┼──────────────────────────────────────────────────────────────────────────┘
       │  C++ Serializable（按值深拷贝，无共享内存）
       ▼
┌─────────────────── UI 线程（UI Runtime，独立 Hermes，每 App 一个）────────────────┐
│  __valueUnpacker：eval(code) / evalBytecode → .bind(unpackedObj)                 │
│      │      （this.__closure 在构建期已被编译进函数体首行）                         │
│      ▼                                                                          │
│  worklet 执行：每帧 frame callback（requestAnimationFrame → AnimationFrame-      │
│  Batchinator → 平台 vsync：CADisplayLink / Choreographer）                        │
│      │                                                                          │
│      ▼                                                                          │
│  同步提交 props / layout → ShadowTree（同步 updateProps / CommitHook / MountHook）│
└─────────────────────────────────────────────────────────────────────────────────┘
       ▲                     ▲
       │ runOnJS / scheduleOnRN（JSScheduler → CallInvoker::invokeAsync）
       │ SharedValue：Synchronizable 锁 + 异步脏值传播（见 §2）
```

三个运行时种类（runtime kinds，官方文档确认）：**RN Runtime**（JS 线程，唯一能访问 RN API）、**UI Runtime**（worklet runtime 中特殊的一个，通常由 UI/主线程执行，可同帧响应原生事件）、**Worker Runtimes**（`createWorkletRuntime` 按需创建，独立线程）。各 runtime 不共享内存，靠 Serializable 拷贝通信。（来源：`docs.swmansion.com/react-native-worklets/docs/fundamentals/runtimeKinds`）

---

## 1. Worklet Runtime：Babel 序列化 + UI runtime 重建

### 1.1 构建期序列化流水线（Babel 插件）**[确认]**

入口：`packages/react-native-worklets/plugin/src/plugin.ts`（`WorkletsBabelPlugin`）。对带 `'worklet'` 指令的函数：

1. **指令检测与替换**：`plugin/src/workletSubstitution.ts` `processIfWithWorkletDirective` / `processWorklet` —— 递归处理嵌套 worklet，然后把原函数节点替换为 factory 调用。
2. **代码字符串化**：`plugin/src/workletFactory.ts` `makeWorkletFactory` —— `generate()` 生成源码，包成 `(function ...)`，再用一组 Babel 插件二次降级（shorthand、箭头函数、optional chaining 等）。
3. **闭包捕获**：`plugin/src/closure.ts` `getClosure` —— 遍历自由变量；命中 `plugin/src/globals.ts` `notCapturedIdentifiers` 白名单（`global`、`console`、`performance`、`fetch` 等内建）的**不捕获**，留到目标 runtime 的全局对象上解析；其余外部绑定全部隐式捕获进 `__closure`。
4. **闭包解构编译进代码串**：`plugin/src/workletStringCode.ts` `buildWorkletString` —— 在函数体首行注入 `const {a, b} = this.__closure;`（递归 worklet 另注入 `const f = this._recur;`）。**关键设计：闭包变量不进代码串，运行期经 `this.__closure` 传入，函数重建时 `.bind(unpackedObject)`**。
5. **哈希**：对最终代码串做双 DJB2 哈希 → `__workletHash`，同时作为 UI runtime 上的缓存键。
6. **init data**：模块级常量 `worklet_<hash>_init_data`，含 `{code}` 或（release + hermes 字节码选项时）`{bytecode: Uint8Array.buffer}`，dev 下附 `location` / `sourceMap`。
7. **factory 发射**：返回的普通 JS 函数上挂载 `__closure`、`__workletHash`、`__initData`，dev 另挂 `__stackDetails`（错误符号化）。
8. **调用点替换**：原函数 → `factory({initData, ...捕获值})`（bundle 模式下每个 worklet 单独成文件，`require('react-native-worklets/.worklets/<hash>.js')`）。

类型形态：`src/types.ts` —— `WorkletFunction` = 可调用 & `{__closure, __workletHash, __initData?, ...}`。

### 1.2 JS 线程 → C++ 的序列化（createSerializable）**[确认]**

- TS：`src/memory/serializable.native.ts` `createSerializable` → JSI host functions；worklet 走 `cloneWorklet`：dev 校验 `__pluginVersion`，注册 `__stackDetails`，把 `__initData` 标记 `shouldPersistRemote=true`（重的代码串每个 runtime 只反序列化一次），调 `WorkletsModule.createSerializableWorklet`。
- C++：`Common/cpp/worklets/SharedItems/SerializableFactory.cpp` `makeSerializableWorklet` → `RetainingSerializable<SerializableWorklet>`。**没有任何 JS 函数对象跨 runtime**——跨过去的全是按值深拷贝的 C++ Serializable（字符串、数字、嵌套对象）。

### 1.3 UI runtime 重建与执行流水线 **[确认]**

1. **创建**：`WorkletsModuleProxy` 构造时 `RuntimeManager::createUninitializedUIRuntime(AsyncQueueUI)`；`start()` 时 `init()` 并装 `requestAnimationFrame`。
2. **runtime 构造**：`WorkletRuntime.cpp makeRuntime` 建独立 Hermes runtime（`WorkletHermesRuntime`，可选 `LockableRuntime` 给每次 JSI 调用加递归互斥锁）。
3. **装饰（注入 C++ 全局）**：`WorkletRuntimeDecorator::decorate` 注入 `_WORKLET=true`、`__RUNTIME_KIND`、`globalThis.__workletsModuleProxy`、`performance.now`、`_log`、`nativeLoggingHook`、`_createSerializable*` 系列、`_scheduleOnRuntime` 等；`WorkletHermesRuntime` 注入 `evalBytecode`（release 路径）/`evalWithSourceMap`（debug）。
4. **unpacker 自举**：新 runtime 里一开始什么 JS 库都没有。RN 线程把「已 worklet 化的 unpacker 函数」自己的 `__initData.code` 经 JSI 推过去（`loadUnpackersWithCode`），native 侧 `rt.evaluateJavaScript` 执行 → `globalThis.__valueUnpacker` 就位（`UnpackerLoader.h`、`src/memory/valueUnpacker.native.ts`）。
5. **worklet 物化**：`SerializableWorklet::toJSValue`（`Serializable.cpp`）重建对象后调 `__valueUnpacker(obj, "Worklet")`；unpacker 首次见到某 `__workletHash` 时 `eval(initData.code)`（或 `evalBytecode`/`evalWithSourceMap`），结果入 `workletsCache`（每 runtime 每 hash 只 eval 一次），然后 `workletFun.bind(objectToUnpack)`，`this.__closure` 完成闭包解析。
6. **执行**：`WorkletRuntime::runSyncImpl` → `function.call(rt, args...)`，末尾 drain microtasks。debug 下 JSError 被捕获并经 `JSLogger` 回抛到 RN JS 线程（带调度栈符号化）。
7. **调度**：`runOnUI`（`src/threads.native.ts`）批量打包 → `WorkletsModule.scheduleOnUI` → `AsyncQueueUI` → `UIScheduler::scheduleOnUI` → 平台 UI 线程派发；`runOnUISync` 则持 runtime 锁从 JS 线程同步调进 UI runtime。回程 `runOnJS` → `JSScheduler` → RN `CallInvoker::invokeAsync`。

### 1.4 UI runtime 可用的 JS 子集 **[确认]**

- 有：纯 JS 内建（白名单内不捕获的全局在 UI runtime 上解析）、`console`（只是异步转发到 RN runtime 的 shim）、`performance.now`、`queueMicrotask`、`requestAnimationFrame`（JS polyfill over `__nativeRequestAnimationFrame`）、`setTimeout` 系列 polyfill。
- 没有：TurboModules/NativeModules、Metro `require`（模块级 import 若不在白名单则必须可序列化、按值捕获进 `__closure`）、`fetch`/`XMLHttpRequest`/`WebSocket`（白名单内但 UI runtime 全局上通常为 `undefined`）、DOM/`window`、Promise 不可序列化跨 runtime。

---

## 2. SharedValue 跨线程协议

> 术语注意（v4 改名）：旧的 `ShareableValue`/`FrozenObject`/`FrozenValue`/`makeShareableCloneRecursive` 现在是 `Serializable` / `Shareable` / `RetainingSerializable` / `makeSerializableClone`。**不再有 C++ `MutableValue`/`SharedValue` 类——Reanimated 的 `Mutable` 是「JS 装饰过的 worklets Shareable」**。

### 2.1 值在 runtime 间的拷贝 **[确认]**

- `worklets/Compat/StableApi.h:27` —— `class Serializable { virtual jsi::Value toJSValue(jsi::Runtime&) = 0; }` 是一切可传输值的基类（Scalar、String、Array、Object、Map、Set、Worklet、HostObject、ArrayBuffer、Synchronizable、Shareable…）。
- `worklets/SharedItems/Serializable.cpp:29` —— `makeSerializableClone(rt, value, ...)` 把 JSI 值深拷贝成 C++ Serializable 树（标量/容器是纯数据，不带 JSI 句柄），交给 JS 的是带 `NativeState` 的 host object（`__serializableRef`）。物化到目标 runtime 是**惰性**的（`toJSValue(rt)`）。
- 每 runtime 缓存：`worklets/SharedItems/RetainingSerializable.h:20-108` —— `RetainingSerializableStore` 按 runtime 缓存物化结果，同一 Serializable 在同一 runtime 只物化一次。
- JS 侧身份缓存：`src/memory/serializableMappingCache.native.ts`（WeakMap：JS 对象 → SerializableRef），防止已转换对象被重复克隆（如被 worklet 闭包捕获的 Mutable）。

### 2.2 Shareable：跨 runtime 引用 **[确认]**

`Shareable::toJSValue`（`Shareable.cpp:50-56`）按 runtime 身份分岔：host runtime 返回惰性初始化的 `hostJSValue`（经 `__shareableHostUnpacker`）；guest runtime 调 `__shareableGuestUnpacker`。unpacker 是每 runtime 安装的 JS 闭包，guest 侧动词：

- `getSync` → `runOnRuntimeSyncWithId(hostId, get)` —— 同步
- `setAsync` → `scheduleOnRuntimeWithId(hostId, ...)` —— **异步消息，fire-and-forget**
- `setSync` / `getAsync` 同理存在

### 2.3 Mutable = 宿主在 UI runtime 的 Shareable + 双侧装饰器 **[确认]**

`reanimated/src/mutables.native.ts:147-168` —— `makeMutable` = `createShareable(UIRuntimeId, initial, {hostDecorator, guestDecorator})` + 可选 `dirtyFlag = createSynchronizable(false)`。

- **Host（UI runtime）侧**（`src/mutablesCommon.ts:34-152`）：值就是 JS 闭包变量；`.value` 读写皆本地；setter 触发 listeners，并在未脏时 `setDirty(true)`（写跨线程标记 `dirtyFlag?.setBlocking`）。
- **Guest（RN/JS 线程）侧**：`.value` **读** = 先 `dirtyFlag.getBlocking()`，干净则返回缓存 `latest`，脏则 `runOnUISync` 拉取新值并清标记；`.value` **写** = `scheduleOnUI(() => { mutable.value = newValue })` —— **单向异步消息，无确认**。

### 2.4 `runOnUISync` 的真相：持锁内联执行，不是线程跳转 **[确认]**

`WorkletRuntime::runSync`（`WorkletRuntime.h:106-126`）= `acquireRuntimeLock()`（`std::recursive_mutex`）+ `runSyncImpl` **在调用线程上**直接 `function.call(rt, ...)`。同一把锁也被 runtime 自己的队列线程持有（`WorkletRuntime.cpp:211-219`），以此串行化 UI 队列工作与外部线程的同步调用。

**JS 线程读 `sv.value` = JS 线程阻塞在 UI runtime 的递归互斥锁上、内联执行 getter worklet**；它不入队、不等 UI 线程响应。

### 2.5 Synchronizable：RW 锁下的共享 C++ 对象 **[确认]**

`worklets/SharedItems/Synchronizable.h:13-51` —— 一个 `shared_ptr<Synchronizable>` 被所有 runtime 原样共享（双侧持同一 C++ 对象的 JSRef）。并发控制在 `SynchronizableAccess.{h,cpp}`：`std::mutex` + `condition_variable` + 读写计数；`getDirty()` 是刻意无锁的快路径。JS 包装层直接映射到 `__workletsModuleProxy` 的 JSI 方法，**不跨 runtime、不传消息**。

### 2.6 协议总结（显式回答）

**混合，但主体是「读 = 锁式同步共享内存 + 脏标记缓存；写 = 异步单向消息」**：

| 方向                | 机制                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------- |
| JS → UI 读 `.value` | 同步：持 UI runtime 递归锁内联执行；`Synchronizable<bool>` 脏标记 + `latest` 缓存优化 |
| JS → UI 写 `.value` | 异步：`scheduleOnUI` 单向消息，无确认                                                 |
| UI 侧读写 Mutable   | 纯本地（host 闭包），listeners + setDirty 传播通知                                    |
| Synchronizable 原语 | 纯锁式共享 C++ 状态（RW 锁 + condvar），双线程对称                                    |

---

## 3. Frame Callback 帧驱动模型

### 3.1 vsync 源 **[确认]**

- **iOS：`CADisplayLink`**（`WorkletsDisplayLink`）挂在主 runloop —— `worklets/apple/AnimationFrameQueue.mm:63-70` 入队回调并唤醒 display link；`executeQueue:` 以 `displayLink.targetTimestamp` 执行。另有 ProMotion 自适应帧率路径。Reanimated 自己还持第二个 display link（`REANodesManager.mm:25-32`）驱动 `performOperations`。
- **Android：`Choreographer`** 经 `ReactChoreographer.postFrameCallback(NATIVE_ANIMATED_MODULE, …)` —— `worklets/android/.../AnimationFrameQueue.kt:86-90`；vsync 落进 `executeQueue(frameTimeNanos)`（去重时间戳，ns→ms），过 JNI 到 C++。

### 3.2 注册链路与批处理 **[确认]**

1. 平台队列注入为 `RuntimeBindings::RequestAnimationFrame`（iOS `WorkletsModule.mm:137-139`；Android `WorkletsModule.cpp:96-103`）。
2. `WorkletsModuleProxy::start()` 构造 `AnimationFrameBatchinator`（UI runtime + 平台 RAF），把 `batchinator->getJsiRequestAnimationFrame()` 装成 UI runtime 全局的 `__nativeRequestAnimationFrame`。
3. **批处理点**：`AnimationFrameBatchinator.cpp` —— `addToBatch` 在互斥锁下追加 `jsi::Function`；`flush()` 用 `flushRequested_.exchange(true)` 保证**每帧只发一次平台 vsync 请求**；vsync 触发时 `pullCallbacks()` 后在**单次 `uiWorkletRuntime->runSync`**（整批只持一次锁）内执行所有回调，传入 `timestampMs`。
4. JS 层（UI runtime 内）：`worklets/src/runLoop/uiRuntime/requestAnimationFrame.ts` —— 实现 JS `requestAnimationFrame/cancelAnimationFrame` 自续循环：`flushQueue` 设 `__frameTimestamp`、跑全部排队回调、drain microtasks、跑 finalizer，再向 `__nativeRequestAnimationFrame` 重新注册自己。**帧回调的执行位置 = UI runtime 内、batchinator 的单次 runSync 里、每个 vsync 一次**。

### 3.3 mapper 体系（样式更新引擎）**[确认]**

`src/mappers.native.ts:78-122` —— SharedValue listener 只设 `mapper.dirty = true`；`scheduledMapperRun` 每帧经 `requestAnimationFrameFinalizer` 重新注册自己；`mapperRun` 按拓扑序执行脏 mapper，然后跑 finalizer（props 落地下刷）。

### 3.4 withTiming / withSpring 就是帧回调 **[确认]**

动画 builder（`src/animation/timing.ts` 等）返回带 `onStart/onFrame/current/callback` 的普通对象，**不进任何 C++ 注册表**。`sv.value = withTiming(...)` 命中 `valueSetter`（`src/valueSetter.ts:45-69`）：立即 `onStart`；定义 `step(timestamp)`（调 `onFrame`、写 `mutable._value = animation.current` → 触发 listeners → 脏 mapper），并**用 UI rAF 重新注册 `step`** 直到结束。每个进行中的动画 = worklets rAF 队列里的一个 JS 帧回调，被 batchinator 按 vsync 批处理。

### 3.5 props 如何落到原生视图 **[确认]**

1. mapper/style updater 调 `updateProps`（`src/updateProps/updateProps.native.ts:64-129`）：拆分 native vs JS props，批 `nativeOperations`，用 `__requestMapperRunFinalizer(this.flush)` 把落地下刷推迟到**同帧**的 mapper finalizer。
2. `flush()`：native ops → `global._updateProps`（C++ JSI）；JS props → `scheduleOnRN(updateJSProps, …)`。
3. C++ `_updateProps` → `AnimatedPropsRegistry::update` → 按 `ShadowNodeFamily` 累积 props。
4. 每帧 `ReanimatedModuleProxy::performOperations()`（`:785-836`）把 registry 汇成 `UpdatesBatch`，然后二选一：
   - **同步直改路径**（feature flag）：非布局属性（opacity、transform…）→ iOS `synchronouslyUpdateViewOnUIThread` / Android `FabricUIManager.synchronouslyUpdateViewOnUIThread`（int/double 序列化缓冲区喂入）。
   - **ShadowTree 路径**：`cloneShadowTreeWithNewProps` + `commit`，打 `ReanimatedCommitTrait` 标记；React 自己的 commit 会被 `ReanimatedCommitHook::shadowTreeWillCommit` 合并动画 props（避免互相覆盖）。

> 注意：新的 `USE_ANIMATION_BACKEND` 静态开关会把 rAF 与落地都改道到 RN ≥ 0.85 的 `UIManagerAnimationBackend`；上面描述的是默认经典路径。

---

## 4. LayoutAnimations 拦截架构

> 此版本**仅支持 New Architecture（Fabric）**，旧架构代码（`REAUIManager`、Android `layoutReanimation` Java 包）已删除。仅剩的双实现是 Fabric 内部的双 proxy，由静态开关 `ENABLE_SHARED_ELEMENT_TRANSITIONS`（默认 `false`）选择：`LayoutAnimationsProxy_Legacy`（默认）/ `LayoutAnimationsProxy_Experimental`（加共享元素转场）。

**设计不变量**（Legacy proxy 注释，`:19-22`）：_"We never modify the Shadow Tree, we just send some additional mutations to the mounting layer."_——**从不改 ShadowTree，只向挂载层注入/暂扣 mutation**。

### 4.1 配置注册（JS/JSI 侧）**[确认]**

- `AnimatedComponent.native.tsx:355-361` `_configureLayoutAnimation` 在 mount/update/unmount 时调 `updateLayoutAnimations(viewTag, type, config)`。**键控细节：ENTERING 用 `this.reanimatedID`**（JS 计数器——因为 Fabric viewTag 此时还不存在），组件渲染 `nativeID={reanimatedID}` 让 native 侧事后重键（`transferConfigFromNativeID`）；EXITING/LAYOUT 用真 viewTag。
- 批处理后落到 `LayoutAnimationsManager::configureAnimationBatch`（`LayoutAnimationsManager.cpp:10-42`），存进 `enteringAnimations_` / `exitingAnimations_` / `layoutAnimations_` / `sharedTransitions_` 四张 map。类型枚举：ENTERING=1 / EXITING=2 / LAYOUT=3 / SHARED_ELEMENT_TRANSITION=4。
- `<LayoutAnimationConfig skipExiting>` → `setShouldAnimateExitingForTag`。

### 4.2 拦截点：MountingOverrideDelegate **[确认]**

`ReanimatedCommitHook::maybeInitializeLayoutAnimations`（`ReanimatedCommitHook.cpp:35-44`）对每个 surface：

```cpp
layoutAnimationsProxy_->startSurface(shadowTree.getSurfaceId());
shadowTree.getMountingCoordinator()->setMountingOverrideDelegate(layoutAnimationsProxy_);
```

**这是唯一拦截点**：两个 proxy 都继承 `facebook::react::MountingOverrideDelegate`，RN 的 `MountingCoordinator::pullTransaction` 把该 surface 的每笔事务委托给 proxy 的 `pullTransaction`。

### 4.3 `pullTransaction` 内的 mutation 拦截管线 **[确认]**

Legacy proxy 每笔事务：

1. `reconcileContradictedRemovals` —— React 重建了正被暂扣删除的 tag 时，立即冲掉旧 Remove/Delete。
2. `addOngoingAnimations` —— 为进行中的动画注入 Update mutation（见 §4.5）。
3. `parseRemoveMutations` —— 区分**移动**（Remove 无配对 Delete → `MOVED`）与**删除**（Remove + Delete 同 commit）。
4. `handleRemovals` → `startAnimationsRecursively` —— 有 EXITING 配置的节点标记 ANIMATING、启动离场动画、**暂扣 Remove/Delete**（节点保持挂载直到动画结束）；无离场动画的子树立即删除。RNScreens 的 pop 场景有专门抑制。
5. `handleUpdatesAndEnterings`：
   - **Entering（Insert）**：重键配置；有 ENTERING 配置 → 启动进场动画，转发 Insert，追加一个 **opacity-0 的 Update**（`cloneViewWithoutOpacity`）防首帧闪烁。
   - **Layout（Update）**：有 LAYOUT 配置且 frame 真变了 → **吞掉该 Update**（宿主视图保持旧 frame），记录 `finalView`/`currentView` 进 `layoutAnimations_[tag]`；若只是进行中动画的目标变了，`updateOngoingAnimationTarget` 重定向。

Experimental proxy 换成维护一棵持久 **light tree**（`lightNodes_`），分类逻辑相同，差异：exiting 节点会被**重新插到父节点末尾**（zIndex 考量），死节点在后续事务中统一清理；并多了共享元素转场的整套处理（`SharedTransitions.cpp`，由 RNScreens 的 `onTransitionProgress` 事件喂入）。

### 4.4 启动动画（config → UI runtime）**[确认]**

三个 `start*Animation` 同构：

1. `scheduleOnUI` 跳到 UI 线程（Android 注意：`pullTransaction` 可能跑在 JS 线程，启动经 `pendingStarts_` 代际句柄防丢失取消）。
2. 注册 `LayoutAnimation{finalView, currentView, startView, parentTag, opacity, count}`。
3. 对 Fabric `layoutMetrics.frame` 拍 `Snapshot`，构造 JSI 对象 `yogaValues`（current/target 的 originX/Y + width/height + windowWidth/Height）。
4. `LayoutAnimationsManager::startLayoutAnimation(rt, tag, type, values)` 取出序列化配置，调进 UI runtime JS：`global.LayoutAnimationsManager.start(tag, type, values, config)`（`src/layoutReanimation/animationsManager.native.ts:53-137`）：用户 config 函数（吃 yogaValues）→ `initialValues` + `animations` → 包 `withStyleAnimation` → **赋给一个 UI 侧 Mutable**。同时注册 SV listener（`tag + 1e9`）：每帧值变 → `global._notifyAboutProgress(tag, value)` + `scheduleFlush()`；结束 → `_notifyAboutEnd(tag, removeView = (type===EXITING))`。

### 4.5 帧进度如何写回视图 **[确认]**

进度计算在 UI runtime（§3.4 的 valueSetter 机制），写回路径：

1. `_notifyAboutProgress(tag, newStyle)`（JSI host 函数，"Always on UI thread"）→ proxy 的 `progressLayoutAnimation`：clone final view 的 props + 动画样式，解析出 `Frame{originX, originY, width, height}`，存进 `surfaceManager.getUpdateMap(surfaceId)[tag]`。
2. 帧末 `executeLayoutAnimationsRequests` → `shadowTree.notifyDelegatesOfUpdates()` —— **强制 RN 再拉一笔挂载事务**（即使树没变）。
3. 这次 pull 重新进 proxy，`addOngoingAnimations` 把 update map 倒成 `ShadowViewMutation::UpdateMutation`，`updateLayoutMetrics` 把动画中的 origin/size 写进 LayoutMetrics → **走正常 Fabric 挂载路径落地**。

即：**布局动画 = 每帧在 UI 线程 JS 算进度，同步注入 Update mutation（props + LayoutMetrics），经常规 Fabric 挂载落地。proxy 只注入 mutation，从不动 ShadowTree。**

### 4.6 三件套触发与清理速查

| 类型     | mutation 级触发                                                       | 起始值                | 清理                                                                |
| -------- | --------------------------------------------------------------------- | --------------------- | ------------------------------------------------------------------- |
| ENTERING | `Insert` + entering 配置；视图立即插入但 opacity-0 Update 藏到首帧    | target rect           | `endLayoutAnimation(shouldRemove=false)`                            |
| LAYOUT   | `Update` 且 frame 变了 + layout 配置；Update 被吞，视图停旧 frame     | current + target rect | 同上；`updateOngoingAnimationTarget` 可重定向                       |
| EXITING  | `Remove` + `Delete` 同 commit（真卸载，非移动）；Remove/Delete 被暂扣 | current rect          | JS 完成 → `_notifyAboutEnd(tag, true)` → 下笔事务补发 Remove+Delete |

取消：`maybeCancelAnimation` → `global.LayoutAnimationsManager.stop(tag)` → SharedValue `cancelAnimation`。竞态：`reconcileContradictedRemovals` 覆盖「动画中 React 重建同 tag」。

---

## 5. 对 Lynx 的可移植性总判 **[初判]**

> 待 R2（MTS 能力）、R4（布局观测）校准；以下是基于 R3 单方事实的第一印象。

| 机制                 | 可移植性                                 | 依据                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Worklet runtime**  | **能照搬**（思想 + 大部分工程形态）      | ReactLynx 已有同构实现：`registerWorklet(hash, fn)` + `'main thread'` 指令（SWC 插件，`packages/react/transform/src/swc_plugin_worklet_post_process/mod.rs`）+ `runOnMainThread`（`packages/react/runtime/src/snapshot/worklet/call/runOnMainThread.ts`）。Lynx 的 MTS 运行时 ≈ Lynx 版 UI runtime（PrimJS on main thread）。**[推断]** 需补齐：`__closure` 隐式捕获、`__valueUnpacker` 惰性物化、每 runtime 缓存这套通用基建——Lynx 现有事件 worklet 更轻（服务事件/手势），「动画状态每帧在 MTS 侧推进」的链路要自建 |
| **SharedValue**      | **部分可移植，同步语义是关键未知**       | Reanimated 的读路径依赖 JSI 级 runtime 递归锁（JS 线程持锁内联执行）。Lynx MTS↔BTS 通信目前是消息式（`registerCallable` 句柄、`runOnMainThread`/`runOnBackgroundThread`，见 R1 发现）。有无锁式同步调用等价物，R2 必须回答；没有则退化为「消息式 + 本地缓存 + 脏标记」，语义弱一档但可用                                                                                                                                                                                                                              |
| **Frame driving**    | **大概率可行**                           | R1 已确认 mini 版在 MTS 用 rAF + Date.now() 跑动画循环——说明主线程存在某种帧回调（具体语义 R2 确认）。另有 R1 发现的原生 PAPI `__ElementAnimate`（引擎驱动、WAAPI 风格）作为备选引擎侧路径                                                                                                                                                                                                                                                                                                                            |
| **LayoutAnimations** | **机制思想可移植，拦截点位置是最大未知** | Reanimated 的拦截点在 `MountingOverrideDelegate`（引擎挂载层）。Lynx 的对应层：ReactLynx snapshot/commit 管线（JS 侧可拦截？）vs 引擎内部（需要引擎侧新能力）。R4 的布局观测事实直接决定三件套落在哪一层                                                                                                                                                                                                                                                                                                              |
| **手势/滚动驱动**    | 待 R5                                    | —                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

**一句话**：Reanimated 的架构思想与 Lynx 双线程模型高度同构（MTS ≈ UI runtime，BTS ≈ RN runtime），worklet 链路 Lynx 已有雏形；**两个硬骨头**是 ① SharedValue 的同步读语义在 Lynx 消息式线程模型上如何近似，② 布局动画的 mutation 拦截点在 Lynx 技术栈的哪一层。

---

## 附：关键源码索引

- worklets Babel 插件：`packages/react-native-worklets/plugin/src/`（`plugin.ts` / `workletFactory.ts` / `closure.ts` / `workletStringCode.ts`）
- Serializable/Shareable：`packages/react-native-worklets/Common/cpp/worklets/SharedItems/`
- UI runtime：`packages/react-native-worklets/Common/cpp/worklets/WorkletRuntime/`
- 帧批处理：`packages/react-native-worklets/Common/cpp/worklets/AnimationFrameQueue/AnimationFrameBatchinator.cpp` + `src/runLoop/uiRuntime/requestAnimationFrame.ts`
- Mutable：`packages/react-native-reanimated/src/mutables.native.ts` + `mutablesCommon.ts`
- valueSetter：`packages/react-native-reanimated/src/valueSetter.ts`
- LayoutAnimations proxy：`packages/react-native-reanimated/Common/cpp/reanimated/LayoutAnimations/LayoutAnimationsProxy_{Legacy,Experimental}.cpp`
- 布局动画 JS 管理器：`packages/react-native-reanimated/src/layoutReanimation/animationsManager.native.ts`
