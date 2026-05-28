# Android 技能知识百科全书

> 作者：10年资深 Android 架构师
> 说明：这是一份"打开即用"的知识体系，每个知识点都包含核心原理、源码分析、常见问题与最佳实践，无需再去网上搜索资料。
>
> **版本说明**：本文档持续跟进 Android 官方最新推荐实践。当前覆盖 Android 15 (API 35) 及 Jetpack 最新稳定版。

---

## 一、基础篇

### 1.1 Java 核心

#### 1.1.1 HashMap 源码与原理

**核心原理**
- 数据结构：数组 + 链表 + 红黑树（JDK 1.8+）
- 默认容量：16，负载因子：0.75
- 扩容阈值：capacity * loadFactor，超过则扩容为 2 倍

**put 流程**
1. 计算 key 的 hash 值：`(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)`
2. 定位桶位置：`(n - 1) & hash`
3. 如果桶为空，直接插入
4. 如果桶不为空，遍历链表/红黑树
   - 找到相同 key 则覆盖 value
   - 没找到则尾插法插入
5. 链表长度 >= 8 且数组长度 >= 64，转为红黑树
6. 检查是否需要扩容

**为什么引入红黑树？**
- 链表查找 O(n)，红黑树查找 O(log n)
- 当 hash 冲突严重时，防止链表过长导致性能退化
- 为什么是 8？泊松分布计算，链表长度达到 8 的概率极低（约 0.00000006），是一种安全防护
- 为什么不是 6？红黑树转回链表的阈值是 6，留出 1 的缓冲避免频繁转换

**扩容机制**
- 新数组 = 旧数组 * 2
- 元素重新分配：要么在原位置，要么在原位置 + 旧容量
- 判断依据：`(e.hash & oldCap) == 0`

**常见问题**
- HashMap 线程不安全：并发 put 可能导致死循环（JDK 1.7 头插法）或数据丢失
- 解决方案：ConcurrentHashMap 或 Collections.synchronizedMap
- 为什么容量是 2 的幂？因为 `(n - 1) & hash` 等价于 `hash % n`，且位运算更快

#### 1.1.2 ConcurrentHashMap 原理

**JDK 1.7 分段锁**
- 数据结构：Segment 数组 + HashEntry 数组
- Segment 继承 ReentrantLock
- 并发度：默认 16，即最多 16 个线程同时写

**JDK 1.8 CAS + synchronized**
- 放弃分段锁，使用 Node 数组 + 链表/红黑树
- 写入时使用 CAS 尝试，失败则 synchronized 锁住链表头节点
- 并发度更高，锁粒度更细

**关键方法**
- `size()`：通过 sumCount() 累加 CounterCell 数组的值
- `transfer()`：多线程协助扩容，每个线程负责一部分桶

#### 1.1.3 动态代理与 Retrofit

**动态代理原理**
```java
// Proxy.newProxyInstance 创建代理对象
public static Object newProxyInstance(ClassLoader loader, 
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
```
- 运行时动态生成代理类字节码
- 所有方法调用都会转发到 InvocationHandler.invoke()

**Retrofit 如何用动态代理？**
1. 定义接口：`interface ApiService { @GET("users") suspend fun getUsers(): List<User> }`
2. Retrofit.create(ApiService.class) 时，通过动态代理创建接口实现
3. 每次调用接口方法，InvocationHandler 解析方法上的注解（@GET/@POST/@Path 等）
4. 构建 OkHttp Call 对象
5. 通过 CallAdapter 适配返回值（协程/RxJava/Call）

**为什么 Retrofit 用接口而不是抽象类？**
- 动态代理只能代理接口，不能代理类
- 接口天然支持多继承，更灵活

---

### 1.2 Kotlin 核心

#### 1.2.1 协程原理深度解析

**协程是什么？**
- 协程不是线程，是"可挂起的计算"
- 线程由操作系统调度，协程由编译器调度
- 挂起不阻塞线程，类似"暂停-恢复"

**挂起函数的秘密（CPS 变换）**
```kotlin
// 你写的代码
suspend fun fetchUser(): User {
    val response = api.getUser()  // 挂起点
    return response.data
}

// 编译后的代码（简化）
fun fetchUser(continuation: Continuation<User>): Any? {
    // 状态机实现
    when (continuation.label) {
        0 -> {
            continuation.label = 1
            if (api.getUser(continuation) == COROUTINE_SUSPENDED) 
                return COROUTINE_SUSPENDED
        }
        1 -> {
            val response = continuation.result as Response
            return response.data
        }
    }
}
```
- 每个挂起函数被编译成一个状态机
- Continuation 对象保存了挂起时的上下文和回调
- 返回值 Object 可能是结果，也可能是 COROUTINE_SUSPENDED 标记

**协程调度器**
- Dispatchers.Main：Android 主线程，UI 操作
- Dispatchers.IO：64 线程池（默认），适合网络/文件/数据库
- Dispatchers.Default：CPU 核心数线程池（最少 2 个），适合计算密集型
- Dispatchers.Unconfined：不切换线程，极少用

**协程异常处理**
```kotlin
// 方式一：try-catch 包裹
viewModelScope.launch {
    try {
        val data = repository.fetchData()
    } catch (e: Exception) {
        // 处理异常
    }
}

// 方式二：CoroutineExceptionHandler
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("TAG", "Caught $exception")
}

// 方式三：SupervisorScope 防止异常传播
// viewModelScope 内部已经是 SupervisorJob，无需额外创建
viewModelScope.launch(handler) {
    // 如果需要隔离多个子协程：
    supervisorScope {
        launch { /* 子协程1 */ }
        launch { /* 子协程2，失败不影响1 */ }
    }
}
```

**常见问题**
- `withContext(Dispatchers.IO)` 和 `launch(Dispatchers.IO)` 区别？
  - withContext 是挂起函数，等待执行完毕返回结果
  - launch 是启动协程，不等待结果
- `viewModelScope` 为什么用 SupervisorJob？
  - 一个子协程失败不影响其他子协程
  - 适合 UI 层，一个请求失败不应取消所有请求

#### 1.2.2 Flow 详解

**冷流 vs 热流**
- 冷流（Flow）：每次 collect 都重新执行，数据生产 = 消费开始
- 热流（StateFlow/SharedFlow）：多个 collect 共享数据，数据生产独立于消费

**StateFlow 与 LiveData 对比**
| 特性 | StateFlow | LiveData |
|------|-----------|----------|
| 初始值 | 必须提供 | 可选 |
| 线程安全 | 是 | 是（主线程通知） |
| 粘性事件 | 有 | 有 |
| 组合/变换 | 丰富操作符 | map/switchMap |
| 协程支持 | 原生 | 需转换 |
| 单元测试 | 简单 | 需 InstantTaskExecutorRule |
| 数据倒灌 | 有（需额外处理） | 有（粘性事件特性） |

**SharedFlow 配置**
```kotlin
// replay：新订阅者能收到最近 N 条历史数据
// extraBufferCapacity：缓冲区大小，超出则根据 onBufferOverflow 策略处理
// onBufferOverflow：BufferOverflow.SUSPEND/DROP_OLDEST/DROP_LATEST
val sharedFlow = MutableSharedFlow<Int>(
    replay = 1,
    extraBufferCapacity = 0,
    onBufferOverflow = BufferOverflow.SUSPEND
)
```

---

### 1.3 四大组件

#### 1.3.1 Activity 生命周期深度解析

**完整生命周期**
```
onCreate     → 初始化、setContentView、绑定 ViewModel
onStart      → 界面可见但不可交互
onResume     → 获取焦点，开始交互
onPause      → 失去焦点，轻量操作（暂停动画、释放独占资源）
onStop       → 完全不可见，释放用户级资源
onDestroy    → 释放所有资源、ViewModel.onCleared()
```

**屏幕旋转发生了什么？**
```
onPause → onStop → onSaveInstanceState → onDestroy 
→ onCreate → onStart → onRestoreInstanceState → onResume
```
- Activity 被销毁重建
- onSaveInstanceState 保存临时数据到 Bundle
- onRestoreInstanceState 恢复数据（也可以在 onCreate 中恢复）

**什么数据应该存 onSaveInstanceState？**
- 用户输入的表单数据
- 当前滚动位置
- 选中的 Tab/页面状态
- **不适合**：大量数据（序列化耗时）、网络请求结果（应走 ViewModel 缓存）

**启动模式详解**
- standard：默认，每次创建新实例
- singleTop：如果栈顶是此 Activity，复用；否则创建新实例
- singleTask：如果栈中有此 Activity，清除其上所有 Activity 并复用
- singleInstance：独立 Task，此 Activity 独占一个 Task

**taskAffinity 的作用**
- 默认与包名相同
- singleTask 启动时，如果指定了不同的 taskAffinity，会在对应 Task 中创建
- allowTaskReparenting：Activity 从后台 Task 移动到前台 Task

#### 1.3.2 Service 与后台限制

**Android 8+ 后台限制**
- 后台启动 Service 被禁止
- 替代方案：JobIntentService（Android 8+ 自动转为 JobScheduler 执行）
- 前台服务必须在 5 秒内显示通知

**Android 14 前台服务类型**
- 必须声明前台服务类型：`android:foregroundServiceType="camera|microphone|location|..."`
- 新增 shortService：短时间后台任务，无需通知
- 违反限制会被系统杀死

**Binder 通信原理**
- 客户端通过 Binder 驱动调用服务端方法
- 数据通过 Parcel 序列化
- 一次拷贝（mmap 共享内存）提高性能

---

#### 1.3.3 Notification 通知系统

Android 8.0+ 引入了通知渠道（Notification Channel），所有通知必须归属到某个渠道，用户可按渠道精细控制通知行为。

**创建通知渠道**

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        createNotificationChannels()
    }

    private fun createNotificationChannels() {
        val channel = NotificationChannel(
            CHANNEL_ID_MESSAGES,
            "消息通知",
            NotificationManager.IMPORTANCE_HIGH  // 重要性决定展示方式
        ).apply {
            description = "用于显示聊天消息通知"
            vibrationPattern = longArrayOf(0, 200, 100, 300)  // 震动模式：等待/震动/等待/震动
            lockscreenVisibility = Notification.VISIBILITY_PRIVATE
        }
        val manager = getSystemService(NotificationManager::class.java)
        manager.createNotificationChannel(channel)
    }
}
```

**重要性等级与行为**

| 级别 | 常量 | 行为 |
|------|------|------|
| 紧急 | `IMPORTANCE_HIGH` | 弹出横幅 + 声音 + 震动 |
| 高 | `IMPORTANCE_DEFAULT` | 声音 |
| 低 | `IMPORTANCE_LOW` | 无声音，状态栏显示 |
| 最低 | `IMPORTANCE_MIN` | 静默，不显示图标 |
| 无 | `IMPORTANCE_NONE` | 完全屏蔽 |

用户可在系统设置中修改每个渠道的重要性，应用读取到的 `channel.importance` 为实际生效值。

**发送通知**

```kotlin
val notification = NotificationCompat.Builder(this, CHANNEL_ID_MESSAGES)
    .setSmallIcon(R.drawable.ic_notification)
    .setContentTitle("新消息")
    .setContentText("张三：你好")
    .setPriority(NotificationCompat.PRIORITY_HIGH)
    .setAutoCancel(true)           // 点击后自动消失
    .setContentIntent(pendingIntent)  // 点击跳转
    .addAction(R.drawable.ic_reply, "回复", replyPendingIntent)
    .build()

NotificationManagerCompat.from(this).notify(notificationId, notification)
```

**Android 13+ 运行时权限**

```kotlin
// Android 13+ 发送通知前需请求权限
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    requestPermissions.launch(Manifest.permission.POST_NOTIFICATIONS)
}
```

**FCM 推送基础**

```
App Server → FCM Server → Android Device
                    ↓
            系统级长连接（无需应用保活）
                    ↓
         BroadcastReceiver/FirebaseMessagingService
                    ↓
              显示通知/处理数据
```

```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {
    override fun onMessageReceived(message: RemoteMessage) {
        // 前台时自定义处理，后台时系统自动显示通知
        if (isAppInForeground()) {
            showCustomNotification(message)
        }
    }

    override fun onNewToken(token: String) {
        // 令牌刷新，上报到服务器
        sendTokenToServer(token)
    }
}
```

**通知分组（Android 7+）**

```kotlin
// 同组通知折叠显示
NotificationCompat.Builder(this, channelId)
    .setGroup("chat_group")
    .setGroupSummary(true)  // 分组摘要
    .build()
```

**Bubble 气泡通知（Android 11+）**

```kotlin
// 类似 Facebook Messenger 的浮动气泡
val bubbleIntent = NotificationCompat.BubbleMetadata.Builder(
    bubblePendingIntent, Icon.createWithResource(this, R.drawable.ic_chat)
).build()
NotificationCompat.Builder(this, channelId)
    .setBubbleMetadata(bubbleIntent)
    .build()
```

---

#### 1.3.4 SplashScreen API 与预测性返回手势

**SplashScreen API（Android 12+ 强制）**

Android 12 起，系统为所有 Activity 自动注入启动画面。自定义仅需控制主题和退出动画：

```xml
<!-- themes.xml -->
<style name="Theme.MyApp.Splash" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/white</item>
    <item name="windowSplashScreenAnimatedIcon">@mipmap/ic_launcher</item>
    <item name="postSplashScreenTheme">@style/Theme.MyApp</item>
</style>

// Activity 中控制退出时机
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        installSplashScreen().apply {
            setKeepOnScreenCondition { viewModel.isLoading }  // 加载完成前保持 Splash
        }
        super.onCreate(savedInstanceState)
        setContent { ... }
    }
}
```

- 不再需要自定义 Splash Activity
- `setKeepOnScreenCondition` 控制 Splash 显示时长，适合等数据加载

**预测性返回手势（Android 14+）**

Android 14 引入预测性返回（Predictive Back Gesture），用户侧滑返回时展示透明预览，让用户知道即将返回到哪个页面。

```kotlin
// 在 AndroidManifest.xml 中启用（Android 15 起默认启用）
<application android:enableOnBackInvokedCallback="true">

// Activity 中处理
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Android 14+ 使用 OnBackInvokedCallback 替代 onBackPressed
        onBackInvokedDispatcher.registerOnBackInvokedCallback(
            OnBackInvokedDispatcher.PRIORITY_DEFAULT
        ) {
            // 自定义返回逻辑
            if (hasUnsavedChanges()) {
                showDiscardDialog()
            } else {
                finish()
            }
        }
    }
}
```

**Compose 中适配预测性返回**

```kotlin
@Composable
fun PredictiveBackScreen() {
    BackHandler(enabled = hasUnsavedChanges) {
        showDiscardDialog = true
    }
}
```

---

### 1.4 UI 基础

#### 1.4.1 RecyclerView 核心原理

**缓存层级（四级缓存）**
```
mAttachedScrap       → 屏幕内的 ViewHolder，无需重新绑定
mChangedScrap        → 数据变化的 ViewHolder，需重新绑定
mCachedViews         → 屏幕外的 ViewHolder，默认缓存 2 个，无需重新绑定
mRecyclerViewPool    → 共享池，需重新绑定，可配置最大数量
```

**DiffUtil 原理**
- 异步计算两个列表的差异
- 使用 Myers 差分算法
- 输出：更新/删除/插入/移动操作列表
- ListAdapter 封装了 DiffUtil + AsyncListDiffer，自动在后台线程计算

**性能优化**
- 固定 Item 高度：`setHasFixedSize(true)` 避免重新测量
- 预取机制：Prefetch 提前布局下一帧的 Item
- 嵌套滑动：NestedScrolling 机制解决 RecyclerView 嵌套滑动冲突

#### 1.4.2 ConstraintLayout 核心

**相对定位**
```xml
app:layout_constraintLeft_toLeftOf="parent"
app:layout_constraintRight_toRightOf="parent"
app:layout_constraintTop_toBottomOf="@id/textView"
```

**链（Chain）**
- spread：均匀分布
- spread_inside：两端贴边，中间均匀
- packed：打包在一起
- weighted：按权重分配空间（类似 LinearLayout weight）

**屏障（Barrier）**
- 多个 View 的动态边界对齐
- 适合"标签 + 内容"布局，标签宽度不确定时

**Guideline**
- 百分比定位辅助线
- `app:layout_constraintGuide_percent="0.5"` 50% 位置

---

## 二、中高级篇

### 2.1 自定义 View

#### 2.1.1 View 绘制流程

**measure 阶段**
```kotlin
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    val widthMode = MeasureSpec.getMode(widthMeasureSpec)
    val widthSize = MeasureSpec.getSize(widthMeasureSpec)
    
    when (widthMode) {
        MeasureSpec.EXACTLY -> // match_parent 或固定值，直接使用
        MeasureSpec.AT_MOST -> // wrap_content，计算实际大小，不超过 widthSize
        MeasureSpec.UNSPECIFIED -> // ScrollView 内部，自由测量
    }
    
    setMeasuredDimension(measuredWidth, measuredHeight)
}
```

**layout 阶段**
- 确定子 View 的位置：left/top/right/bottom
- 调用子 View 的 layout() 方法

**draw 阶段**
```
drawBackground(canvas)     → 绘制背景
onDraw(canvas)             → 绘制内容
dispatchDraw(canvas)       → 绘制子 View
onDrawForeground(canvas)   → 绘制前景（滚动条等）
```

**requestLayout vs invalidate**
- requestLayout：触发 measure + layout + draw，耗时
- invalidate：只触发 draw，轻量
- 尺寸变化用 requestLayout，外观变化用 invalidate

#### 2.1.2 事件分发机制

**分发流程**
```
Activity.dispatchTouchEvent()
  └── ViewGroup.dispatchTouchEvent()
        ├── onInterceptTouchEvent() → true 拦截
        │     └── onTouchEvent() 处理
        └── onInterceptTouchEvent() → false 不拦截
              └── 子 View.dispatchTouchEvent()
                    └── onTouchEvent() 消费
```

**核心原则**
1. 同一事件序列从 DOWN 开始，到 UP 结束
2. 一旦 View 消费了 DOWN，后续事件都交给它
3. ViewGroup 可以在 UP 时拦截（onInterceptTouchEvent 返回 true）
4. 子 View 可以通过 `requestDisallowInterceptTouchEvent(true)` 阻止父 View 拦截

**滑动冲突解决**
- 外部拦截法：父 View 在 onInterceptTouchEvent 中判断是否拦截
- 内部拦截法：子 View 通过 requestDisallowInterceptTouchEvent 控制

---

### 2.2 网络编程

#### 2.2.1 OkHttp 拦截器链

**拦截器类型**
```
应用拦截器（addInterceptor）
  ├── 重试/重定向拦截器
  ├── 桥接拦截器（添加请求头）
  ├── 缓存拦截器
  └── 连接拦截器
网络拦截器（addNetworkInterceptor）
```

**应用拦截器 vs 网络拦截器**
- 应用拦截器：只调用一次，无论重试多少次
- 网络拦截器：每次重试都调用，能看到真实网络请求
- 应用拦截器：能看到请求/响应头（包括默认添加的）
- 网络拦截器：能看到重定向和重试

**连接池复用**
- 默认最大空闲连接数：5
- 默认存活时间：5 分钟
- HTTP/2 多路复用：一个连接上同时发送多个请求

#### 2.2.2 Retrofit 源码分析

**动态代理创建**
```java
// Retrofit.create()
public <T> T create(final Class<T> service) {
    return (T) Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] { service },
        new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) {
                // 解析方法注解，构建 ServiceMethod
                ServiceMethod serviceMethod = loadServiceMethod(method);
                // 创建 OkHttpCall
                OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
                // 通过 CallAdapter 适配返回值
                return serviceMethod.callAdapter.adapt(okHttpCall);
            }
        }
    );
}
```

**自定义 Converter**
```kotlin
// 自定义 JSON 解析器
class GsonConverterFactory(private val gson: Gson) : Converter.Factory() {
    override fun responseBodyConverter(
        type: Type,
        annotations: Array<Annotation>,
        retrofit: Retrofit
    ): Converter<ResponseBody, *>? {
        return Converter { body -> gson.fromJson(body.charStream(), type) }
    }
}
```

---

### 2.3 Jetpack 核心

#### 2.3.1 Lifecycle 原理

**Lifecycle 状态机**
```
INITIALIZED → CREATED → STARTED → RESUMED
                  ↑          ↓          ↓
              DESTROYED ← STOPPED ← PAUSED
```

**LifecycleObserver 注册**
```kotlin
class MyObserver : DefaultLifecycleObserver {
    override fun onCreate(owner: LifecycleOwner) { }
    override fun onStart(owner: LifecycleOwner) { }
    override fun onResume(owner: LifecycleOwner) { }
    override fun onPause(owner: LifecycleOwner) { }
    override fun onStop(owner: LifecycleOwner) { }
    override fun onDestroy(owner: LifecycleOwner) { }
}
```

**ProcessLifecycleOwner**
- 监听整个 App 的前后台切换
- 所有 Activity 都不可见时触发 onStop
- 适合做全局的前后台状态判断

#### 2.3.2 ViewModel 原理

**为什么屏幕旋转不丢失数据？**
- ViewModelStore 存储在 Activity 的 NonConfigurationInstances 中
- 屏幕旋转时，Activity 重建但 ViewModelStore 被保留
- 新 Activity 通过 ViewModelProvider 获取同一个 ViewModel

**SavedStateHandle**
- 用于保存和恢复 UI 状态
- 内部使用 Bundle 存储
- 屏幕销毁重建时也能恢复数据

**ViewModel 作用域**
- Activity 作用域：Activity 销毁时清除
- Fragment 作用域：Fragment 销毁时清除
- Navigation 作用域：NavBackStackEntry 生命周期
- KMP 共享 ViewModel：通过 `androidx.lifecycle:lifecycle-viewmodel-compose` 在 Compose Multiplatform 中共享

#### 2.3.3 DataStore 替代 SharedPreferences

**为什么弃用 SharedPreferences？**
- SP 的 apply() 是异步写入但回调在主线程，commit() 是同步 IO 阻塞主线程
- SP 加载时全量读入内存，浪费内存
- SP 没有类型安全，key-value 容易拼写错误
- SP 在多进程下数据不一致

**Preferences DataStore**
```kotlin
val Context.dataStore by preferencesDataStore(name = "settings")

val exampleCounterFlow: Flow<Int> = context.dataStore.data
    .map { preferences ->
        preferences[EXAMPLE_COUNTER] ?: 0
    }

suspend fun incrementCounter(context: Context) {
    context.dataStore.edit { preferences ->
        val current = preferences[EXAMPLE_COUNTER] ?: 0
        preferences[EXAMPLE_COUNTER] = current + 1
    }
}
```

**Proto DataStore（类型安全）**
```kotlin
// 定义 .proto 文件
// 生成 UserSettings 类

val Context.userSettingsStore by protoDataStore<UserSettings>(
    fileName = "user_settings.pb",
    serializer = UserSettingsSerializer
)
```

#### 2.3.4 Room 数据库

> Room 是 Android 官方 ORM，构建在 SQLite 之上，提供编译时 SQL 校验、Flow 集成和自动迁移支持。

**核心注解**

```kotlin
@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "user_name") val name: String,
    val age: Int,
    @Embedded val address: Address  // 嵌入对象，字段平铺在同一张表
)

data class Address(val city: String, val street: String)

@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE age > :minAge")
    fun getUsersOlderThan(minAge: Int): Flow<List<User>>  // Flow 自动观察变化

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User): Long

    @Update
    suspend fun updateUser(user: User)

    @Delete
    suspend fun deleteUser(user: User)

    @Transaction
    @Query("SELECT * FROM users")
    fun getUsersWithOrders(): Flow<List<UserWithOrders>>  // 一对多关系
}

// 一对多关系
data class UserWithOrders(
    @Embedded val user: User,
    @Relation(parentColumn = "id", entityColumn = "user_id")
    val orders: List<Order>
)

@Database(entities = [User::class, Order::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

**为什么 Room 比原生 SQLite 好？**
- 编译时校验 SQL：写错字段名编译不过，不会运行时崩溃
- Flow 集成：`@Query` 返回 `Flow<List<T>>`，数据变化时自动发射新数据（通过 InvalidationTracker 监听表变更）
- 协程原生支持：`suspend` 函数自动在 `TransactionExecutor`（后台线程池）执行
- 无需手动游标管理、ContentValues、Cursor 转换

**数据库版本迁移**

```kotlin
// 方式一：自动迁移（Room 2.4+，推荐简单变更用）
@Database(
    entities = [User::class],
    version = 2,
    autoMigrations = [
        AutoMigration(from = 1, to = 2)
    ]
)

// 方式二：手动 Migration（复杂变更，如数据迁移改造）
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN email TEXT NOT NULL DEFAULT ''")
    }
}

// 方式三：破坏性迁移（开发阶段，数据不重要时）
.fallbackToDestructiveMigration()
```

**常见问题**
- **主线程访问**：Room 默认不允许主线程查询，需显式调用 `.allowMainThreadQueries()`（不推荐）
- **@Relation 懒加载陷阱**：`@Relation` 注解的字段只在查询时填充，不自动更新；KMP 中不支持 `@Relation`，需用 `@Query` + 多表 JOIN
- **Flow 触发频率**：InvalidationTracker 是表级监听，任意表变更都会触发表中所有 Flow 查询重新计算；高频写入需用 `distinctUntilChanged` 过滤
- **Schema 导出**：发布版必须导出 schema JSON 用于测试自动迁移，配置 `room.schemaLocation` 并纳入版本控制

**Room KMP（Kotlin 2.0+ / Room 2.7+）**

Room 已支持 Kotlin Multiplatform，可在 commonMain 中定义 Entity 和 DAO：

```kotlin
// commonMain
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

fun getRoomDatabase(builder: RoomDatabase.Builder<AppDatabase>): AppDatabase {
    return builder
        .setDriver(BundledSQLiteDriver())  // 跨平台 SQLite 驱动
        .build()
}
```
- Android 端用 `Room.databaseBuilder(context, AppDatabase::class.java, "app.db")`
- iOS 端用 `RoomDatabase.Builder` + `BundledSQLiteDriver`
- 限制：不支持 `@Relation`、`@DatabaseView` 等仅 Android 功能

---

#### 2.3.5 WorkManager 后台任务

**适用场景**
- 延迟执行的后台任务（上传日志、同步数据）
- 需要保证执行的任务（即使 App 被杀死）
- 约束条件执行（网络可用、充电状态、空闲状态）

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        return try {
            syncData()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

// 调度任务
val request = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
    .build()
WorkManager.getInstance(context).enqueue(request)
```

---

#### 2.3.6 Jetpack Navigation

Compose Navigation 已在 [7.1.4 Compose Navigation](#714-compose-navigation) 详述，这里聚焦 View 体系下的 Navigation。

**导航图定义（res/navigation/nav_graph.xml）**

```xml
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment">
        <argument
            android:name="userId"
            app:argType="string" />
        <action
            android:id="@+id/action_to_detail"
            app:destination="@id/detailFragment" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment">
        <argument
            android:name="itemId"
            app:argType="integer" />
        <deepLink app:uri="myapp://detail/{itemId}" />
    </fragment>
</navigation>
```

**Safe Args（类型安全传参）**

```kotlin
// 生成 HomeFragmentDirections 类，编译时校验参数类型
val action = HomeFragmentDirections.actionToDetail(itemId = 42)
findNavController().navigate(action)

// 目标 Fragment 接收参数
val args: DetailFragmentArgs by navArgs()
val itemId = args.itemId
```

**Deep Link 处理**

```kotlin
// 方式一：在导航图中声明（见上方 XML deepLink）
// 方式二：代码注册
// AndroidManifest.xml 中通过 <nav-graph> 引用，自动处理
```

**Navigation 作用域 ViewModel**

```kotlin
// ViewModel 与导航图节点绑定，返回时自动清除
class DetailFragment : Fragment() {
    private val viewModel: DetailViewModel by navGraphViewModels(R.id.detailFragment)
}
```

**常见问题**

- **返回栈管理**：`NavigationUI.setupActionBarWithNavController()` 自动处理返回按钮
- **跨导航图导航**：使用全局 action 或 `findNavController().navigate(R.id.otherGraph)`
- **Activity 结果回传**：使用 `savedStateHandle` 而非 `onActivityResult`

---

### 2.4 架构模式

#### 2.4.1 MVVM 架构详解

**分层结构**
```
View（Activity/Fragment/Composable）
  │ 观察 StateFlow/LiveData
  ▼
ViewModel
  │ 调用 Repository
  ▼
Repository
  │ 决定数据来源（网络/本地）
  ├── RemoteDataSource（网络）
  └── LocalDataSource（数据库/SP）
```

**单向数据流（UDF）**
```
Event → ViewModel → State → View
  │        │          │        │
  │        │          │        └── 渲染 UI
  │        │          └── 暴露给 View 观察
  │        └── 处理业务逻辑，更新 State
  └── 用户操作/系统事件
```

**依赖注入（Hilt）**
```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val repository: MainRepository
) : ViewModel() {
    val uiState: StateFlow<UiState> = ...
}
```

**Hilt 作用域与生命周期**

| 作用域 | 注解 | 生命周期 | 适用场景 |
|--------|------|---------|---------|
| Singleton | `@Singleton` | Application 生命周期 | 全局单例（API 客户端、数据库） |
| ViewModelScoped | `@ViewModelScoped` | ViewModel 存活期间 | ViewModel 及其依赖 |
| ActivityScoped | `@ActivityScoped` | Activity 存活期间 | Activity 级别的临时数据 |
| FragmentScoped | `@FragmentScoped` | Fragment 存活期间 | Fragment 级别的 UI 状态 |
| ServiceScoped | `@ServiceScoped` | Service 存活期间 | 前台/后台 Service 的依赖 |
| Unscoped | 无注解（默认） | 每次注入创建新实例 | 无状态工具类 |

```kotlin
@Module
@InstallIn(SingletonComponent::class)  // 安装到 Application 容器
object NetworkModule {
    @Singleton
    @Provides
    fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder().build()

    @Singleton
    @Provides
    fun provideRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
        .client(client)
        .baseUrl("https://api.example.com/")
        .build()
}

@Module
@InstallIn(ViewModelComponent::class)  // 安装到 ViewModel 容器
object RepositoryModule {
    @ViewModelScoped
    @Provides
    fun provideUserRepository(api: UserApi, db: UserDao): UserRepository =
        UserRepository(api, db)
}
```

**Qualifier：同类型多实例绑定**

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthOkHttp  // 带认证的 OkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class LoggingOkHttp  // 带日志的 OkHttpClient

@Module
@InstallIn(SingletonComponent::class)
object HttpClientModule {
    @AuthOkHttp
    @Singleton
    @Provides
    fun provideAuthClient(): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(AuthInterceptor())
        .build()

    @LoggingOkHttp
    @Singleton
    @Provides
    fun provideLoggingClient(): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(HttpLoggingInterceptor())
        .build()
}

// 使用
class MyRepository @Inject constructor(
    @AuthOkHttp private val authClient: OkHttpClient,
    @LoggingOkHttp private val loggingClient: OkHttpClient
)
```

**Compose 中的 Hilt**

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val viewModel: MainViewModel = hiltViewModel()  // Compose 内获取
            MainScreen(viewModel)
        }
    }
}
```

**Hilt vs Koin：如何选型**

| 维度 | Hilt | Koin |
|------|------|------|
| 实现方式 | 编译时注解处理器（APT），基于 Dagger | 运行时 DSL 注册 + 反射/ServiceLoader |
| 错误发现 | 编译时报错 | 运行时崩溃 |
| 性能 | 零运行时开销（生成代码） | 有轻微运行时开销 |
| 学习成本 | 高（注解、Component、Scope） | 低（纯 Kotlin DSL） |
| 适合规模 | 中大型项目 | 中小型项目、KMP 项目 |
| KMP 支持 | 有限（Android 为主） | 原生支持 KMP |
| 编译速度 | 较慢（注解处理） | 快 |

#### 2.4.2 MVI 架构详解

**核心概念**
```
Intent（用户意图）→ ViewModel（处理）→ State（状态）→ View（渲染）
                                              │
                                              └── Effect（一次性事件）
```

**与 MVVM 对比**
| 特性 | MVVM | MVI |
|------|------|-----|
| 状态管理 | 多个 LiveData/StateFlow | 单一不可变 State |
| 数据流 | 双向（用户操作 + 数据回调） | 单向（Intent → State） |
| 可回溯性 | 难 | 易（记录 State 历史） |
| 学习曲线 | 低 | 高 |
| 适合场景 | 简单/中等复杂度 | 复杂交互/需要状态回溯 |

**MVI 实现示例**
```kotlin
// 定义 Intent
sealed class MainIntent {
    data class LoadData(val id: String) : MainIntent()
    object Refresh : MainIntent()
}

// 定义 State
data class MainState(
    val isLoading: Boolean = false,
    val data: List<Item> = emptyList(),
    val error: String? = null
)

// 定义 Effect（一次性事件）
sealed class MainEffect {
    data class ShowSnackbar(val message: String) : MainEffect()
    object NavigateToDetail : MainEffect()
}

// ViewModel
class MainViewModel : ViewModel() {
    private val _state = MutableStateFlow(MainState())
    val state: StateFlow<MainState> = _state.asStateFlow()
    
    private val _effect = MutableSharedFlow<MainEffect>()
    val effect: SharedFlow<MainEffect> = _effect.asSharedFlow()
    
    fun processIntent(intent: MainIntent) {
        when (intent) {
            is MainIntent.LoadData -> loadData(intent.id)
            is MainIntent.Refresh -> refresh()
        }
    }
}
```

---

#### 2.4.3 Paging 3 分页加载

**核心组件**
```kotlin
// PagingSource：定义数据加载逻辑
class MoviePagingSource(
    private val api: FilmApi
) : PagingSource<Int, Movie>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Movie> {
        return try {
            val page = params.key ?: 1
            val response = api.getMovies(page = page)
            LoadResult.Page(
                data = response.data,
                prevKey = if (page > 1) page - 1 else null,
                nextKey = if (response.data.isNotEmpty()) page + 1 else null
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}

// PagingData：在 ViewModel 中使用
class MovieViewModel : ViewModel() {
    val pagingData: Flow<PagingData<Movie>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            enablePlaceholders = false,
            prefetchDistance = 5
        ),
        pagingSourceFactory = { MoviePagingSource(api) }
    ).flow.cachedIn(viewModelScope)
}

// Compose UI 中使用
@Composable
fun MovieList(viewModel: MovieViewModel) {
    val lazyPagingItems = viewModel.pagingData.collectAsLazyPagingItems()
    LazyColumn {
        items(lazyPagingItems) { movie ->
            MovieCard(movie = movie)
        }
        lazyPagingItems.apply {
            when {
                loadState.refresh is LoadState.Loading -> LoadingItem()
                loadState.append is LoadState.Loading -> LoadingItem()
                loadState.refresh is LoadState.Error -> ErrorItem()
            }
        }
    }
}
```

**RemoteMediator（网络+数据库分页）**
- 适用于"网络数据缓存到本地数据库"的场景
- 结合 Room 使用，实现离线优先
- 自动处理页面失效和刷新

---

### 2.5 性能优化

#### 2.5.1 启动优化

**启动类型**
- 冷启动：进程不存在，从头开始
- 温启动：进程存在，Activity 被销毁
- 热启动：进程存在，Activity 在内存中

**冷启动流程**
```
Zygote fork 进程
  → Application.onCreate()
    → ContentProvider.onCreate()
    → 主线程初始化
  → Activity.onCreate()
  → Activity.onResume()
  → 第一帧绘制
```

**优化策略**
1. **App Startup 初始化任务依赖图**
   ```kotlin
   // 定义初始化任务
   class MyInitializer : Initializer<Unit>() {
       override fun create(context: Context) {
           // 初始化 SDK
       }
       override fun dependencies(): List<Class<out Initializer<*>>> {
           // 声明依赖的其他 Initializer
           return listOf(OtherInitializer::class.java)
       }
   }
   ```
   - 自动拓扑排序，并行执行无依赖的任务
   - 支持延迟初始化（通过 `startup:startup-runtime` 按需加载）

2. **异步初始化**
   - 非必须的初始化放到后台线程
   - 使用 `init {}` 块 + 协程

3. **类预加载**
   - 在 Application 中提前加载关键类
   - 减少首次使用时的类加载耗时

4. **IO 重排**
   - 优化启动路径上的文件访问顺序
   - 减少磁盘寻道时间

#### 2.5.2 内存泄漏与诊断

**常见内存泄漏场景**
1. **Handler 泄漏**
   ```java
   // 错误：内部类持有 Activity 引用
   Handler handler = new Handler() {
       @Override
       public void handleMessage(Message msg) { }
   }
   
   // 正确：静态内部类 + WeakReference
   static class SafeHandler extends Handler {
       private WeakReference<Activity> activityRef;
       SafeHandler(Activity activity) {
           activityRef = new WeakReference<>(activity);
       }
   }
   ```

2. **匿名内部类泄漏**
   - 匿名 Runnable/Thread 持有外部类引用
   - 解决方案：静态内部类 + WeakReference

3. **单例持有 Context**
   - 单例持有 Activity Context 导致 Activity 无法回收
   - 解决方案：使用 Application Context

4. **静态 View 引用**
   - 静态变量持有 View，View 持有 Activity
   - 解决方案：及时置空

**LeakCanary 原理**
1. Activity.onDestroy() 时，ObjectWatcher 记录引用
2. 等待 5 秒后，检查引用是否被回收
3. 如果未被回收，触发 GC 再次检查
4. 仍然存在，dump hprof 文件
5. 分析 hprof，找到 GC Root 到泄漏对象的引用链

**hprof 分析**
- MAT：Dominator Tree 查看大对象，GC Root 分析引用链
- Android Studio Profiler：直接查看对象分配和引用

---

## 三、框架层

### 3.1 Binder 机制

#### 3.1.1 Binder 原理

**为什么需要 Binder？**
- Android 是多进程系统（每个 App 一个进程）
- 进程间通信需要 IPC
- Binder 相比 Linux 传统 IPC（Socket/Pipe/共享内存）的优势：
  - 安全性：Binder 有 UID/PID 验证
  - 性能：一次拷贝（传统 IPC 两次拷贝）
  - 面向对象：调用远程服务像调用本地方法

**mmap 一次拷贝**
```
传统 IPC（两次拷贝）：
  进程 A 数据 → 内核空间 → 进程 B 数据

Binder IPC（一次拷贝）：
  进程 A 数据 → 内核空间（mmap 共享）
                ↓
  进程 B 直接读取（mmap 映射到同一物理内存）
```

**Binder 架构**
```
Client → Binder Driver → ServiceManager → Server
  │          │               │              └── 提供服务
  │          │               └── 服务注册/查询
  │          └── /dev/binder 设备驱动
  └── 调用服务
```

**AIDL 自动生成**
- Stub（服务端）：继承 Binder，实现接口方法
- Proxy（客户端）：通过 Binder 驱动调用服务端方法
- 数据通过 Parcel 序列化/反序列化

#### 3.1.2 Binder 死锁

**场景**
- 主线程调用 Binder 方法，等待服务端返回
- 服务端又回调客户端的 Binder 方法
- 客户端主线程被阻塞，无法处理回调
- 死锁！

**解决方案**
- Binder 调用使用异步（oneway）
- 或者在工作线程中调用 Binder 方法
- 避免主线程 Binder 调用 + 回调的循环

---

### 3.2 Handler 机制

#### 3.2.1 消息循环

**Looper 准备**
```java
// 主线程自动调用 Looper.prepareMainLooper()
// 子线程需要手动调用
class MyThread extends Thread {
    public Handler handler;
    
    @Override
    public void run() {
        Looper.prepare();
        handler = new Handler();
        Looper.loop();  // 进入消息循环
    }
}
```

**消息发送与处理**
```
Handler.sendMessage(msg)
  → MessageQueue.enqueueMessage（按时间排序插入链表）
  → Looper.loop() 循环
    → MessageQueue.next() 取出消息
      → 没有消息：epoll.wait() 阻塞
      → 有消息：返回消息
    → Handler.dispatchMessage(msg)
      → handleMessage(msg)
```

**同步屏障（SyncBarrier）**
- 插入一个屏障消息（target = null）
- 屏障后的同步消息被阻塞
- 只有异步消息可以执行
- 应用：Choreographer 用同步屏障优先处理 Vsync 信号

**IdleHandler**
- 消息队列空闲时执行
- 适合做非紧急的初始化（预加载、懒加载）
- 返回 true 保留，false 移除

---

### 3.3 Activity 启动流程

**完整流程**
```
1. Launcher.startActivity()
2. Instrumentation.execStartActivity()
3. ATMS（ActivityTaskManagerService）
   - 检查 Activity 是否存在
   - 决定启动模式
   - 创建 ActivityRecord/TaskRecord
4. 如果进程不存在：
   - Zygote fork 新进程
   - ActivityThread.main() 启动
   - ActivityThread.attach() 绑定到 ATMS
5. ATMS 通知 ActivityThread 创建 Activity
6. Instrumentation.newActivity() 反射创建
7. Activity.onCreate() 调用
```

**冷启动为什么慢？**
- Zygote fork 进程
- 加载 Application 类
- 执行 ContentProvider.onCreate()
- 加载 Activity 类
- 布局 inflate 和测量

---

### 3.4 Window 与 View 系统

#### 3.4.1 Window 层级

**Window 类型**
- 应用窗口（1-99）：Activity 的 Window
- 子窗口（1000-1999）：Dialog、PopupWindow
- 系统窗口（2000-2999）：Toast、状态栏、输入法

**Window 添加流程**
```
WindowManager.addView(view, params)
  → WindowManagerGlobal.addView()
    → ViewRootImpl.setView()
      → WMS.addWindow()
        → 创建 WindowState
        → 分配 Surface
      → ViewRootImpl.requestLayout()
        → performTraversals()
```

#### 3.4.2 SurfaceFlinger 合成

**合成策略**
- Client 合成：GPU 合成，性能好，支持特效
- Device 合成：HWC 硬件合成，功耗低
- 混合合成：部分图层 GPU，部分 HWC

**Triple Buffer**
- 双缓冲：A 绘制，B 显示，Vsync 到来时交换
- 三缓冲：A 绘制，B 等待，C 显示
- 解决双缓冲下的"掉帧-空闲"问题

---

## 四、架构设计与工程化

### 4.1 大型项目架构演进

#### 4.1.1 单体 → 模块化 → 组件化

**单体架构**
- 所有代码在一个 module
- 优点：简单，适合小团队
- 缺点：编译慢（10min+），耦合高，协作困难

**模块化架构**
```
app（壳工程）
  ├── module_common（基础组件）
  ├── module_network（网络层）
  ├── module_home（首页业务）
  └── module_mine（我的业务）
```
- 按功能拆分 module
- 依赖关系：app → 业务模块 → 基础模块
- 优点：编译快（增量编译），职责清晰

**组件化架构**
```
app（壳工程）
  ├── module_common（基础组件）
  ├── module_network（网络层）
  ├── component_home（首页组件，可独立运行）
  └── component_mine（我的组件，可独立运行）
```
- 业务组件可独立运行（application/library 切换）
- 组件间通过路由通信
- 优点：独立开发/测试，团队并行

#### 4.1.2 组件化通信方案

**路由框架（ARouter）**
```kotlin
// 定义路由
@Route(path = "/home/main")
class HomeActivity : AppCompatActivity()

// 跳转
ARouter.getInstance()
    .build("/home/main")
    .withString("key", "value")
    .navigation()

// 拦截器
@Interceptor(priority = 1)
class LoginInterceptor : IInterceptor {
    override fun process(postcard: Postcard, callback: InterceptorCallback) {
        if (需要登录) {
            // 跳转登录页
        } else {
            callback.onContinue(postcard)
        }
    }
}
```

**服务发现**
```kotlin
// 定义服务接口
interface IUserService {
    fun getUserInfo(): UserInfo
}

// 实现服务
@Route(path = "/service/user")
class UserServiceImpl : IUserService {
    override fun getUserInfo(): UserInfo { ... }
}

// 调用服务
val userService = ARouter.getInstance().navigation(IUserService::class.java)
```

---

### 4.2 Gradle 高级工程化

#### 4.2.1 Version Catalog

**libs.versions.toml**
```toml
[versions]
kotlin = "2.1.0"
agp = "8.7.0"
compose-bom = "2024.12.01"
compose-compiler = "1.5.15"  # Kotlin 2.0+ 编译器插件已内置

[libraries]
kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin" }
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "compose-bom" }

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
```

**build.gradle.kts 引用**
```kotlin
plugins {
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)  // Kotlin 2.0+ 必须
}

android {
    compileSdk = 35
    defaultConfig {
        minSdk = 26
        targetSdk = 35
    }
}

dependencies {
    implementation(libs.kotlin.stdlib)
    implementation(platform(libs.compose.bom))
}
```

#### 4.2.2 Convention Plugin（预编译脚本插件）

**项目结构**
```
build-logic/
  ├── settings.gradle.kts
  └── convention/
      ├── build.gradle.kts
      └── src/main/kotlin/
          └── AndroidApplicationConventionPlugin.kt
```

**build-logic/settings.gradle.kts**
```kotlin
dependencyResolutionManagement {
    repositories.gradlePluginPortal()
}
rootProject.name = "build-logic"
include(":convention")
```

**自定义插件（Kotlin DSL 预编译脚本）**
```kotlin
// build-logic/convention/src/main/kotlin/AndroidApplicationConventionPlugin.kt
class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            plugins.apply("com.android.application")
            plugins.apply("org.jetbrains.kotlin.android")
            plugins.apply("org.jetbrains.kotlin.plugin.compose")
            
            android {
                compileSdk = 35
                defaultConfig {
                    minSdk = 26
                    targetSdk = 35
                }
                buildFeatures {
                    compose = true
                }
            }
        }
    }
}
```

**根项目引用**
```kotlin
// settings.gradle.kts
pluginManagement {
    includeBuild("build-logic")
}
```

---

### 4.3 CI/CD 流水线

**GitHub Actions 示例**
```yaml
name: Android CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
      - name: Run tests
        run: ./gradlew test
      - name: Build
        run: ./gradlew assembleDebug
      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: app-debug.apk
          path: app/build/outputs/apk/debug/
```

---

## 五、测试体系

> 测试不是"写完代码再补"的事后工作，而是架构设计的一部分。Google 官方架构指南将测试列为 Android 开发的三大支柱之一（架构、测试、性能）。

### 5.1 测试金字塔

```
         /\
        /E2E\       极少，验证关键用户路径（如完成购买流程）
       /------\
      /  集成  \     中等，验证多模块协作（如 DB + Network + Repository）
     /----------\
    /   单元测试  \   最多，快速反馈，验证单个类或函数
   /--------------\
```

**各层职责与工具**

| 层级 | 验证内容 | 推荐工具 | 执行速度 |
|------|---------|---------|---------|
| 单元测试 | 单个函数/类的逻辑正确性 | JUnit5 + MockK + Turbine | 毫秒级 |
| 集成测试 | 多组件协作（Room + DAO、Retrofit + 拦截器） | Room 内存数据库、MockWebServer | 秒级 |
| UI 测试 | 用户可见的行为（点击、滑动、导航） | ComposeTestRule / Espresso | 秒-分钟级 |
| E2E 测试 | 完整用户场景 | UIAutomator / Maestro | 分钟级 |

**70/20/10 法则**：70% 单元测试、20% 集成测试、10% E2E。

### 5.2 单元测试

#### JUnit5 + MockK 实战

```kotlin
// 被测类
class UserRepository(
    private val api: UserApi,
    private val db: UserDao
) {
    suspend fun getUser(id: String): User {
        // 先读缓存
        val cached = db.getUser(id)
        if (cached != null) return cached
        // 缓存未命中，请求网络
        val user = api.getUser(id)
        db.insertUser(user)
        return user
    }
}

// 测试类
class UserRepositoryTest {
    @MockK
    private lateinit var api: UserApi

    @MockK
    private lateinit var db: UserDao

    private lateinit var repository: UserRepository

    @BeforeEach
    fun setUp() {
        MockKAnnotations.init(this)
        repository = UserRepository(api, db)
    }

    @Test
    fun `getUser returns cached user when available`() = runTest {
        val cachedUser = User(id = "1", name = "Cached")
        coEvery { db.getUser("1") } returns cachedUser

        val result = repository.getUser("1")

        assertEquals(cachedUser, result)
        coVerify(exactly = 0) { api.getUser(any()) }  // 确认没调网络
    }

    @Test
    fun `getUser fetches from network when cache miss`() = runTest {
        val apiUser = User(id = "1", name = "Fetched")
        coEvery { db.getUser("1") } returns null
        coEvery { api.getUser("1") } returns apiUser
        coEvery { db.insertUser(apiUser) } just runs

        val result = repository.getUser("1")

        assertEquals(apiUser, result)
        coVerify { api.getUser("1") }
        coVerify { db.insertUser(apiUser) }
    }
}
```

**关键要点**
- `runTest` 提供测试用协程作用域，自动跳过延迟，不依赖真实时间
- `@MockK` + `coEvery` 模拟协程函数，`coVerify` 验证调用行为
- 测试方法名应描述行为而非实现（如 `getUser returns cached user when available`）
- 每个测试只验证一个行为路径

#### Flow 测试（Turbine）

```kotlin
@Test
fun `Flow emits data when database changes`() = runTest {
    val userFlow = MutableStateFlow(User("1", "Old"))
    coEvery { db.observeUser("1") } returns userFlow

    repository.observeUser("1").test {
        // 初始值
        assertEquals(User("1", "Old"), awaitItem())
        // 模拟数据变更
        userFlow.value = User("1", "New")
        assertEquals(User("1", "New"), awaitItem())
        // 取消订阅
        cancelAndIgnoreRemainingEvents()
    }
}
```
- Turbine 专为 Flow 测试设计，`test { }` 块内按顺序 `awaitItem()` 消费发射值
- `cancelAndIgnoreRemainingEvents()` 告诉 Turbine 剩余事件不重要，避免测试因未消费事件而失败

#### TestDispatcher 与主线程调度

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule : TestWatcher() {
    val testDispatcher = UnconfinedTestDispatcher()

    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```
- 替换 `Dispatchers.Main` 为 `UnconfinedTestDispatcher`，避免测试中主线程限制
- 使用 `TestWatcher` 自动管理生命周期（`@get:Rule` 在 JUnit4 中）

### 5.3 UI 测试

#### Compose UI Test

```kotlin
class CounterScreenTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun counter_increments_when_button_clicked() {
        composeTestRule.setContent {
            CounterScreen()  // Composable 函数
        }

        // 断言初始状态
        composeTestRule.onNodeWithText("Count: 0").assertExists()

        // 点击按钮
        composeTestRule.onNodeWithText("Increment").performClick()

        // 断言状态变更
        composeTestRule.onNodeWithText("Count: 1").assertExists()
    }

    @Test
    fun list_displays_items_from_viewmodel() {
        val fakeRepository = FakeUserRepository().apply {
            users = listOf(User("1", "Alice"), User("2", "Bob"))
        }
        composeTestRule.setContent {
            UserListScreen(userRepository = fakeRepository)
        }

        composeTestRule.onNodeWithText("Alice").assertIsDisplayed()
        composeTestRule.onNodeWithText("Bob").assertIsDisplayed()
    }
}
```

**查找器语义（Semantics）**
- Compose 通过语义树（Semantics Tree）定位元素，不是 View ID
- `onNodeWithText()` / `onNodeWithTag()` / `onNode(hasContentDescription(...))`
- 自定义 Composable 应添加 `Modifier.semantics { contentDescription = "..." }` 提高可测试性

#### 等待与异步

```kotlin
// 等待条件满足
composeTestRule.waitUntil(timeoutMillis = 5000) {
    composeTestRule.onAllNodesWithText("Loaded").fetchSemanticsNodes().isNotEmpty()
}

// 等待空闲（所有重组和动画完成）
composeTestRule.waitForIdle()
```

### 5.4 截图测试（Snapshot Testing）

截图测试捕获 Composables 的像素级输出，防止意外的 UI 回归。

**Roborazzi（推荐）**

```kotlin
class UserCardScreenshotTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun captureUserCard() {
        composeTestRule.setContent {
            UserCard(user = User("Alice", "alice@example.com"))
        }

        // 截图并对比，输出路径：build/outputs/roborazzi/
        composeTestRule
            .onRoot()
            .captureRoboImage("src/test/screenshots/UserCard.png")
    }
}
```
- 首次运行生成基准截图
- 后续运行自动对比像素差异
- 差异图保存在 `build/outputs/roborazzi/`，可直接查看

### 5.5 假数据策略（Fakes vs Mocks）

| 方式 | 适用场景 | 示例 |
|------|---------|------|
| **Mock（MockK）** | 验证行为交互（是否调用、调用次数） | 验证 Repository 调用了 API |
| **Fake** | 需要真实行为的测试 | FakeUserRepository 用内存 Map 模拟数据库 |
| **Stub** | 只返回预设数据 | 固定返回值的简单场景 |

```kotlin
// Fake：行为接近真实实现，可在多个测试间复用
class FakeUserDao : UserDao {
    private val users = mutableMapOf<String, User>()

    override suspend fun insertUser(user: User) {
        users[user.id] = user
    }

    override suspend fun getUser(id: String): User? = users[id]
}

// Fake 的优势
// 1. 可复用：一套 Fake 用于多个测试类
// 2. 状态隔离：每个测试创建新 Fake 实例，无污染
// 3. 行为真实：插入后再查询能检查到，不需要 coEvery 逐一配置
```

### 5.6 测试最佳实践

- **F.I.R.S.T 原则**：Fast（快）、Independent（独立）、Repeatable（可重复）、Self-validating（自验证）、Timely（及时）
- **不在测试中创建真实网络连接**：用 MockWebServer 或 MockK
- **Room 测试用内存数据库**：`Room.inMemoryDatabaseBuilder()`
- **避免测试实现细节**：测试行为而非内部状态变量名
- **CI 中运行测试**：`./gradlew test` + `./gradlew connectedAndroidTest`

---

## 六、性能优化体系

### 6.1 稳定性治理

#### 6.1.1 Crash 治理

**Java Crash 捕获**
```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
            // 保存崩溃信息到本地
            saveCrashLog(throwable)
            // 重启 App
            restartApp()
        }
    }
}
```

**Native Crash 捕获**
- 使用 Breakpad 或 Matrix 捕获 Native 崩溃
- 原理：注册 signal handler，捕获 SIGSEGV/SIGABRT 等信号
- 解析 minidump 文件获取崩溃堆栈

**崩溃率 SLA 指标（参考值，因业务而异）**
- OOM 率：< 0.1%（千分之一）
- ANR 率：< 0.05%（万分之五）
- Java Crash 率：< 0.1%（千分之一）
- Native Crash 率：< 0.05%（万分之五）

#### 6.1.2 ANR 分析

**ANR 类型**
- InputDispatching：输入事件未在 5 秒内处理完毕
- Broadcast：前台广播 onReceive 10 秒内未返回 (后台广播 60 秒，Android 10+ 有更严限制)
- Service：前台服务生命周期方法 20 秒内未完成 (后台服务 200 秒)
- ContentProvider：Provider 进程首次启动时，其 onCreate 未在 10 秒内返回
- JobService：前台 JobService 的 onStartJob/onStopJob 约 10 秒内未完成 (后台约几分钟)
- 前台 Service 启动：调用 startForegroundService() 后，未在 10 秒内调用 startForeground()
- 异步广播：BroadcastReceiver 调用 goAsync() 后未及时调用 PendingResult.finish()
- 系统 WatchDog：system_server 的关键线程在 1 分钟内未响应 (非 App 直接导致)

**ANR 触发机制**
- 系统服务超时监控：InputManagerService 监控输入超时，AMS 监控广播、服务、ContentProvider 超时
- 检测到超时后，系统向应用进程发送 SIGQUIT (Signal 3) 收集所有线程堆栈，写入 traces.txt
- 进程若被终止，可能收到 SIGABRT (Signal 6)，但这是 ANR 发生后的结果，非触发原因

**traces.txt 分析**
- 定位主线程堆栈，确认当前状态 (RUNNABLE/BLOCKED/WAITING/NATIVE)
- 若主线程为 BLOCKED 或 WAITING，需找到导致其阻塞的锁持有线程或条件变量
- 关注 Binder 线程池状态，排查跨进程死锁
- 常见原因：主线程 IO、锁竞争、Binder 调用延迟、死锁、主线程等待子线程未通知

---

### 6.2 内存管理原理与高级优化

#### 6.2.1 GC 原理

**ART GC 算法**
- Concurrent Copying：并发复制，暂停时间短
- Generational：分代回收，新生代和老年代分开处理
- 暂停时间目标：< 2ms

**GC 暂停时间优化**
- 减少 GC 触发频率：避免频繁创建大量临时对象
- 优化对象分配：使用对象池、避免在循环中创建对象
- 监控 GC 日志：通过 logcat 查看 GC 暂停时间

**大对象分配与 TLAB**
- TLAB（Thread Local Allocation Buffer）：每个线程在堆中分配一块缓冲区
- 小对象在 TLAB 中分配，无需同步
- 大对象直接分配在堆中，需要同步

#### 6.2.2 Native Heap 分析

**Native 内存泄漏检测**
- malloc debug：通过系统属性开启 Native 内存调试
- malloc hook：Hook malloc/free 调用，记录分配堆栈
- AddressSanitizer（ASan）：检测内存越界、使用后释放等问题

**常见 Native 泄漏场景**
- JNI 全局引用未释放
- Native 线程创建未销毁
- 第三方 Native 库内存泄漏

#### 6.2.3 内存抖动

**什么是内存抖动？**
- 短时间内频繁创建和销毁对象
- 导致频繁 GC，造成卡顿

**常见原因**
- 循环中创建对象
- onDraw 中创建对象（每帧都创建）
- 频繁的字符串拼接

**检测方法**
- Android Profiler Memory 查看分配频率
- Allocation Tracking 查看分配堆栈

---

### 6.3 卡顿优化体系

#### 6.3.1 掉帧监控

**FrameCallback 实现**
```kotlin
class FpsMonitor {
    private var lastFrameTime = 0L
    private var frameCount = 0
    private var lastFpsTime = 0L
    
    fun start() {
        Choreographer.getInstance().postFrameCallback { frameTimeNanos ->
            if (lastFrameTime == 0L) {
                lastFrameTime = frameTimeNanos
                lastFpsTime = frameTimeNanos
            }
            
            val diff = (frameTimeNanos - lastFrameTime) / 1_000_000
            if (diff > 16.6f) {
                // 掉帧，diff 为实际耗时
                val droppedFrames = (diff / 16.6f).toInt()
                reportJank(droppedFrames)
            }
            
            frameCount++
            if (frameTimeNanos - lastFpsTime >= 1_000_000_000) {
                val fps = frameCount
                reportFps(fps)
                frameCount = 0
                lastFpsTime = frameTimeNanos
            }
            
            lastFrameTime = frameTimeNanos
            Choreographer.getInstance().postFrameCallback(this)
        }
    }
}
```

#### 6.3.2 布局优化

**布局层级优化**
- 使用 ConstraintLayout 减少嵌套层级
- 使用 merge 标签减少冗余布局
- 使用 ViewStub 延迟加载不常用的布局

**异步布局**
- AsyncLayoutInflater：在后台线程 inflate 布局
- Compose：声明式 UI，天然支持异步布局
- Litho：Facebook 的异步布局框架

**过度绘制优化**
- 移除不必要的背景（Activity 默认背景、重复背景）
- 使用 clipRect 限制绘制区域
- 使用 GPU Overdraw 工具检测

---

### 6.4 网络优化

#### 6.4.1 HTTPDNS

**为什么需要 HTTPDNS？**
- 传统 DNS 解析存在劫持风险（运营商劫持、DNS 污染）
- DNS 解析速度慢（可能经过多级 DNS 服务器，且 UDP 不可靠）
- 不支持精准的流量调度（无法根据用户来源返回最近的 IP）
- 域名解析结果不缓存或缓存不可控

**实现方案**
```kotlin
// OkHttp 集成 HTTPDNS
class HttpDnsInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val originalUrl = request.url
        val host = originalUrl.host
        
        // 从 HTTPDNS 获取 IP
        val ip = HttpDnsService.getIpByHost(host)
        
        if (ip != null) {
            // 替换 Host 为 IP
            val newUrl = originalUrl.newBuilder()
                .host(ip)
                .build()
            val newRequest = request.newBuilder()
                .url(newUrl)
                .header("Host", host)
                .build()
            return chain.proceed(newRequest)
        }
        
        return chain.proceed(request)
    }
}
```

#### 6.4.2 弱网优化

**超时策略**
- 连接超时：5-10 秒
- 读取超时：10-30 秒（根据业务调整）
- 写入超时：10-30 秒

**重试策略**
- 指数退避：1s、2s、4s、8s、16s
- 最大重试次数：3-5 次
- 不同错误码不同策略（4xx 不重试，5xx 重试）

**请求合并**
- 相同接口的请求合并为一个
- 减少网络连接次数
- 使用 Flow 的 debounce 操作符合并高频请求

---

### 6.5 包体积优化

#### 6.5.1 代码混淆

**ProGuard 规则**
```proguard
# 保留实体类
-keep class com.example.model.** { *; }

# 保留反射调用的类
-keep class com.example.reflection.** { *; }

# 保留 JNI 方法
-keepclasseswithmembernames class * {
    native <methods>;
}

# 移除日志
-assumenosideeffects class android.util.Log {
    public static boolean isLoggable(java.lang.String, int);
    public static int v(...);
    public static int d(...);
    public static int i(...);
}
```

**R8 全量模式**
- 开启 R8 全量模式：`android.enableR8.fullMode=true`
- 更激进的优化：类合并、方法内联、未使用代码移除

#### 6.5.2 资源优化

**图片压缩**
- WebP：有损/无损压缩，比 PNG 小 25-35%
- AVIF：下一代图片格式，比 WebP 小 20%
- 矢量图：适合图标，无限缩放

**无用资源移除**
- Lint 检查：`./gradlew lint` 检测未使用的资源
- 资源 shrink：`shrinkResources true` 自动移除

**动态下发**
- On Demand Delivery：按需下载模块
- 资源分包：不常用的资源延迟下载

---

### 6.6 Baseline Profiles 与启动加速

Baseline Profiles 是 AGP 7.0+ 引入的 AOT 编译优化，可显著提升应用首次启动和页面打开速度。

**原理**

```
Cloud Profiles → APK 内置 baseline-prof.txt
        ↓
Dex2oat AOT 编译 → 关键代码路径提前编译为机器码
        ↓
启动时跳过 JIT 解释 → 启动速度提升 20-40%
```

**生成步骤**

```kotlin
// 1. 创建基准配置文件生成器
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generate() = rule.collect(packageName = "com.example.app") {
        pressHome()
        // 遍历关键用户路径（冷启动 → 首页 → 详情页）
        startActivityAndWait()
        device.findObject(By.text("详情")).click()
        device.waitForIdle()
    }
}

// 2. 运行：./gradlew :app:generateBaselineProfile
// 3. 输出：app/src/main/baseline-prof.txt 纳入版本控制
```

**Macrobenchmark 测量**

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule
    val rule = MacrobenchmarkRule()

    @Test
    fun measureColdStart() = rule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        setupBlock = {
            pressHome()
            // 每个迭代前杀死进程模拟冷启动
        }
    ) {
        startActivityAndWait()
    }
}
```

**关键指标**

| 指标 | 说明 | 目标值 |
|------|------|--------|
| `timeToInitialDisplay` | 首帧显示时间 | < 500ms |
| `timeToFullDisplay` | 完全可交互时间 | < 1.5s |
| `frameTimingMs` | 首帧渲染耗时分布 | P50 < 16ms |

### 6.7 App Startup 初始化任务管理

App Startup 替代传统 Application.onCreate() 中散乱的初始化代码，支持任务依赖图和自动拓扑排序。

```kotlin
// 定义初始化任务
class AnalyticsInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        // 初始化 Firebase Analytics
        Firebase.analytics.setAnalyticsCollectionEnabled(true)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        // 依赖其他初始化任务先完成
        return listOf()
    }
}

class CrashInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        // 初始化崩溃收集
        Thread.setDefaultUncaughtExceptionHandler { _, e -> saveCrash(e) }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf()  // 无依赖，可与 Analytics 并行
    }
}
```

**注册（AndroidManifest.xml 或 Provider 自动发现）**

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.AnalyticsInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="com.example.CrashInitializer"
        android:value="androidx.startup" />
</provider>
```

**延迟初始化**

```kotlin
// 非启动必备的初始化，标记为手动
class CacheWarmInitializer : Initializer<Unit> {
    override fun create(context: Context) { ... }
    override fun dependencies() = listOf()

    // 不自动初始化
}

// 按需手动初始化
AppInitializer.getInstance(context)
    .initializeComponent(CacheWarmInitializer::class.java)
```

**对比传统方式**

| 维度 | 传统 Application.onCreate() | App Startup |
|------|---------------------------|-------------|
| 执行顺序 | 手动控制（容易遗漏依赖） | 自动拓扑排序 |
| 并行初始化 | 需手动线程管理 | 自动识别无依赖任务并行执行 |
| 延迟初始化 | 需自己判断 | 原生支持按需加载 |
| 可测试性 | 难以单独测试 | 每个 Initializer 可独立测试 |

---

## 七、跨平台与新技术

### 7.1 Compose Multiplatform

#### 7.1.1 声明式 UI 原理

**重组（Recomposition）**
- 当状态变化时，Compose 重新执行可组合函数
- 智能跳过（Skipping）：只有参数变化的组件才会重组
- 跳过条件：参数 equals 比较（稳定类型）或 @Stable 注解
- 不稳定类型（如 data class 中的 List/Interface 字段）会导致过度重组

**状态管理**
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

**状态提升（State Hoisting）**
```kotlin
@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    Counter(count = count, onIncrement = { count++ })
}

@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Column {
        Text("Count: $count")
        Button(onClick = onIncrement) {
            Text("Increment")
        }
    }
}
```

**Compose 稳定性优化**
- 使用 `@Stable` 注解标记稳定类型
- 使用 `kotlinx-collections-immutable` 确保集合稳定性
- 避免在 Composable 参数中使用接口类型（如 List 用 immutableList）
- 使用 `derivedStateOf` 减少不必要的重组

**副作用**
- LaunchedEffect：进入组合时启动协程，离开时取消
- DisposableEffect：需要清理的资源（如注册/反注册监听器）
- SideEffect：每次重组都执行（用于与非 Compose 代码共享状态）
- rememberCoroutineScope：获取组合感知的协程作用域
- snapshotFlow：将 Compose 状态转换为 Flow

#### 7.1.2 跨平台渲染

**Android 平台**
- Compose 基于 View 系统
- 通过 AndroidComposeView 桥接
- 与 View 系统互操作：AndroidView/ComposeView

**iOS 平台**
- 基于 UIKit，使用 SKIA 渲染
- 通过 ComposeUIViewController 嵌入
- 性能接近原生

**Desktop 平台**
- 基于 Skia 渲染引擎，直接与操作系统窗口交互
- 不再基于 Swing/JavaFX（旧版 Compose for Desktop 曾基于 Swing，现已独立），直接使用 Skia 渲染引擎与操作系统窗口交互

#### 7.1.3 Compose 动画系统

Compose 动画分为两大类：**值动画**（animate*AsState / Animatable）和**过渡动画**（Transition / AnimatedVisibility）。

**animate*AsState（最简单，推荐首选）**

```kotlin
@Composable
fun AnimatedBox(isBig: Boolean) {
    // 值变化时自动插值过渡
    val size by animateDpAsState(
        targetValue = if (isBig) 100.dp else 50.dp,
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy)
    )
    val color by animateColorAsState(
        targetValue = if (isBig) Color.Red else Color.Blue,
        animationSpec = tween(durationMillis = 300)
    )
    Box(Modifier.size(size).background(color))
}
```

**动画规格对比**

| 规格 | 特点 | 适用场景 |
|------|------|---------|
| `spring()` | 物理弹簧，自然有弹性 | 尺寸变化、拖拽回弹 |
| `tween()` | 线性插值，可配缓动曲线 | 颜色过渡、透明度 |
| `keyframes()` | 关键帧控制 | 路径动画、多阶段动画 |
| `snap()` | 瞬间切换 | 模式切换（暗黑模式） |
| `repeatable()` | 循环重复 | 加载动画、脉冲效果 |

**Animatable（精细控制）**

```kotlin
@Composable
fun GestureAnimation() {
    val offsetX = remember { Animatable(0f) }

    Box(
        Modifier
            .offset { IntOffset(offsetX.value.roundToInt(), 0) }
            .draggable(
                orientation = Orientation.Horizontal,
                state = rememberDraggableState { delta ->
                    // 拖拽时跟随手指
                    coroutineScope.launch { offsetX.snapTo(offsetX.value + delta) }
                },
                onDragStopped = {
                    // 松手后回弹到 0
                    coroutineScope.launch {
                        offsetX.animateTo(0f, spring())
                    }
                }
            )
    )
}
```

**AnimatedVisibility（显示/隐藏动画）**

```kotlin
AnimatedVisibility(
    visible = isVisible,
    enter = fadeIn() + slideInVertically(),
    exit = fadeOut() + slideOutVertically()
) {
    Text("Hello Compose")
}
```

**Transition（多属性同步动画）**

```kotlin
enum class BoxState { Collapsed, Expanded }

@Composable
fun TransitionBox(state: BoxState) {
    val transition = updateTransition(targetState = state, label = "box")

    val size by transition.animateDp(label = "size") { s ->
        when (s) { BoxState.Collapsed -> 50.dp; BoxState.Expanded -> 100.dp }
    }
    val color by transition.animateColor(label = "color") { s ->
        when (s) { BoxState.Collapsed -> Color.Gray; BoxState.Expanded -> Color.Green }
    }

    Box(Modifier.size(size).background(color))
}
```

#### 7.1.4 Compose Navigation

Compose Navigation 2.8+ 支持类型安全的导航，无需字符串路由。

```kotlin
// 定义路由（类型安全）
@Serializable data class Detail(val id: String, val name: String)
@Serializable object Home
@Serializable object Settings

@Composable
fun App() {
    val navController = rememberNavController()
    NavHost(navController = navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(onNavigateToDetail = { id, name ->
                navController.navigate(Detail(id, name))
            })
        }
        composable<Detail> { backStackEntry ->
            val detail = backStackEntry.toRoute<Detail>()  // 自动反序列化
            DetailScreen(id = detail.id, name = detail.name)
        }
        composable<Settings> { SettingsScreen() }
    }
}
```

**Deep Link**

```kotlin
composable<Detail>(
    deepLinks = listOf(navDeepLink<Detail>(basePath = "myapp://detail"))
) { ... }
```

**多返回栈（Bottom Navigation 必备）**

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    Scaffold(
        bottomBar = {
            NavigationBar {
                // 使用 saveState + restoreState 保留各 Tab 的返回栈
                listOf(Tab.Home, Tab.Profile, Tab.Settings).forEach { tab ->
                    NavigationBarItem(
                        selected = currentTab == tab,
                        onClick = {
                            navController.navigate(tab.route) {
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true   // 离开时保存状态
                                }
                                launchSingleTop = true
                                restoreState = true    // 回来时恢复状态
                            }
                        }
                    )
                }
            }
        }
    ) { ... }
}
```

#### 7.1.5 Compose 性能诊断

**重组计数（Layout Inspector）**

Android Studio → Layout Inspector → 勾选 "Show Recomposition Counts"，可看到哪些组件被频繁重组，用颜色标注（红=高频）。

**常见性能陷阱与修复**

| 问题 | 表现 | 修复方案 |
|------|------|---------|
| 不稳定参数 | 参数未变但组件重组 | 用 `@Stable`/`@Immutable` 注解，或用 `kotlinx-collections-immutable` |
| Lambda 频繁重建 | `remember` 的 lambda 每次重组都是新对象 | 用 `remember { }` 包裹 lambda，或用函数引用 `::method` |
| 大列表无 key | 列表项全部重建 | 给 `LazyColumn items` 加 `key` 参数 |
| 过度使用 `derivedStateOf` | 过度重组与性能损耗 | 只在需要"跳过中间状态"时使用，普通计算直接用 `remember` |
| Modifier 顺序错误 | 点击/测量性能下降 | `clickable` 放前面，`size`/`padding` 放后面 |
| 在 Composable 中做耗时计算 | 每帧都重新计算 | 用 `remember` 记忆化，或用 `LaunchedEffect` + 后台线程 |

**derivedStateOf 的正确用法**

```kotlin
// 场景：列表滚动时，只在"超过阈值"的瞬间触发重组，而非每次滚动都重组
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
// 只在 showButton 的值（true/false）改变时才重组
// 而 listState.firstVisibleItemIndex 每次变化不会直接触发重组
```

**Phases（理解跳过机制的前提）**

```
Composition  →  Layout  →  Drawing
（执行 Composable） （测量+摆放） （绘制）
       ↓                ↓           ↓
    可跳过条件：      不影响跳过    不影响跳过
    - 参数 equals 为 true
    - 参数为稳定类型
```

- 只有 **Composition 阶段**可以跳过（Skip）
- 状态读取在 Composition 阶段进行
- 把状态读取推迟到 Layout/Drawing 阶段（使用 `Modifier.layout` 中的 lambda）可减少重组范围

---

### 7.2 Kotlin Multiplatform

#### 7.2.1 expect/actual 机制

```kotlin
// commonMain
expect fun getPlatformName(): String

// androidMain
actual fun getPlatformName(): String = "Android ${Build.VERSION.SDK_INT}"

// iosMain
actual fun getPlatformName(): String = "iOS ${UIDevice.currentDevice.systemVersion}"
```

> **Kotlin 2.1+ 新变化**：KMP 引入了新的 `@Expect` / `@Actual` 注解作为实验性功能，未来将替代旧的 `expect`/`actual` 关键字声明方式，提供更好的 IDE 支持和类型检查。

#### 7.2.2 共享逻辑层

**网络层共享**
- Ktor：跨平台 HTTP 客户端
- 序列化：kotlinx.serialization
- KMP-Native-coroutines：桥接 Kotlin 协程与 Swift async/await

**数据层共享**
- Room：跨平台数据库（KMP 支持，Kotlin 2.0+ 稳定）
- SQLDelight：SQL 驱动的跨平台数据库

**业务逻辑共享**
- ViewModel 共享（`lifecycle-viewmodel-compose` 支持 KMP）
- Repository 共享
- UseCase 共享

**KMP 项目结构最佳实践**
```
com.example.app/
  ├── commonMain/        # 共享代码
  │   ├── domain/        # 业务逻辑（纯 Kotlin，无平台依赖）
  │   ├── data/          # 数据层（依赖 Ktor/Room）
  │   └── presentation/  # UI 状态管理（共享 ViewModel）
  ├── androidMain/       # Android 平台实现
  └── iosMain/           # iOS 平台实现
```
- 核心原则：domain 层保持纯 Kotlin，不依赖任何平台框架
- 使用 `@Immutable` / `@Stable` 确保 Compose 跨平台 UI 性能

---

### 7.3 AI 与端侧模型

#### 7.3.1 Google ML Kit

**功能**
- 文本识别：OCR 识别
- 图像标签：自动标注图片内容
- 对象检测：检测和跟踪物体
- 条形码扫描：扫码
- 数字墨水识别：手写识别
- 自拍分割：人像背景分离

**使用方式**
```kotlin
val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)

val image = InputImage.fromBitmap(bitmap, 0)
recognizer.process(image)
    .addOnSuccessListener { result ->
        result.textBlocks.forEach { block ->
            Log.d("OCR", block.text)
        }
    }
```

#### 7.3.2 Google AI Edge (替代 TensorFlow Lite)

> Google 已推出 **AI Edge** 作为 TFLite 的下一代演进方案，提供更好的模型管理和硬件加速支持。

**MediaPipe Tasks（推荐）**
```kotlin
// 使用 MediaPipe Tasks 进行图像分类
val options = ImageClassifierOptions.builder()
    .setBaseOptions(BaseOptions.builder()
        .setModelAssetPath("model.tflite")
        .setDelegate(Delegate.GPU)
        .build())
    .setMaxResults(3)
    .build()
val classifier = ImageClassifier.createFromOptions(context, options)
```

**TensorFlow Lite（仍在维护）**
```kotlin
// 加载模型
val interpreter = Interpreter(loadModelFile(context, "model.tflite"))

// 输入数据
val input = Array(1) { FloatArray(224 * 224 * 3) }

// 推理
val output = Array(1) { FloatArray(1000) }
interpreter.run(input, output)

// 解析结果
val maxIndex = output[0].indices.maxByOrNull { output[0][it] }
```

**优化技术**
- 量化：FP32 → INT8，体积减少 75%，速度提升 2-4 倍
- GPU 委托：使用 GPU 加速推理
- NNAPI：使用 NPU/DSP 加速
- XNNPACK：CPU 推理加速库

#### 7.3.3 大语言模型 (LLM) 端侧部署

**Gemini Nano（Android 内置）**
- Android 14+ 内置 Gemini Nano，通过 AI Core 服务提供端侧大模型能力
- 无需下载模型，系统级集成
- 支持文本摘要、智能回复、翻译等

```kotlin
// 使用 Gemini Nano（通过 AICore）
val aiCore = AICore.create(context)
val prompt = "Summarize this text: ..."
aiCore.generateContent(prompt) { response ->
    // 处理生成结果
}
```

**MediaPipe LLM Inference**
- 支持 Gemma、Phi-2 等小型 LLM 端侧部署
- 量化模型（4-bit）可在手机上流畅运行
- 适用于离线 AI 助手、文档摘要等场景

---

### 7.4 现代 Android 特性（Android 13+）

#### Per-app Language（应用内语言切换）

Android 13+ 支持用户为每个应用单独设置语言，无需修改系统语言。

```kotlin
// 获取 LocaleManager
val localeManager = context.getSystemService(LocaleManager::class.java)

// 设置应用语言
localeManager.applicationLocales = LocaleList.forLanguageTags("zh-CN")

// 读取当前语言
val currentLocales = localeManager.applicationLocales  // [zh-CN]

// 监听语言变化
val receiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 系统语言/应用语言变化时触发，需重建 Activity
        recreate()
    }
}
context.registerReceiver(receiver, IntentFilter(Intent.ACTION_LOCALE_CHANGED))
```

- 应用重启自动恢复上次设置的语言
- 支持多语言列表（fallback 机制）

#### Photo Picker（照片选择器）

Android 13+ 提供系统级 Photo Picker，替代 `READ_EXTERNAL_STORAGE` 权限。

```kotlin
// 单选照片（无需任何权限声明）
val pickMedia = registerForActivityResult(PickVisualMedia()) { uri ->
    if (uri != null) {
        imageView.setImageURI(uri)
    }
}
pickMedia.launch(PickVisualMediaRequest(PickVisualMedia.ImageOnly))

// 多选照片（最多 5 张）
val pickMultipleMedia = registerForActivityResult(
    PickMultipleVisualMedia(5)
) { uris -> /* 处理多张照片 */ }
```

**对比传统方式**

| 维度 | 传统 `READ_EXTERNAL_STORAGE` | Photo Picker |
|------|----------------------------|-------------|
| 权限 | 需要运行时权限 | 无需任何权限 |
| UI | 需自己实现 | 系统级 UI，体验一致 |
| 访问范围 | 整个外部存储 | 用户选择的媒体 |
| 文件类型 | 任意文件 | 照片/视频 |
| 版本 | 所有版本 | Android 13+（Google Play Services 可回退到 11+） |

#### 每应用语言 + 照片选择器回退兼容

```kotlin
// Google Play Services 为 Photo Picker 提供回退支持
dependencies {
    implementation("androidx.activity:activity:1.9+")
    // 自动通过 Play Services 在 Android 11-12 上提供 Photo Picker
}
```

---

## 八、安全与合规

### 8.1 数据安全

#### 8.1.1 加密体系

**对称加密 AES-GCM**
```kotlin
fun encrypt(data: ByteArray, key: SecretKey): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.ENCRYPT_MODE, key)
    val iv = cipher.iv
    val encrypted = cipher.doFinal(data)
    return iv + encrypted  // IV + 密文
}

fun decrypt(data: ByteArray, key: SecretKey): ByteArray {
    val iv = data.copyOfRange(0, 12)
    val encrypted = data.copyOfRange(12, data.size)
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(128, iv))
    return cipher.doFinal(encrypted)
}
```

**国密算法**
- SM2：非对称加密，替代 RSA
- SM3：哈希算法，替代 SHA-256
- SM4：对称加密，替代 AES

#### 8.1.2 安全存储

**EncryptedSharedPreferences**
```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val sharedPreferences = EncryptedSharedPreferences.create(
    context,
    "secure_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
```

**SQLCipher 数据库加密**
- 使用 256 位 AES 加密整个数据库文件
- 性能损耗约 5-15%

---

### 8.2 网络安全

#### 8.2.1 SSL Pinning

```kotlin
// OkHttp CertificatePinner
val certificatePinner = CertificatePinner.Builder()
    .add("example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .build()

val client = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build()
```

**为什么需要 SSL Pinning？**
- 防止中间人攻击
- 即使 CA 被攻破，也能保证连接安全
- 企业内网环境自定义证书

#### 8.2.2 隐私合规

**GDPR 要求**
- 数据最小化：只收集必要的数据
- 用户同意：明确告知并获取同意
- 数据删除：用户有权删除个人数据

**Android 隐私最佳实践**
- 使用 OAID/AAID 替代 IMEI
- 权限最小化：只在需要时申请权限
- 隐私政策：清晰说明数据收集和使用

---

## 九、系统底层与内核

### 9.1 Linux 内核基础

#### 9.1.1 进程调度

**CFS 完全公平调度器**
- 每个进程分配虚拟运行时间（vruntime）
- 选择 vruntime 最小的进程运行
- nice 值影响权重：nice 越低，权重越高

**cgroups 资源限制**
- CPU 限制：cpu.shares
- 内存限制：memory.limit_in_bytes
- IO 限制：blkio.throttle

#### 9.1.2 内存管理

**LMK（Low Memory Killer）**
- 根据 oom_score_adj 评分选择杀死的进程
- oom_score_adj 范围：-1000 到 1000
- 前台进程：0，后台进程：900-1000，空进程：1000
- Android 10+ 使用 LMKD 替代旧版 LMK，通过内核 PSI 机制更精准地预测内存压力

**内存回收**
- kswapd：后台内存回收线程
- 直接回收：内存不足时同步回收
- 回收策略：LRU 页面置换

---

### 9.2 图形渲染管线

#### 9.2.1 SurfaceFlinger 与硬件合成器 (HWC)

> 关于 SurfaceFlinger 的基础合成策略，请参考 [3.4.2 SurfaceFlinger 合成](#342-surfaceflinger-合成) 章节。

**HWC 版本演进**
- HWC 1.0：基本合成
- HWC 1.4：支持图层压缩
- HWC 2.0：虚拟显示、客户端合成

#### 9.2.2 Vsync 同步机制

> 关于 Triple Buffer 的原理，请参考 [3.4.2 SurfaceFlinger 合成](#342-surfaceflinger-合成) 章节。

**Vsync 信号**
- 硬件 Vsync：显示硬件发出的同步信号
- 软件 Vsync：SurfaceFlinger 模拟的 Vsync
- 频率：60Hz/90Hz/120Hz

---

## 十、架构师软技能与工程管理

### 10.1 技术选型方法论

#### 10.1.1 选型评估矩阵

**评估维度**
- 功能完整性：是否满足业务需求
- 性能：基准测试数据
- 稳定性：线上运行时间、Bug 数量
- 社区活跃度：Star 数、贡献者、更新频率
- 学习成本：上手时间、文档质量

**POC 流程**
1. 确定评估维度
2. 搭建 Demo 验证核心功能
3. 性能基准测试
4. 稳定性测试
5. 团队评估

#### 10.1.2 技术决策记录（ADR）

```markdown
# ADR-001：选择 MVI 架构

## 背景
项目需要统一的状态管理方案

## 方案对比
- MVVM：学习成本低，但状态分散
- MVI：状态集中，可回溯，但学习成本高

## 决策
选择 MVI 架构

## 原因
1. 项目复杂度高，需要统一状态管理
2. 团队 Kotlin 经验丰富
3. 需要状态回溯调试

## 后果
- 正面：状态管理统一，调试方便
- 负面：学习成本增加，状态类膨胀
```

### 10.2 架构演进策略

#### 10.2.1 渐进式重构

**防腐层（Anti-Corruption Layer）**
- 在新旧系统之间建立适配层
- 新系统不直接依赖旧系统
- 逐步替换旧系统实现

**绞杀者模式（Strangler Fig）**
- 逐步用新功能替代旧功能
- 旧功能逐渐被"绞杀"
- 最终完全替换

**特性开关（Feature Toggle）**
```kotlin
object FeatureToggle {
    private val features = mutableMapOf<String, Boolean>()
    
    fun isEnabled(feature: String): Boolean {
        return features[feature] ?: false
    }
    
    fun enable(feature: String) {
        features[feature] = true
    }
}

// 使用
if (FeatureToggle.isEnabled("new_home_page")) {
    showNewHomePage()
} else {
    showOldHomePage()
}
```

### 10.3 团队效能

#### 10.3.1 研发流程优化

**敏捷开发实践**
- Scrum：2 周一迭代
- 每日站会：15 分钟同步进度
- 迭代评审：展示成果
- 回顾会议：持续改进

**技术方案评审**
- 重大功能必须出技术方案
- 方案包含：架构图、数据流、接口定义
- 团队评审通过后才能开发

#### 10.3.2 Code Review 清单

**必查项**
- 是否有内存泄漏风险
- 是否有线程安全问题
- 是否有性能问题
- 是否符合架构规范
- 是否有足够的测试覆盖

**自动化检查**
- Detekt：Kotlin 代码规范检查
- Ktlint：Kotlin 代码格式检查
- SpotBugs：Java 字节码分析

### 10.4 技术债务管理

#### 10.4.1 债务量化

**SonarQube 指标**
- 代码复杂度：圈复杂度
- 代码重复率：重复代码比例
- 测试覆盖率：单元测试覆盖
- 技术债务：修复所有问题所需时间

#### 10.4.2 偿还优先级

**评估维度**
- 影响范围：影响多少用户/功能
- 风险等级：是否会导致线上故障
- 业务价值：修复后带来多少收益

**决策矩阵**
| 影响范围 | 风险等级 | 优先级 |
|---------|---------|-------|
| 大 | 高 | 立即修复 |
| 大 | 低 | 下个迭代 |
| 小 | 高 | 下个迭代 |
| 小 | 低 | 技术债务清单 |

---

## 附录：学习路径与级别对照

### P5 初中级工程师
- 核心能力：独立完成模块开发
- 知识覆盖：第一章 基础篇
- 学习重点：Java/Kotlin 基础、四大组件、UI 开发、数据持久化、协程基础

### P6 高级工程师
- 核心能力：独立负责功能/模块
- 知识覆盖：第一章 + 第二章 + 第五章（测试基础）
- 学习重点：自定义 View、网络编程、Jetpack（Lifecycle/ViewModel/DataStore/Room）、架构模式（MVVM/MVI）、性能优化、单元测试编写

### P7 资深工程师
- 核心能力：技术攻坚/性能优化/架构设计
- 知识覆盖：第一章 + 第二章 + 第三章 + 第五章（测试体系）
- 学习重点：Binder、Handler、AMS/WMS、编译构建、Gradle 工程化、稳定性治理、UI 测试与截图测试

### P8 架构师
- 核心能力：技术规划/架构演进/团队赋能
- 知识覆盖：全部章节
- 学习重点：架构设计、性能体系、跨平台（KMP/Compose Multiplatform）、安全合规、AI 端侧部署、测试策略、软技能

## 附录 B：推荐学习资源

### 官方文档
- [Android Developers 官方文档](https://developer.android.com/docs) - 最权威的 Android 开发指南
- [Android 架构指南](https://developer.android.com/topic/architecture) - 官方推荐架构最佳实践
- [Kotlin 官方文档](https://kotlinlang.org/docs/home.html) - Kotlin 语言参考
- [Compose 官方文档](https://developer.android.com/develop/ui/compose) - Jetpack Compose 指南

### 必读书籍
- 《Android 开发艺术探索》 - 任玉刚，深入理解 Android 框架层
- 《Android 进阶之光》 - 刘望舒，Android 进阶知识体系
- 《Kotlin 实战》 - Dmitry Jemerov，Kotlin 语言权威指南
- 《Clean Architecture》 - Robert C. Martin，架构设计经典

### 工具与框架
- [LeakCanary](https://square.github.io/leakcanary/) - 内存泄漏检测
- [Ktor](https://ktor.io/) - 跨平台 HTTP 客户端
- [Koin](https://insert-koin.io/) - 轻量级依赖注入框架
- [Detekt](https://detekt.dev/) - Kotlin 静态代码分析

---

> 最后建议：这份百科全书不是用来"背"的，而是用来"查"的。遇到问题时，找到对应的章节，理解原理和最佳实践，然后应用到实际项目中。真正的架构师能力来自于**解决问题的经验积累**，而不是知识的堆砌。
>
> **保持学习**：Android 生态每年都在快速演进，建议每季度关注 Google I/O 大会的新特性，及时更新知识体系。
