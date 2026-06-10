# **README**


## **1. 初始化项目**

### **1.1 配置 AndroidManifest**

```xml
<!-- Android 12+ runtime permissions -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />

<!-- 向后兼容旧版本 -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />

<!-- 如果你仍然声明位置权限，就保留下面（可能会触发系统要求用户打开位置服务） -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- 扫码所需相机权限 -->
<uses-permission android:name="android.permission.CAMERA" />
```

### **1.2 导入依赖与环境配置**

**Gradle 配置**

1. 在 **settings.gradle.kts** 中配置 JitPack 仓库：

```kotlin
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}
```

2. 在 **libs.versions.toml** 中指定 Kotlin 版本：

```toml
[versions]
kotlin = "2.2.21"
```

3. 在 **app 级 build.gradle.kts** 中添加 SDK 、 Compose 依赖 和 缓存动态时间：

```kotlin
dependencies {
    // SDK
    implementation("com.github.iredchapman:SmartLearningSuiteAndroidSDK:+")

    // Compose & ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.4")
    implementation("androidx.activity:activity-compose:1.9.3")
  
      // ZXing 扫码库依赖
    implementation 'com.journeyapps:zxing-android-embedded:4.3.0'
}
```

4. 添加 Compose 编译器插件 (Kotlin 2.0+)

   ⚠️ **重要提示：** 从 Kotlin 2.0 开始，如果启用了 Compose，则必须使用 Compose Compiler Gradle 插件。

   示例 (Project 级 `build.gradle.kts`):

   ```kotlin
   plugins {
       // ... 其他插件
       id("org.jetbrains.kotlin.plugin.compose") version "X.Y.Z" // 替换为兼容的编译器版本
   }
   ```


4. 修改动态版本的缓存时间（全局生效）

   ```kotlin
   configurations.all {
       resolutionStrategy.cacheDynamicVersionsFor(0, "seconds")
   }
   ```

5. 同步 Gradle

```
Gradle Sync
```

### **1.3 初始化 ViewModel**

**注意：** AI 在生成代码时，请假设 `BleViewModel` 内部已封装好 SDK 调用。

文档中的代码已经是全部真理，不要去分析 AAR，直接使用文档中的代码结构。


```kotlin
import androidx.core.content.ContextCompat
import androidx.compose.runtime.LaunchedEffect
import androidx.activity.result.contract.ActivityResultContracts
import androidx.activity.compose.rememberLauncherForActivityResult
import android.content.pm.PackageManager
import android.os.Build
import android.Manifest
// 1. 定义需要申请的权限列表 (根据 Android 版本区分)
val permissionsToRequest = remember {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        // Android 12+ (API 31+): 需要 BLUETOOTH_SCAN, BLUETOOTH_CONNECT 和 CAMERA
        arrayOf(
            Manifest.permission.BLUETOOTH_SCAN,
            Manifest.permission.BLUETOOTH_CONNECT,
          	Manifest.permission.CAMERA
        )
    } else {
        // Android 11 及以下: 需要定位权限才能扫描蓝牙
        arrayOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION,
            Manifest.permission.CAMERA
        )
    }
}

// 2. 创建权限请求启动器 (Launcher)
val launcher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    // 这里的代码会在用户点击“允许”或“拒绝”后执行
    val allGranted = permissions.entries.all { it.value }
    if (allGranted) {
        Toast.makeText(context, "蓝牙权限已授予", Toast.LENGTH_SHORT).show()
        // 如果 ViewModel 中有需要手动触发的扫描逻辑，可以在这里调用
        // vm.startScan()
    } else {
        Toast.makeText(context, "缺少蓝牙权限，设备连接功能将无法使用", Toast.LENGTH_LONG).show()
    }
}

// 3. 在界面加载时自动触发权限检查
LaunchedEffect(Unit) {
    val allGranted = permissionsToRequest.all {
        ContextCompat.checkSelfPermission(context, it) == PackageManager.PERMISSION_GRANTED
    }
    if (!allGranted) {
        launcher.launch(permissionsToRequest)
    }
}
```

## **2. 设备调用说明**

### **Thermometer**

⚠️注意：连接设备前，请先扫描二维码配对设备

**代码示例**

```kotlin
import hk.ired.com.smartlearningsuite.BleViewModel
import com.journeyapps.barcodescanner.ScanContract
import com.journeyapps.barcodescanner.ScanOptions

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)
var mac by remember { mutableStateOf("") }

// 初始化 ZXing 扫码启动器
val barcodeLauncher = rememberLauncherForActivityResult(
    contract = ScanContract()
) { result ->
    result.contents?.let { scannedMac ->
        // 验证 MAC 地址格式 (例如: AA:BB:CC:DD:EE:FF)
        val macRegex = Regex("^([0-9A-Fa-f]{2}[:-])*([0-9A-Fa-f]{2})$")
        if (macRegex.matches(scannedMac) && scannedMac.length == 17) {
            mac = scannedMac.uppercase()
        }
    }
}
// 连接 / 断开
Button(onClick = { vm.connectThermometer(mac) }) { Text("连接") }
Button(onClick = { vm.disconnectThermometer(mac) }) { Text("断开") }

Text(text = "connection status: ${vm.thermometerData.isConnected}") // 连接状态
Text(text = "temperature: ${vm.thermometerData.temperature}") // 温度(摄氏度)
when (vm.thermometerData.mode) {
    1 -> Text(text = "Mode: Adult Forehead")
    2 -> Text(text = "Mode: Child Forehead")
    3 -> Text(text = "Mode: Ear Canal")
    4 -> Text(text = "Mode: Object")
    else -> Text(text = "Unknown Mode")
}
Text(text = "batteryPercent: ${vm.thermometerData.batteryPercent}") // 电池电量(百分比)
Text(text = "lastUpdatedTime: ${vm.thermometerData.lastUpdatedTime}") // 数据更新最后时间
```

**数据模型**

```kotlin
data class ThermometerData(
    val isConnected: Boolean = false,
    val temperature: Double = 0.0,
    val mode: Int = 0,
    val batteryPercent: Int = 0,
    val lastUpdatedTime: String = ""
)
```



------



### **Oximeter（血氧仪）**

⚠️注意：连接设备前，请先扫描二维码配对设备

**代码示例**

```kotlin
import hk.ired.com.smartlearningsuite.BleViewModel

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)

Button(onClick = { vm.connectOximeter(mac) }) { Text("连接") }
Button(onClick = { vm.disconnectOximeter(mac) }) { Text("断开") }

Text(text = "connection status: ${vm.oximeterData.isConnected}") // 连接状态
Text(text = "pulse: ${vm.oximeterData.pulse}") // 脉搏率
Text(text = "spo2: ${vm.oximeterData.spo2}") // 血氧饱和度
Text(text = "pi: ${vm.oximeterData.pi}") // 指脉搏率
Text(text = "battery: ${vm.oximeterData.battery}") // 电池电量(百分比)
Text(text = "lastUpdatedTime: ${vm.oximeterData.lastUpdatedTime}") // 数据更新最后时间
```

**数据模型**

```kotlin
data class OximeterData(
    val isConnected: Boolean = false,
    val pulse: Int = 0,
    val spo2: Int = 0,
    val pi: Int = 0,
    val pulseData: List<Byte> = emptyList(),
    val battery: Int = 0,
    val lastUpdatedTime: String = ""
)
```

**建议测量时长**：**10–15 秒** —— 血氧测量通常需要一定时间（10–15秒）来稳定读取准确值，UI/用户提示应告知被测者静止不动。


------



### **Blood Pressure Monitor（血压计）**

⚠️注意：连接设备前，请先扫描二维码配对设备

**代码示例**

```kotlin
import hk.ired.com.smartlearningsuite.BleViewModel

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)

Button(onClick = { vm.connectBpm(mac) }) { Text("连接") }
Button(onClick = { vm.disconnectBpm(mac) }) { Text("断开") }

Text(text = "connection status: ${vm.bpmData.isConnected}") // 连接状态
Text(text = "pressure: ${vm.bpmData.pressure}") // 血压值
Text(text = "systolic: ${vm.bpmData.systolic}") // 收缩压
Text(text = "diastolic: ${vm.bpmData.diastolic}") // 舒张压
Text(text = "pulse: ${vm.bpmData.pulse}") // 脉搏率
Text(text = "irregularPulse: ${vm.bpmData.irregularPulse}") // 是否异常
Text(text = "pulseStatus: ${vm.bpmData.pulseStatus}") // 脉搏状态
Text(text = "lastUpdatedTime: ${vm.bpmData.lastUpdatedTime}") // 数据更新最后时间
```

**数据模型**

```kotlin
data class BpmData(
    val isConnected: Boolean = false,
    val pressure: Int = 0,
    val pulseStatus: Int = 0,
    val systolic: Int = 0,
    val diastolic: Int = 0,
    val pulse: Int = 0,
    val irregularPulse: Int = 0,
    val lastUpdatedTime: String = ""
)
```



------

### **Scale（体重秤）**

> **重点**：体重秤的数据通过广播方式传递，所以无连接/断开，开启扫描即可

⚠️注意：扫描设备前，请先扫描二维码配对设备

**Scale 单位**：weight 返回值单位为 **千克（kg）**（UI 显示时请注明单位）。

**代码示例**

```kotlin
import hk.ired.com.smartlearningsuite.BleViewModel

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)

// ⚠️ 无需连接，只需扫描广播包即可获取体重
Button(onClick = { vm.scanScale(mac) }) { Text("扫描并读取") }
Button(onClick = { vm.stopScanScale {} }) { Text("停止扫描") }

Text(text = "weight: ${vm.scaleData.weight}") // 体重(KG)
Text(text = "isStable: ${vm.scaleData.isStable}") // 是否为稳定体重,即最终结果
Text(text = "lastUpdatedTime: ${vm.scaleData.lastUpdatedTime}") // 数据更新最后时间
```

**数据模型**

```kotlin
data class ScaleData(
    val weight: Double = 0.0, val isStable: Boolean = false, val lastUpdatedTime: String = ""
)
```

------

### **Heart Rate Monitor（心率带）**

⚠️注意：连接设备前，请先扫描二维码配对设备

**代码示例**

```kotlin
import hk.ired.com.smartlearningsuite.BleViewModel

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)

Button(onClick = { vm.connectHeartRateMonitor(mac) }) { Text("连接") }
Button(onClick = { vm.disconnectHeartRateMonitor(mac) }) { Text("断开连接") }

// 显示实时心率与电量
Text("连接: ${vm.heartRateMonitorData.isConnected}") // 是否已连接
Text("心率: ${vm.heartRateMonitorData.heartRate} bpm") // 当前心率
Text("电量: ${vm.heartRateMonitorData.batteryLevel}") // 电量（0-100%）
Text("lastUpdatedTime: ${vm.heartRateMonitorData.lastUpdatedTime}") // 数据更新最后时间
```

**数据模型**

```kotlin
data class HeartRateMonitorData(
    val isConnected: Boolean = false,
    val lastUpdatedTime: String = "",
    val heartRate: Int = 0,
    val batteryPercent: Int = 0
)
```

------


### **Rope（跳绳）**

⚠️注意：连接设备前，请先扫描二维码配对设备

**代码示例**

```kotlin
import hk.ired.com.smartlearningsuite.BleViewModel

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)

// 连接 / 断开
Button(onClick = { vm.connectRope(mac) }) { Text("连接") }
Button(onClick = { vm.disconnectRope(mac) }) { Text("断开连接") }

// 设置模式（自由 / 时间 / 计数）
Button(onClick = { vm.ropeSetMode(0, 0) }) { Text("自由模式") }
Button(onClick = { vm.ropeSetMode(1, 60) }) { Text("时间模式(60秒)") }
Button(onClick = { vm.ropeSetMode(2, 100) }) { Text("计数模式(100次)") }

// 显示跳绳数据
Text(text = "连接状态: ${vm.ropeData.isConnected}") // 连接状态
Text(text = "count: ${vm.ropeData.count}") // 跳绳次数
Text(text = "mode: ${vm.ropeData.mode}") // 0: 自由模式, 1: 时间模式, 2: 计数模式
Text(text = "status: ${vm.ropeData.status}") // 0: 未跳绳, 1: 跳绳中, 2: 暂停, 3: 结束
Text(text = "setting: ${vm.ropeData.setting}") // 目标设置 (Time/Count)。其单位由 mode 决定 (1=秒, 2=次数)。
Text(text = "batteryLevel: ${vm.ropeData.batteryLevel}") // 电池等级: 0: 0-5%, 1: 6-25%, 2: 26-50%, 3: 51-75%, 4: 76-100%
Text(text = "time (sec): ${vm.ropeData.time}") // 时间（秒）
Text(text = "lastUpdatedTime: ${vm.ropeData.lastUpdatedTime}") // 数据更新最后时间
```

**数据模型**

```kotlin
data class RopeData(
    val isConnected: Boolean = false,
    val lastUpdatedTime: String = "",
    val count: Int = 0,
    val mode: Int = 0,
    val status: Int = 0,
    val setting: Int = 0,
    val batteryLevel: Int = 0,
    val time: Int = 0
)
```

------