# Android 高频面试知识点（资深级/架构师级）- 深度解析版

本文档为 Android 开发面试核心知识点的深度扩展版，旨在从底层原理、源码实现和架构思考三个维度提供专业解答。

---

## 1. 核心四大组件与系统原理

### 1.1 Activity 启动流程深度剖析 (Android 14/15)
启动 Activity 不仅仅是一个方法调用，它是**跨进程协作 (IPC)** 的巅峰体现。

*   **App 进程发起阶段**：
    *   `ContextImpl.startActivity` -> `Instrumentation.execStartActivity`。
    *   **关键点**：`Instrumentation` 是 App 的大管家，它通过 `ActivityTaskManager.getService()` 获取到 SystemServer 进程中 `ATMS` 的 Binder 代理对象。
*   **SystemServer 调度阶段 (核心)**：
    *   `ActivityTaskManagerService (ATMS)`：处理启动请求的核心入口。
    *   `ActivityStarter`：负责解析 Intent（隐式匹配）、校验权限、根据 **LaunchMode** 和 **Flags** 计算最终的目标 TaskStack。
    *   `ActivityTaskSupervisor`：管理所有的 Task。如果目标进程未启动，则调用 `AMS.startProcessLocked`。
    *   **冷启动关键**：SystemServer 通过 **Socket** 向 `Zygote` 进程发送创建新进程的请求。Zygote fork 出新进程后，反射调用 `ActivityThread.main()`。
*   **App 进程落地阶段 (现代架构)**：
    *   **ClientTransaction**：Android 9.0 引入，将多个生命周期请求合并。
    *   `ActivityThread$H`：主线程 Handler。收到 `EXECUTE_TRANSACTION` 消息。
    *   `TransactionExecutor`：执行器。它会先执行“当前状态到目标状态”的所有中间项（如从 Stop 到 Resume，需经历 Restart, Start）。
    *   **最终落地**：`mInstrumentation.callActivityOnCreate` -> `activity.performCreate` -> `setContentView` (触发 DecorView 初始化)。

### 1.2 Handler 机制：不仅仅是消息传递
*   **Looper 的死循环为什么不卡死？**
    *   底层使用了 Linux 的 **epoll 机制**。当 `MessageQueue` 中没有消息时，主线程会释放 CPU 资源进入休眠状态（`nativePollOnce`）；当有新消息或 Vsync 信号到来时，会通过文件描述符 (fd) 唤醒线程。
*   **内存屏障 (Sync Barrier)**：
    *   通过 `postSyncBarrier` 发送一个 `target == null` 的特殊消息。此后，`MessageQueue` 只会处理 `isAsynchronous == true` 的异步消息（如 UI 刷新信号）。这是 UI 流畅度的核心保障。
*   **ThreadLocal 的作用**：
    *   每个线程拥有独立的 `Looper` 实例。`ThreadLocal` 内部维护 `Values` 数组，以 `ThreadLocal` 对象为 Key，实现线程间的数据隔离。

### 1.3 Binder 机制：Android 的脊梁
*   **为什么快？**
    *   传统 IPC（如管道、Socket）需要两次拷贝：App1 -> 内核缓存 -> App2。
    *   **Binder 只有一次拷贝**：通过 `mmap` 在内核空间和接收方用户空间建立映射。发送方只需将数据拷贝到内核缓存区，接收方即可直接读取。
*   **Binder 内存限制**：
    *   普通进程的 Binder 缓冲区约为 **1MB - 8KB**（由 `ProcessState` 决定）。这就是为什么 `Intent` 传大数据会抛出 `TransactionTooLargeException`。
*   **AIDL 编译产物**：
    *   `Proxy`：运行在客户端，负责数据序列化并调用 `mRemote.transact()`。
    *   `Stub`：运行在服务端，本质是一个 `Binder` 对象，在 `onTransact()` 中解包并调用真正的业务代码。

---

## 2. 界面与 UI 渲染深度解析

### 2.1 UI 渲染管线 (Choreographer)
*   **Vsync 信号**：由硬件发出（通常 60Hz/120Hz）。
*   **Choreographer 职责**：
    *   它向 `SurfaceFlinger` 订阅下一个 Vsync 信号。
    *   信号到达时，按序触发：**CALLBACK_INPUT** (输入事件) -> **CALLBACK_ANIMATION** (动画) -> **CALLBACK_TRAVERSAL** (绘制)。
    *   `performTraversals` 内部依次触发 `measure`, `layout`, `draw`。

### 2.2 事件分发与滑动冲突
*   **核心链条**：`DecorView` -> `ViewGroup` -> `View`。
*   **消费逻辑**：如果 `onTouchEvent` 返回 false，事件会向上回溯给父容器的 `onTouchEvent`。
*   **冲突解决策略**：
    *   **外部拦截**：父容器在 `onInterceptTouchEvent` 中判断是否需要拦截（如左右滑动还是上下滑动）。
    *   **内部拦截**：子 View 在 `dispatchTouchEvent` 中调用 `requestDisallowInterceptTouchEvent(true)`，强行禁止父容器拦截。

---

## 3. 性能优化：资深开发的核心竞争力

### 3.1 内存优化与 LeakCanary 原理
*   **LeakCanary 原理**：
    *   利用 `WeakReference` 引用 Activity/Fragment。
    *   在生命周期销毁后，观察 `ReferenceQueue` 是否收到了该引用。
    *   如果没收到，主动触发一次 GC，若还没收到，则判定为泄漏，并使用 **Shark** 引擎分析 Hprof 文件，查找最短引用链。
*   **Bitmap 管理**：
    *   Android 8.0+：Bitmap 像素数据存储在 **Native 堆**，不再占用 JVM 堆，大大减少了 OOM 风险。

### 3.2 ANR 分析与避免
*   **触发原因**：Input (5s)、Broadcast (前台10s/后台60s)、Service (前台20s/后台200s)。
*   **监控逻辑**：AMS 在启动这些任务时会发送一个延时消息。如果任务按时完成，则移除消息；否则，消息触发，Dump 出 `/data/anr/traces.txt`。
*   **解决**：严禁在主线程进行 I/O、数据库操作或耗时循环。

---

## 4. 架构设计与高级组件

### 4.1 ViewModel 的生命力
*   **为什么旋转屏幕不销毁？**
    *   `ViewModel` 存储在 `ViewModelStore` 中。
    *   在 Activity 配置变更时，系统会保留 `NonConfigurationInstances` 对象。新 Activity 启动后，会从这个对象中重新找回之前的 `ViewModelStore`。

### 4.2 协程 (Coroutines) 与异步
*   **挂起 (Suspend) 的本质**：
    *   不是阻塞线程，而是**暂存当前状态 (Continuation)**，释放线程执行权。等到异步结果返回，通过 `Continuation.resume` 恢复执行。
    *   **Dispatcher**：本质是线程池。`Dispatchers.Main` 通过 Handler 将任务 post 到主线程。

### 4.3 Retrofit 原理深度解析
Retrofit 的核心思想是：**将复杂的网络请求抽象为 Java 接口，通过动态代理和装饰者模式简化调用。**

*   **动态代理 (Dynamic Proxy)**：
    *   调用 `retrofit.create(IService.class)` 时，内部利用 `Proxy.newProxyInstance` 生成接口的代理实现。
    *   当调用接口方法时，会触发代理对象的 `InvocationHandler.invoke()`，将方法调用转化为一个 `ServiceMethod` 对象。
*   **解析与缓存 (ServiceMethod)**：
    *   Retrofit 使用**反射**解析接口方法上的注解（@GET, @POST, @Query 等）。
    *   为了性能，解析后的结果会被封装成 `HttpServiceMethod` 并缓存到 `Map` 中，确保同一个方法只解析一次。
*   **适配器模式 (CallAdapter)**：
    *   `CallAdapter` 负责将默认的 `OkHttpCall` 转换为用户期望的返回类型（如 `Observable`, `Deferred`, `Flow` 或 `Call`）。
    *   这使得 Retrofit 能完美兼容 RxJava、协程等多种异步框架。
*   **转换器模式 (Converter)**：
    *   `Converter` 负责数据的序列化与反序列化（如 `GsonConverterFactory`）。
    *   它将 HTTP 响应体 (ResponseBody) 转换为 Java 对象，或将请求对象转换为 RequestBody。
*   **工作流总结**：
    1.  动态代理拦截方法调用。
    2.  解析注解并构建 `Request`。
    3.  通过 `OkHttpClient` 发送真实请求。
    4.  通过 `Converter` 转换数据。
    5.  通过 `CallAdapter` 适配并返回结果。

---

## 5. 跨端与前沿技术

### 5.1 KMP (Kotlin Multiplatform)
*   **共享机制**：通过 `commonMain` 编写通用逻辑，使用 `expect/actual` 关键字在 `androidMain` 和 `iosMain` 中调用平台特有 API。
*   **编译原理**：Android 编译为 JVM Bytecode，iOS 通过 Kotlin/Native 编译为 LLVM IR，最终生成 Mach-O 二进制文件。

### 5.2 Jetpack Compose
*   **组合 (Composition)**：根据状态构建 UI 树。
*   **重组 (Recomposition)**：当 `State` 变化时，Compose 会智能地只更新受影响的代码块（基于插槽表 SlotTable 记录的位置）。

---
*Generated by CineWave AI Assistant - Specialized for Senior Android Engineers*
