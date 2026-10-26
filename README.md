[Uploading README-开发文档.md…]()
# fakelocation-by-jucie
基于各位大佬开发的虚拟定位
# 向量定位 VectorLoc

基于 **Vector**（[JingMatrix/Vector](https://github.com/JingMatrix/Vector)，Magisk/KernelSU 上的 Zygisk ART Hook 框架）重构的 Android 虚拟定位 + 环境伪装工具。

- **不使用 mock provider**：直接注入 `system_server` 定位服务，`Location.isFromMockProvider` 天然为 false。
- **配置实时下发**：libxposed remote preferences 通道，改完立即生效；模块更新还能**热重载**，不用重启。
- **定位 + 环境一起伪**：位置、卫星、WiFi、基站、BLE 广播、AppOps 痕迹，一次配齐。
- **三段式架构**：纯函数运动引擎 → 单一配置源 → 双端（UI / Hook）各自算出一致坐标。

> ⚠️ 合法使用提醒：本项目只应用于自有应用、已获授权的测试环境。请勿用于伪造考勤、伪造运动记录、欺诈或未经授权干预第三方服务。使用者自行承担合规责任。

---

## 1. 目录结构

```
VectorLoc/
├── core/                         纯 Kotlin（JVM），App 与 Hook 共用
│   ├── Config.kt                 配置模型（含 WiFi/基站/BLE 环境数据）+ 坐标换算 + 假 MAC 生成
│   └── Motion.kt                 球面几何、路线引擎、抖动噪声、Fix 快照
├── xposed/                       Hook 模块（编译进同一个 APK）
│   ├── VectorLocModule.kt        libxposed 入口：进程分流 + 热重载
│   ├── Channel.kt                配置通道（remote preferences）
│   ├── SystemLocationHooks.kt    定位注入（核心）
│   ├── SystemGnssHooks.kt        卫星状态伪造
│   ├── EnvFactory.kt             反射构造 ScanResult / WifiInfo / CellInfo / BLE ScanResult
│   ├── EnvHooks.kt               WiFi 服务端 / 基站（com.android.phone）/ App 进程环境 hook
│   ├── AntiDetect.kt             AppOps / Settings.Secure 痕迹清理
│   ├── AppHooks.kt               目标 App 进程内补强
│   ├── LocationFactory.kt        反射构造/改写 android.location.Location
│   └── Reflect.kt / Targets.kt   反射工具 + 目标判定
└── app/                          UI 与前台服务
    ├── data/Repository.kt        状态仓库 + 发布到框架
    ├── service/SpoofService      会话保活
    ├── service/JoystickService   悬浮摇杆（自绘 View）
    └── ui/
        ├── MapPane.kt / MapEngine.kt   地图引擎分发
        ├── AmapPane.kt                 高德引擎（GCJ-02 换算 + 隐私合规）
        ├── EnvSheet.kt                 WiFi / 基站 / 蓝牙配置面板
        ├── JumpDialog.kt               坐标跳转 + OSM 地点搜索
        └── MainScreen.kt               主界面
```

### 为什么把 Hook 代码放进 App 模块一起打包

Vector 加载模块的方式是扫描 APK 里的 `META-INF/xposed/` 描述文件，再按 `java_init.list` 实例化入口类，所以**模块代码必须和 App 在同一个 APK 里**：

| 文件 | 内容 |
|---|---|
| `app/src/main/resources/META-INF/xposed/module.prop` | `minApiVersion=100` / `targetApiVersion=102` / `autoHotReload=true` |
| `app/src/main/resources/META-INF/xposed/java_init.list` | `com.vectorloc.xposed.VectorLocModule` |
| `app/src/main/resources/META-INF/xposed/scope.list` | 默认作用域：`system`、`com.android.phone` |

`libxposed:api` 用 `compileOnly` 引入（运行时由 Vector 提供，**绝不能打进 APK**）；`libxposed:service` 用 `implementation`（App 侧靠它连框架，附带声明的 `XposedProvider` 要合并进清单）。

---

## 2. 编译

要求：JDK 17、Android SDK Platform 35、Gradle 8.13。

```bash
export JAVA_HOME=/path/to/jdk17
export ANDROID_HOME=/path/to/Android/Sdk

gradle :app:assembleDebug
gradle :core:test :app:assembleRelease

# 带百度地图引擎（默认关闭，见下方说明）
gradle :app:assembleRelease -PwithBaidu=true
```

### 两个和内存有关的坑（实测踩过）

1. **百度地图 SDK 默认不编**：`BaiduMapSDK_Map` 7.6.4 的 AAR 有 33 MB，和已存在的高德 SDK 一起 dex 时，
   D8 需要 5 GB 以上的堆，小内存机器会直接 `D8: java.lang.OutOfMemoryError`。
   所以百度引擎放在独立 source set（`app/src/baidu/`），只有 `-PwithBaidu=true` 才参与编译；
   不编时 UI 上会提示"百度引擎未随本次构建启用"，其它引擎不受影响。
2. **不要加 `material-icons-extended`**：它包含上千个图标，往 dex 里塞十几 MB，是本项目构建 OOM 的
   主因（去掉后同样的依赖在 3 GB 堆下就能编过）。项目里用 `AppIcons` 只映射 `material-icons-core` 的基础图标。

或者直接跑 `scripts/build.ps1`（Windows 一键）。

产物在 `dist/`：

| 文件 | 大小 | 说明 |
|---|---|---|
| `dist/VectorLoc-1.3.4-release.apk` | 46.7 MB | 当前版本（含高德 + 百度 + OSM 三引擎，debug 证书签名可直接装） |

ABI 只保留 `arm64-v8a`（本机型 SM8650），两个 SDK 的多 ABI 原生库都已裁掉。

> `libxposed:service/interface:102.0.0` 的 AAR 元数据要求 `minCompileSdk=37`，本项目只用其中少量稳定 API，
> 因此在 `gradle.properties` 用 `android.experimental.disableCompileSdkChecks=true` 关掉强校验，保持 `compileSdk 35`。

**机型适配**：见 [`docs/适配说明-TB710FU.md`](docs/适配说明-TB710FU.md)（联想小新 Pad Pro GT / Android 16 / ZUI 17.5）。

---

## 3. 安装与启用

1. Magisk / KernelSU 装好 Zygisk 与 Vector（Vector 需要较新 Magisk + [NeoZygisk](https://github.com/JingMatrix/NeoZygisk)）。
2. `adb install -r dist/VectorLoc-1.1.0-release.apk`。
3. Vector 管理器 → 启用「向量定位」→ **作用域**勾选：
   - 「**系统**」（`system`）—— 定位 / 卫星 / WiFi / 反检测，**必选**；
   - 「**com.android.phone**」—— 基站伪造，需要才有；
   - 再按需勾选目标 App（走高强度反检测 + App 进程内的环境 hook）。
4. 重启一次（system_server 注入只在开机发生；之后模块更新走热重载，不必再重启）。

生效判断：

```bash
adb logcat -s VectorLoc
```

期望：

```
模块载入: process=system_server systemServer=true api=102
配置通道已就绪，group=vectorloc_config
WiFi 服务端 hook ×2
system_server hook 完成: LocationManagerService.getLastLocation×1 ... acceptLocationChange×N
```

---

## 4. 使用

| 功能 | 操作 |
|---|---|
| 选点 | 点地图任意位置；右上角 🔍 可输入经纬度或按名称搜索（OSM Nominatim） |
| 画路线 | 点面板上的「**画路线**」开关 → **单击**地图连续加航点（长按也行）；≥2 个点自动连成线（**红色**），显示总里程 |
| 地图缩放/平移 | 地图**右侧窄栏**：`+` / `−` 缩放（最大 21，瓦片超采样）+ 小摇杆平移视野；不触地图手势，**不会误加航点** |
| 移动摇杆 | 面板上的**大摇杆**：按住往哪推，**定位点就往哪走**（不是平移地图）。速度 = 当前模式的速度 × 推杆幅度 |
| 微调步进 | 摇杆下面的 `↑ ↓ ← →`：每点一次定位点走 **5 m**；开「画路线」时每步自动落一个航点——用它画直线段最省事 |
| 跑路线 | 点「跑起来」，默认走完原路折返（`routeLoop`）；「清空」重置航点 |
| 实时位置 | 开启模拟后地图上多一个**蓝点**，与目标 App 拿到的位置**同源同算法**（同一 `MotionEngine.fix`），路线模式下沿红线移动；顶部读数也会跟着变（前缀 ▶） |
| 收藏路线 | 画好航点后点「**存路线**」→ 输名字保存；⭐ 面板顶部的「收藏路线」里可**载入**或删除 |
| 运动模式 | 定点 / 步行 / 跑步 / 骑行 / 驾车，或拖速度条自定义 |
| 摇杆 | 点「摇杆」→ 授权悬浮窗 → 拖动圆盘实时转向调速 |
| 收藏地点 | 书签按钮保存当前点，⭐ 面板的「收藏地点」里点击跳转 |
| 环境伪装 | 右上角 ➕ 进入，分别配置 WiFi / 基站 / 蓝牙 |
| 高级设置 | ⚙ 里切地图引擎、全局/分应用、真实度、抖动、卫星数、抹痕迹开关 |

**定位点和路线都是自己画的**（不依赖 osmdroid 的图标/图形内部实现）：

| 元素 | 实现 | 为什么自己画 |
|---|---|---|
| 定位点 | `DotIcon.pin()`：Canvas 画"白环 + 绿芯 + 投影" | osmdroid 默认 marker 图标要 `org.osmdroid.library.R.drawable.marker_default`，资源没合并时 `icon` 为 null，`draw()` 直接跳过 → "能定位但看不到点" |
| 航点 | `DotIcon.dot()` | 同上 |
| 路线 | `RouteOverlay`：`Projection.toPixels` + `Canvas.drawPath` | `Polyline` 的绘制依赖内部 `mPaint`/`mOutlinePaint` 组合，6.1.20 上设了 `outlinePaint` 仍画不出线（实测：航点数和里程都对，就是没线，地图区绿色像素 408≈只有定位点） |

`RouteOverlay` 先画 18px 深色衬底再画 10px 绿色主线（圆头圆角），然后给每个航点画白边蓝芯圆点；
投影未就绪（`width/height<=0`）时跳过绘制，避免画出屏幕外的怪线。

实测修复前后（同一份 9 点 / 358 m 的路线）：地图区域绿色像素 **408 → 4284**。

实时性：配置写入后 Hook 侧 < 300 ms 生效（配置轮询 250 ms、坐标推送 400 ms）；UI 与第三方 App 用同一份 `MotionEngine.fix(config, now)` 计算坐标，因此看到的位置一致。

> 缩放/平移控件为什么放在**右侧窄栏**而不是浮在地图上：`AndroidView` 里的 osmdroid 会把触摸全部吃掉，
> 浮层控件收不到点击，反而会被当成"在地图上点了一下"→ 每点一次就多一个航点（实测踩过：9 个点变 12 个）。

### 横竖屏适配

| 方向 | 布局 |
|---|---|
| 竖屏 | 上方地图（weight 1）+ 下方控制栏 |
| 横屏 | 左侧地图（约 75%）+ **右侧控制栏**（≤ 460dp），避免控制栏把地图压扁 |

两个实现要点：
1. `MainActivity` 声明了 `android:configChanges="orientation|screenSize|screenLayout|smallestScreenSize|keyboardHidden|uiMode|density|navigation"`，
   **旋转不重建 Activity**——否则 osmdroid 会重新建 MapView、瓦片重新下载、`recenter`/画路线状态全丢（实测 pid 保持一致）；
2. 控制面板里「开始模拟 / 摇杆 / 收藏」这一行**固定在面板底部**，上半部分（坐标/模式/速度/路线）单独滚动。
   否则横屏时内容比屏幕高，主按钮会被挤出可视区（这个坑实测踩过：横屏下 dump 不到「开始模拟」）。

布局用 `BoxWithConstraints` 按可用宽高比判断，不依赖设备类型，平板分屏/自由窗口也能自适应。

### 路线运动为什么是平滑的

里程必须对**速度做时间积分**，不能用「瞬时速度 × 总时间」。速度带 ±N% 的正弦浮动时，后者在噪声变负的瞬间会让里程**倒退**，表现就是位置来回跳（实测踩过）。

现在是闭式解：

```
s(t) = base·t + base·pct·∫₀ᵗ noise(0.6τ)dτ
```

`noise` 是三个正弦之和，其原函数 `noiseIntegral` 也是解析解，所以
`s(t)` 单调递增、且任意两端用同一 `(config, now)` 算出的结果完全一致（UI 与 Hook 不会出现"两边不一样"）。
另外推送频率从 1 Hz 提到 **2 Hz**（500ms），走起来更跟手。

单测 `route progress is monotonic even with speed fluctuation` 专门守这个回归：3 m/s、30% 浮动、
走 300 秒，位置不允许出现任何倒退。

---

## 5. 定位 Hook 原理

**注入位置**：`system_server`（`onSystemServerStarting`），不是 mock provider。

| 层 | 目标 | 做法 |
|---|---|---|
| 同步拉取 | `LocationManagerService.getLastLocation` | 替换返回值为假 `Location` |
| 订阅推送 | `registerLocationListener` / `requestLocationUpdates` | 登记回调（IBinder 去重 + `linkToDeath`），注册瞬间推首帧，之后每秒一帧；真实注册永远 `proceed` |
| 单次定位 | `getCurrentLocation` | 立即回调假坐标，同时放行真实请求兜底 |
| 可用性自检 | `isProviderEnabled` / `isLocationEnabledForUser` | 对目标调用者返回 `true` |
| fused 路径 | `LocationProviderManager.getLastLocation` | 返回包装好的 `LocationResult`（**必须先 `deoptimize`**，否则被内联） |
| 交付链路 | `LocationProviderManager$*Registration.acceptLocationChange` | 就地改写 `LocationResult` 里的每个 `Location`，防真实数据盖掉假坐标 |
| 卫星状态 | `registerGnssStatusCallback` | 直接给 `IGnssStatusListener.onSvStatusChanged(...)` 喂合成数组 |
| 反检测 | `AppOpsService` / `Settings.Secure` | `OP_MOCK_LOCATION(58)` 恒 `MODE_IGNORED`；`mock_location` 恒 `"0"` |

**构造假 Location 的坑**（`LocationFactory.kt` 已处理）：`Location` 是 final 只能反射构造；`setTime` / `setElapsedRealtimeNanos` **必须每次刷新为"现在"**，否则运动类 App 丢点；`extras` 要补 `satellites`；一律用目标进程的 ClassLoader。

**目标判定**（`Targets.kt`）：`uid > 1000` + 不在系统豁免名单 + 全局模式（或命中分应用作用域）。GMS 故意**不**豁免，否则用 `FusedLocationProviderClient` 的 App 收不到假坐标。

---

## 6. 环境伪装原理（WiFi / 基站 / 蓝牙）

三条环境线的挂载位置不同，是因为它们跑在不同进程：

| 目标 | 挂在哪 | 覆盖接口 |
|---|---|---|
| WiFi | `system_server` 的 `WifiServiceImpl` | `getScanResults`、`getConnectionInfo` |
| WiFi（补强） | 目标 App 进程 | `WifiManager.getScanResults / getConnectionInfo` |
| 基站 | **`com.android.phone`** 的 `PhoneInterfaceManager` | `getAllCellInfo` |
| 基站（补强） | 目标 App 进程 | `TelephonyManager.getAllCellInfo` |
| 蓝牙 | 目标 App 进程 | `BluetoothLeScanner.startScan` → 主动回调假 `ScanResult` |

**语义统一**：开关打开 = 接管；**列表为空 = 隐藏真实环境**（扫到的就是空）。这本身就是一种反追踪需求。

**构造方式是「多候选 + 逐级降级」**（`EnvFactory.kt`）：`ScanResult` / `WifiInfo` / `CellIdentity*` 都是 @hide 且字段逐版本改名，所以
1. 先试无参私有构造 + 按多个候选字段名写入（`mWifiSsid` / `mSSID`、`mMssid`/`mBssid`、`mMcc`/`mMccStr`…）；
2. `WifiSsid` 依次尝试 `fromBytes` / `fromString` / `createFromAsciiEncoded` / `(byte[])` 构造；
3. BLE 走**公开**构造 `ScanResult(BluetoothDevice, ScanRecord, int rssi, long)`，`ScanRecord` 用 `parseFromBytes` 合成一段合法广播（Flags + 完整本地名）；
4. 全部失败就返回 null → 调用方退回真实数据（或空列表），**绝不抛异常打断系统服务**。

所以如果某台机器上「打开开关但 App 仍看到真实 WiFi」，看日志里 `未找到 WifiServiceImpl` / `hook ×0`，那就是该 ROM 改了实现，按第 9 节的办法加候选即可。

---

## 7. 热重载

模块 `module.prop` 里 `autoHotReload=true`，因此：

- **改配置**：一直是实时的（remote preferences），跟热重载无关；
- **更新模块代码**（`adb install -r` 新 APK）：Vector 会触发热重载，**无需重启**。

实现要点（`VectorLocModule.kt`）：

| 回调 | 做什么 |
|---|---|
| `onHotReloading`（旧代码里执行） | 停掉定位推送线程 / 卫星推送线程 / 配置轮询线程，解注册监听 → 返回 `true` 允许重载 |
| `onHotReloaded`（新代码里执行） | 先 `oldHookHandles.forEach { unhook() }`（覆盖了默认实现，得自己卸），再重新装 hook |

system_server 的新一代代码拿不到 `onSystemServerStarting` 的参数，所以 ClassLoader 是从 `ServiceManager.getService("location")` 的 Binder 对象反查的。在非 system_server 进程里，热重载只恢复 `AppHooks`，**环境 hook 需要该 App 重启一次**才回来（日志会写明）。

---

## 8. 地图引擎

| 引擎 | 坐标系 | 需要 Key | 说明 |
|---|---|---|---|
| **高德·免Key（默认）** | GCJ-02 | 不需要 | 高德栅格瓦片，国内 200ms 级响应；瓦片服务仅建议个人测试使用 |
| OSM | WGS-84 | 不需要 | `tile.openstreetmap.org` 在国内**经常直接超时**（白屏），仅海外或代理环境可用 |
| 高德·SDK | GCJ-02 | 需要 `amap.key` | 高德 3D SDK，功能最全 |
| 百度·SDK | BD-09 | 需要 `baidu.key` | BaiduMapSDK_Map 7.6.4（走阿里云镜像拉取） |

> ⚠️ **v1.4.5 起自动回退**：如果选中了需要 Key 的引擎但 Key 是空的，会自动改用「高德·免Key」，
> 并在底部弹一条「XXX 缺少 Key，已临时改用…」的提示。
> 踩过的坑：引擎被切到 `BAIDU` 而没配 Key 时，地图是白底、**路线也画不出来**（百度引擎走的是另一套渲染路径，
> 不经过 osmdroid 的 `RouteOverlay`），很容易误判成"路线功能坏了"。

配置 Key（`local.properties`，改完重新编译）：

```properties
amap.key=你的高德Key
baidu.key=你的百度Key
```

**没有 Key 时**：高德/百度引擎会显示白底水印并盖一层提示，切回「高德·免Key」即可正常看图。

代码里几处关键处理：
- 高德 SDK：`MapsInitializer.updatePrivacyShow/updatePrivacyAgree` 必须在创建 `MapView` 之前调用，否则白屏；
- 百度 SDK：`SDKInitializer.setAgreePrivacy` 同理；且它的 `MapView.onCreate` 是 `(Context, Bundle)` 双参、点击回调是双方法接口；
- 坐标换算：配置统一存 WGS-84，各引擎显示/回写各自转换（GCJ-02 / BD-09），否则国内会偏 500 米。

---

## 9. ⚠️ 国内地图 App 为什么"还是真实位置"

实测踩到过，三层原因，按顺序排查：

### 9.1 先确认 `active=true`

`active=false` 时模块**一律透传真实定位**，所有 hook 都不介入。判断：

```bash
adb logcat -s VectorLoc | grep 下发坐标
```

有 `下发坐标[tick] lat=…` 才说明真的在模拟；一条都没有就是没开（App 里点「开始模拟」）。

> v1.3.0 起 `active` 状态会持久化，App 被系统杀掉再打开不会自动关掉。

### 9.2 百度地图 / 高德地图要加进作用域

`system_server` 的 hook 只能改"系统给的定位"。国内地图 App 还会：

- 自己扫 WiFi/基站做指纹，上传自家服务器换坐标（**完全不经过 LocationManager**）；
- 把结果封成自家对象（`BDLocation` / `AMapLocation` / `TencentLocation`）。

v1.3.0 为此加了 **SDK 层 hook**（`SdkLocationHooks.kt`）：直接改写这些对象的
纬度/经度/速度/精度/时间/定位类型，并清空地址字段（避免泄露真实城市街道）。
前提是**该 App 在 Vector 作用域里**：

```bash
adb shell su -c '/data/adb/lspd/cli scope add com.vectorloc.app com.baidu.BaiduMap/0'
adb shell am force-stop com.baidu.BaiduMap      # 下次启动才会被注入
```

> ⚠️ **v1.3.4 修的关键坑**：这些 SDK 都经过 R8 优化，`getLatitude()` 这类字段 getter 会被**内联**，
> 只装 hook 不会触发——必须先 `module.deoptimize(method)`。
> 修复前百度地图一直显示 0,0（几内亚湾），修复后日志能看到：
>
> ```
> SDK 定位 hook 命中: BDLocation#getLatitude / getLongitude / getLocType / getRadius …
> ```

### 9.4 "位置在两个点之间闪" 怎么治

真实 provider 的推送会绕过 Hook 直接到 App，和我们 2Hz 的假推送交替出现，表现就是**来回闪**。
AOSP 12 的解法是 hook `LocationProviderManager$*Registration.acceptLocationChange`，但
**Android 16 这台机器上 `LocationProviderManager` 根本没有可枚举的内部类**（日志 `LPM 内部类:` 为空），
按名字永远挂不上。

v1.4.1 起改用不依赖 ROM 内部结构的做法：hook AIDL 生成的
`android.location.ILocationListener$Stub$Proxy.onLocationChanged` 与
`ILocationCallback$Stub$Proxy.onLocation`，把所有发给 App 的定位就地改写：

```bash
adb logcat -s VectorLoc | grep "交付改写命中"
# 交付改写命中: Proxy#onLocationChanged（真实推送已被替换为假坐标）
```

配合两方面降低"闪"的观感：

- 运动时抖动幅度压到 0.15 倍（`MotionEngine.jitterOffset`），噪声频率降到 0.10/0.12；
- 精度字段只在 ±0.15m 内缓慢变化（`accuracyFor`），避免某些 SDK 判定"信号不稳"丢点。

验证命中情况（**这是判断 SDK 层是否生效的唯一可靠信号**）：

```bash
adb logcat -s VectorLoc | grep "SDK 定位 hook 命中"
```

### 9.3 WiFi / 基站要一起伪造

`spoofWifi` / `spoofCell` 打开但列表为空 = **隐藏真实环境**——这正好让百度/高德的
WiFi 指纹定位失败，退回系统定位、吃我们的假坐标。这两项关着的话，地图 App 拿到真实
WiFi 指纹就会显示真实位置。

---

## 10. 不要同时启用多个定位模块

Vector 里**同时启用多个**会 hook 定位的模块（原版 LocationSpoofer、Portal、allmap、本模块）时，
它们抢同一批方法，谁最后挂上谁说了算，表现是**位置乱跳，或稳定显示 0,0（几内亚湾 / 赤道几内亚）**。

```bash
adb shell su -c '/data/adb/lspd/cli modules ls'
```

只保留一个定位模块再重启。

---

## 11. 排查

```bash
adb logcat -s VectorLoc
adb shell dumpsys location | head -60
```

| 日志 / 现象 | 含义 | 处理 |
|---|---|---|
| 没有 `模块载入` | 模块没被注入 | Vector 未启用 / 作用域没勾「系统」/ 没重启 |
| `未找到 LocationManagerService` | ROM 改了类名 | 在 `SystemLocationHooks.kt` 的 `Reflect.findClass(...)` 候选数组里加 |
| `xxx 未挂上（该版本可能没有此方法）` | 该方法名在该版本不存在 | 关注 `getLastLocation` 是否 `×1` |
| `LPM.getLastLocation ... 失败` | `deoptimize` 失败 | 看堆栈，必要时改挂 `getLastLocationUnsafe` |
| `能定位但偶尔跳真实位置` | `acceptLocationChange×0` | 该 ROM 改了内部类名，放宽 `declaredClasses` 过滤 |
| `未找到 WifiServiceImpl` | WiFi 服务被搬到 APEX 私有实现 | 加候选类名；或依赖 App 进程侧的 `WifiManager` hook |
| `未找到 PhoneInterfaceManager` | 当前进程不是 `com.android.phone` | 作用域里补勾 `com.android.phone` |
| `hook ×0`（环境） | 构造能力不足 | 看 `EnvFactory` 的 `Logs.d`，说明该版本字段全变了 |
| `高德引擎未配置 Key` | 没配 `amap.key` | 见第 8 节，或切回 OSM |
| App 显示「未连接框架服务」 | App 与框架没连上 | 确认 Vector 支持 libxposed api 102；重装后重启 |

---

## 12. 已知限制

- **Android 12+「大致位置」**：目标 App 只有 `ACCESS_COARSE_LOCATION` 时，系统会在 App 进程内把坐标模糊到约 2 km 网格。给目标 App 授「精确位置」即可。这是系统行为。
- **环境伪装的构造层依赖 ROM 版本**：`ScanResult` / `CellIdentity*` 字段逐版本改名，已做多候选 + 降级，但深度定制 ROM 仍可能需要补候选字段名（日志会指出）。
- **基站在 `com.android.phone` 进程内 hook**：不勾该作用域就只有 App 进程侧的补强路径。
- **热重载只覆盖 system_server 与 `AppHooks`**；App 进程内的环境 hook 需目标 App 重启。
- **ABI 只保留 arm64-v8a**（按本机型裁剪，同时压掉高德多 ABI 原生库）。

---

## 13. 许可证

GPL-3.0。设计思路参考：

- [HuangZhuoRui/LocationSpoofer](https://github.com/HuangZhuoRui/LocationSpoofer)（GPL-3.0，已归档）—— Hook 目标清单、双端一致的运动引擎、remote preferences 通道、环境伪造的分层策略；
- [JingMatrix/Vector](https://github.com/JingMatrix/Vector)（GPL-3.0）—— Zygisk ART Hook 框架；
- [libxposed/api](https://github.com/JingMatrix/libxposed-api) —— 模块 API 102（含热重载）。

代码为重新实现，未直接复制上游源码；继续分发改动版本请同样以 GPL-3.0 开源。
