# **Android 开发百科全书**

本指南按照常见的 **Android Project 架构** 组织知识点，涵盖 Android 面试所需的几乎所有主题。

## **目录概览**
1. [Android 基础](#android-基础)
2. [UI 体系](#ui-体系)
3. [数据存储](#数据存储)
4. [Framework 体系](#framework-体系)
5. [Jetpack 组件](#jetpack-组件)
6. [高级优化](#高级优化)
7. [CI/CD 与测试](#cicd-与测试)
8. [Android 适配](#android-适配)

---

## **1. Android 基础**

### **1.1 四大组件（Four Components）**

#### **Activity**
1. **生命周期**
   - 典型回调顺序：`onCreate()` → `onStart()` → `onResume()` → `onPause()` → `onStop()` → `onDestroy()`
   - 特殊回调：`onRestart()`（从停止状态重新启动）
   - **状态保存与恢复**：
     - `onSaveInstanceState()`：在 Activity 被销毁前保存瞬态数据到 `Bundle`。
     - `onRestoreInstanceState()`：在 `onCreate()` 或 `onStart()` 中恢复状态。
   - **重要场景**：屏幕旋转、进程被系统杀死等。

2. **launchMode**（任务栈管理）
   - **Standard**：每次启动都会创建一个新的 Activity 实例。
   - **SingleTop**：如果栈顶已存在此 Activity，则复用，而不再创建新的实例。
   - **SingleTask**：会在已有的任务栈中寻找 Activity 实例并置顶（清除其上的 Activity）。
   - **SingleInstance**：独占一个新的任务栈。
   - 应用场景：浏览器、通知点击跳转等。

3. **Intent**
   - **显式 Intent**：通过指定 `ComponentName`（包名/类名）启动目标 Activity。
   - **隐式 Intent**：通过 `Action`、`Category`、`Data`、`Type` 等匹配系统或第三方组件。
   - **数据传递**：`Bundle.putXXX()`（基础类型）或 `Parcelable` / `Serializable`（对象）。
   - **请求返回**：`startActivityForResult()` 旧方案，Android 11+ 建议用 `ActivityResultLauncher`。

4. **Fragment**
   - **生命周期**：
     - `onAttach()` → `onCreate()` → `onCreateView()` → `onViewCreated()` → `onStart()` → `onResume()`
     - `onPause()` → `onStop()` → `onDestroyView()` → `onDestroy()` → `onDetach()`
   - **加载与切换**：`add()`, `replace()`, `remove()`, `show()`, `hide()`
   - **`FragmentManager` 与 `FragmentTransaction`**：
     - `beginTransaction()` → `commit()` / `commitNow()`
   - **`ViewPager2` + `FragmentStateAdapter`**：
     - 动态创建/销毁 Fragment，实现滑动翻页。

#### **Service**
1. **类型**
   - **前台服务**：调用 `startForeground()` 并显示通知，优先级高，不易被杀。
   - **后台服务**：单纯使用 `startService()`，系统内存不足时易被回收。

2. **启动方式**
   - `startService(Intent)`：在后台长期运行，需手动停止。
   - `bindService(Intent, ServiceConnection, flags)`：与客户端绑定，结束时自动停止。

3. **`onStartCommand()` 返回值**
   - `START_STICKY`：服务被杀后，系统尝试重启，Intent 可能为 null。
   - `START_NOT_STICKY`：不重启。
   - `START_REDELIVER_INTENT`：重启后重传最后一次 Intent。

4. **IntentService（已过时，建议 WorkManager）**
   - 适合处理短时间后台任务，自动在任务完成后停止。
   - 单线程执行队列任务。

5. **JobScheduler & WorkManager**
   - `JobScheduler`：基于系统条件（网络、充电等）定时或条件触发任务。
   - `WorkManager`：可串行/并行队列执行任务，支持重试和约束条件（推荐）。

#### **BroadcastReceiver**
1. **注册方式**
   - **静态注册**（Manifest）：常见于监听开机、安装、网络变更等系统广播。
   - **动态注册**（代码）：灵活，可在运行时根据需要进行注册和取消。

2. **本地广播（LocalBroadcastManager）**
   - 仅限于当前应用内部，不会被其他应用接收。
   - 更安全和高效。

3. **有序广播（Ordered Broadcast）**
   - 广播可以按优先级顺序传递，每个接收者可修改或终止广播。

4. **粘性广播（Sticky Broadcast）**
   - 已被废弃，不建议使用。

#### **ContentProvider**
1. **作用**
   - 在不同应用/进程之间共享数据。
   - 访问系统联系人、短信、媒体库等。

2. **使用 `ContentResolver` 进行 CRUD**
   - `insert()`, `query()`, `update()`, `delete()`
   - 通过 URI（content://authority/…）定位特定数据表或记录。

3. **自定义 `ContentProvider`**
   - 实现抽象方法并注册到 Manifest。
   - 用于自定义跨进程共享数据，例如 IM 应用。

### **1.2 Android 进程与线程**
1. **主线程（UI 线程）**
   - **严禁**在主线程执行耗时操作（网络/IO），否则导致 ANR。
   - 更新 UI 必须在主线程执行。
2. **Handler / Looper / MessageQueue**
   - `Looper.prepare()` → `Looper.loop()`：创建消息循环。
   - `Handler.sendMessage()` / `Handler.post(Runnable)`：把消息/任务投递到队列。
   - **MessageQueue** 依次分发消息给 `Handler`。
3. **AsyncTask（已弃用） & 替代方案**
   - 原理：线程池 + Handler 通信。
   - 替代：**Kotlin Coroutines**、`ExecutorService`、`WorkManager`。
4. **IntentService**
   - 单独线程执行队列任务，自动停止。
   - Android 11+ 不再推荐，优先使用 **WorkManager**。
5. **ANR（Application Not Responding）**
   - **触发条件**：主线程阻塞超过 5 秒。
   - **常见原因**：死锁、密集计算、IO、主线程等待。
   - **优化**：使用异步、减少耗时，`StrictMode` 检测。

---

## **2. UI 体系**

### **2.1 View 体系**
1. **View 的事件分发机制**
   - 流程：`Activity.dispatchTouchEvent()` → `ViewGroup.dispatchTouchEvent()` → `ViewGroup.onInterceptTouchEvent()` → `View.dispatchTouchEvent()` → `View.onTouchEvent()`。
   - `onInterceptTouchEvent()` 返回 `true` 时，子 View 不再接收事件。
2. **自定义 View**
   - **重写测量**：`onMeasure(int widthMeasureSpec, int heightMeasureSpec)`
   - **布局位置**：`onLayout()`（通常在自定义 ViewGroup 中）
   - **绘制**：`onDraw(Canvas canvas)`，配合 `Paint`、`Path`、`Canvas` API。
   - 性能优化：减少不必要的绘制，善用硬件加速。
3. **RecyclerView 优化**
   - **DiffUtil**：高效更新列表，减少全局刷新。
   - **setHasFixedSize(true)**：如果内容大小不变，可提高性能。
   - **ItemAnimator**：自定义动画扩展。
   - **LayoutManager**：LinearLayoutManager, GridLayoutManager, StaggeredGridLayoutManager。
   - **分页加载**：Paging 3 库，实现列表分页。

### **2.2 动画**
1. **属性动画（Property Animation）**
   - `ObjectAnimator`, `ValueAnimator`：通过修改对象属性 (alpha, translateX, scaleY 等) 实现动画。
   - `AnimatorSet`：组合多个动画并行或顺序执行。
   - 配合插值器 (Interpolator) 与估值器 (TypeEvaluator) 可实现复杂动画。
2. **补间动画（Tween Animation）**
   - 传统 XML 定义（scale, rotate, translate, alpha）。
   - 针对 View 的视觉变换，不影响点击范围。
3. **帧动画（Frame Animation）**
   - 连续播放多张 Drawable 图片。
   - 适合简易动画效果，性能较差。
4. **Lottie 动画**
   - 基于 JSON，支持矢量图形和更复杂的动画。
   - 体积小，跨平台性好。
5. **共享元素动画（Shared Element Transition）**
   - 在 Activity/Fragment 切换时，对相同元素进行连贯动画。
   - 通过 `setTransitionName()` 标识元素。

### **2.3 Jetpack Compose（现代 UI 方案）**
1. **声明式 UI**
   - `@Composable` 函数直接描述界面。
   - `State` 和 `remember {}` 管理状态，UI 与数据同步。
2. **布局与组件**
   - `Row`, `Column`, `Box`, `LazyColumn`, `LazyRow` 等。
   - `Modifier` 修饰大小、padding、背景、点击事件等。
3. **Recomposition**
   - 数据变化 → 触发 UI 重组。
   - 避免过度重组：`remember`、`derivedStateOf`、`LaunchedEffect`。
4. **Material3 主题**
   - 采用 Material Design 3，直接在 Compose 中定义 `colors`, `typography`, `shapes`。
5. **Navigation-Compose**
   - `NavHost`, `NavController`，在 Composable 间无缝跳转。

---

## **3. 数据存储**

### **3.1 本地存储**
1. **SharedPreferences vs DataStore**
   - **SharedPreferences**：基于 XML 文件；多并发写入可能导致 ANR。
   - **DataStore**：基于 `Flow`，支持异步读写；有 `Preferences DataStore` 和 `Proto DataStore`。
   - 代码示例：
     ```kotlin
     // Preferences DataStore 示例
     val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")
     // 读写
     val EXAMPLE_KEY = stringPreferencesKey("example")
     
     suspend fun saveExample(value: String) {
         context.dataStore.edit { prefs ->
             prefs[EXAMPLE_KEY] = value
         }
     }
     
     val exampleFlow: Flow<String> = context.dataStore.data
         .map { prefs -> prefs[EXAMPLE_KEY] ?: "" }
     ```

2. **Room 数据库**
   - 基于 SQLite 封装的 ORM。
   - 主要组件：
     - `@Entity` 数据模型类
     - `@Dao` 数据访问接口
     - `@Database` 定义数据库持有者
   - 提供编译时 SQL 语法检查，支持 **Coroutines**、**RxJava**、**LiveData**。
   - **示例**：
     ```kotlin
     @Entity
     data class User(
         @PrimaryKey val id: Int,
         val name: String
     )

     @Dao
     interface UserDao {
         @Insert
         suspend fun insertUser(user: User)
         @Query("SELECT * FROM User")
         fun getAllUsers(): List<User>
     }

     @Database(entities = [User::class], version = 1)
     abstract class AppDatabase : RoomDatabase() {
         abstract fun userDao(): UserDao
     }
     ```

3. **SQLite vs Room**
   - 手写 SQL vs 注解自动生成
   - 手动管理 Cursor vs 自动映射 Entity
   - Room 更安全高效，易维护

### **3.2 网络数据获取 & 缓存**
1. **Retrofit + OkHttp**
   - Retrofit：基于注解的网络请求（`@GET`, `@POST`, `@Field`, `@Path`）。
   - OkHttp：底层 HTTP 客户端，可配置拦截器、缓存、SSL。
2. **Gson / Moshi**
   - JSON 序列化/反序列化。
3. **WebSocket、GRPC**
   - WebSocket：双向实时通信，适合聊天、推送。
   - GRPC：基于 HTTP/2，可高效传输二进制数据。
4. **缓存策略**
   - OkHttp 缓存：Cache-Control 头或 `cache()` 配置。
   - 数据持久化：与 Room 结合 + Repository 模式实现离线缓存。

---

## **4. Framework 体系**

### **4.1 Android Framework 组件**
1. **Binder 机制**
   - Android 跨进程通信核心：C/S 架构。
   - Binder 驱动 & ServiceManager：将远程调用映射为本地对象。
2. **AMS（ActivityManagerService）**
   - 管理所有进程、Activity 栈、Service 等。
3. **PMS（PackageManagerService）**
   - 负责应用安装、卸载、权限校验、签名验证。
4. **WMS（WindowManagerService）**
   - 管理应用窗口与层级，控制窗口显示、动画、焦点。

### **4.2 系统优化**
1. **AOSP 源码**
   - 学习进程启动、四大组件调度、系统服务运行机制。
2. **ART vs Dalvik**
   - Dalvik：JIT 编译，较低效率。
   - ART：AOT + JIT 混合，提升性能与内存利用。
3. **系统启动流程**
   - Zygote 进程 fork 出应用进程。
   - SystemServer 启动关键服务 (AMS, PMS, WMS)。

---

## **5. Jetpack 组件**

### **5.1 Lifecycle, ViewModel, LiveData**
1. **Lifecycle**
   - `LifecycleOwner` (Activity/Fragment)，`LifecycleObserver` (监听生命周期事件)。
   - 避免在错误时机访问 UI，减少内存泄漏。
2. **ViewModel**
   - 在配置变更（如旋转）后依然存活，避免重复请求或数据丢失。
   - 存放与 UI 相关但不依赖 UI 的数据。
3. **LiveData / StateFlow**
   - LiveData：组件感知生命周期，自动通知观察者。
   - StateFlow：基于 Kotlin 协程的响应式流，更灵活，适合数据流。

### **5.2 Navigation**
1. **Navigation 组件**
   - `NavHostFragment` + `NavController` 管理页面跳转，支持动画、Deep Link。
   - SafeArgs：类型安全地传递参数，避免硬编码 Keys。
2. **Fragment 间通信**
   - `setResultListener()` 或 Shared ViewModel。

### **5.3 依赖注入：Hilt / Dagger 2**
1. **Dagger 2**
   - @Module, @Component, @Inject 注解；编译时生成依赖图。
   - 提升可维护性与可测试性。
2. **Hilt**
   - 封装 Dagger 2 的模板化流程，在 Android 上使用更方便。
   - @HiltAndroidApp, @AndroidEntryPoint, @InstallIn 等注解。

---

## **6. 高级优化**

### **6.1 启动速度优化**
1. **冷启动 vs 热启动 vs 温启动**
   - 冷启动：完全新进程，耗时最大。
   - 热启动：App 在后台，Activity 尚存在。
   - 温启动：进程存在但 Activity 被销毁。
2. **方法**
   - 减少 `Application` 中的初始化。
   - 延迟或按需加载模块。
   - 使用 `SplashScreen` API。

### **6.2 卡顿优化**
1. **性能分析工具**
   - **Systrace** / **TraceView**：跟踪线程调度、方法调用耗时。
   - **Choreographer**：监控帧率，检测掉帧。
2. **UI 过度绘制（Overdraw）**
   - 开发者选项中开启 GPU 过度绘制调试。
   - 减少层级、移除不必要背景。

### **6.3 内存优化**
1. **LeakCanary**
   - 检测 Activity、Fragment、View 泄漏。
2. **Bitmap 优化**
   - `inSampleSize` 缩放加载，`inBitmap` 复用内存。
3. **LRUCache**
   - 常用对象缓存，避免频繁创建。
4. **解决常见内存泄漏**
   - 静态持有 Context；Handler 引用外部类；未关闭资源等。

### **6.4 电量优化**
1. **任务合并**
   - 批量处理网络请求，减少唤醒 CPU 次数。
2. **WorkManager**
   - 避免频繁启动后台服务。
3. **Battery Historian**
   - 分析耗电热点。

---

## **7. CI/CD 与测试**

### **7.1 自动化测试**
1. **JUnit + Mockito**
   - 单元测试：业务逻辑、工具方法。
   - Mockito：Mock 依赖或网络请求，避免真实调用。
2. **Espresso**
   - UI 测试：模拟点击、滑动、输入，并验证 UI 状态。
3. **Robolectric**
   - 在 JVM 环境下模拟 Android，速度更快，但不完全真实。
4. **Instrumented Test**
   - 真机/模拟器环境下测试，最贴近真实用户场景。

### **7.2 持续集成/持续部署**
1. **GitHub Actions / Jenkins**
   - 配置流水线（Pipeline），自动构建 & 测试，节省人工。
2. **Fastlane**
   - 自动生成截图、打包、上传 Play Store。
3. **Firebase Test Lab**
   - 多机型云测试。

---

## **8. Android 适配**

### **8.1 版本适配**
1. **Android 12/13 新特性**
   - 通知权限、后台启动限制、剪贴板隐私等。
   - 对新 API / 权限变更进行兼容处理。
2. **Scoped Storage**
   - 无法随意访问外部存储，使用 `MediaStore` 或 SAF。
3. **App Bundle (AAB)**
   - 用于分包分发，按需下载。

### **8.2 屏幕适配**
1. **多分辨率**
   - dp、sp、px 概念；常用资源目录：`drawable-hdpi`, `drawable-xxhdpi` 等。
   - ConstraintLayout，RelativeLayout 适配不同屏幕。
2. **折叠屏/平板适配**
   - `res/layout-sw600dp` 等资源目录区分大屏。
   - 注意多窗口模式 & 横向布局。
3. **RTL 适配**
   - `android:supportsRtl=\"true\"`，在布局中使用 `start` 和 `end` 而非 `left` 和 `right`。

### **8.3 国际化**
1. **字符串资源**
   - `strings.xml`，`plurals.xml`，在不同 `values-xx` 文件夹中存放翻译文本。
2. **Locale 切换**
   - 动态切换语言，需考虑配置更改后 Activity 重建。
3. **多语言发布**
   - Google Play Console 地区配置，自动匹配语言。

