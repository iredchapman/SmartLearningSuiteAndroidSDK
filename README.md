---
name: smart-learning-suite-android-sdk
description: 提供針對智慧健康與運動設備（體溫計、血氧儀、血壓計、體重計、心率監測儀及跳繩）的 BLE 整合方案。包含完整的 Android 權限配置，以及基於 Jetpack Compose 與 ViewModel 的 SDK 程式碼範例。
---

## **1. 初始化項目**

### **1.1 配置 AndroidManifest**

```xml
<!-- Android 12+ runtime permissions -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />

<!-- 向後兼容舊版本 -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />

<!-- 如果你仍然聲明位置權限，就保留下面（可能會觸發系統要求用戶打開位置服務） -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- 掃碼所需相機權限 -->
<uses-permission android:name="android.permission.CAMERA" />
```

### 1.2 導入依賴與環境配置

**Gradle 配置**

1. 在 **settings.gradle.kts** 中配置 JitPack 倉庫：

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

3. 在 **app 級 build.gradle.kts** 中添加 SDK 、 Compose 依賴 和 緩存動態時間：

```kotlin
dependencies {
    // SDK
    implementation("com.github.iredchapman:SmartLearningSuiteAndroidSDK:+")

    // Compose & ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.4")
    implementation("androidx.activity:activity-compose:1.9.3")
  
    // ZXing 掃碼庫依賴
    implementation("com.journeyapps:zxing-android-embedded:4.3.0")
}
```

4. 添加 Compose 編譯器插件 (Kotlin 2.0+)

   ⚠️ **重要提示：** 從 Kotlin 2.0 開始，如果啟用了 Compose，則必須使用 Compose Compiler Gradle 插件。

   示例 (Project 級 `build.gradle.kts`)：

   ```kotlin
   plugins {
       // ... 其他插件
       id("org.jetbrains.kotlin.plugin.compose") version "X.Y.Z" // 替換為兼容的編譯器版本
   }
   ```


5. 修改動態版本的緩存時間（全局生效）

   ```kotlin
   configurations.all {
       resolutionStrategy.cacheDynamicVersionsFor(0, "seconds")
   }
   ```

6. 同步 Gradle

```
Gradle Sync
```

### **1.3 初始化 ViewModel**

**注意：** AI 在生成代碼時，請假設 `BleViewModel` 內部已封裝好 SDK 調用。

文檔中的代碼已經是全部真理，不要去分析 AAR，直接使用文檔中的代碼結構。


```kotlin
import android.Manifest
import android.content.pm.PackageManager
import android.os.Build
import android.widget.Toast
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.remember
import androidx.compose.ui.platform.LocalContext
import androidx.core.content.ContextCompat

val context = LocalContext.current

// 1. 定義需要申請的權限列表 (根據 Android 版本區分)
val permissionsToRequest = remember {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        arrayOf(
            Manifest.permission.BLUETOOTH_SCAN,
            Manifest.permission.BLUETOOTH_CONNECT,
          	Manifest.permission.CAMERA
        )
    } else {
        arrayOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION,
            Manifest.permission.CAMERA
        )
    }
}

// 2. 創建權限請求啟動器 (Launcher)
val launcher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    val allGranted = permissions.entries.all { it.value }
    if (allGranted) {
        Toast.makeText(context, "藍牙權限已授予", Toast.LENGTH_SHORT).show()
    } else {
        Toast.makeText(context, "缺少藍牙權限，設備連接功能將無法使用", Toast.LENGTH_LONG).show()
    }
}

// 3. 在界面加載時自動觸發權限檢查
LaunchedEffect(Unit) {
    val allGranted = permissionsToRequest.all {
        ContextCompat.checkSelfPermission(context, it) == PackageManager.PERMISSION_GRANTED
    }
    if (!allGranted) {
        launcher.launch(permissionsToRequest)
    }
}
```

## **2. 設備調用說明**

### **Thermometer**

⚠️注意：連接設備前，請先掃描二維碼配對設備

**代碼示例**

```kotlin
import android.app.Application
import android.widget.Toast
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.compose.viewModel
import hk.ired.com.smartlearningsuite.BleViewModel
import com.journeyapps.barcodescanner.ScanContract
import com.journeyapps.barcodescanner.ScanOptions

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)
var mac by remember { mutableStateOf("") }

val barcodeLauncher = rememberLauncherForActivityResult(
    contract = ScanContract()
) { result ->
    result.contents?.let { scannedMac ->
        val macRegex = Regex("^([0-9A-Fa-f]{2}[:-])*([0-9A-Fa-f]{2})$")
        if (macRegex.matches(scannedMac) && scannedMac.length == 17) {
            mac = scannedMac.uppercase()
        } else {
            Toast.makeText(context, "掃描結果不是有效的 MAC 地址", Toast.LENGTH_SHORT).show()
        }
    }
}

Button(onClick = {
    val options = ScanOptions().apply {
        setDesiredBarcodeFormats(ScanOptions.QR_CODE)
        setPrompt("請掃描設備二維碼")
        setBeepEnabled(false)
        setOrientationLocked(false)
    }
    barcodeLauncher.launch(options)
}) { Text("掃描設備二維碼") }

Button(onClick = { vm.connectThermometer(mac) }) { Text("連接") }
Button(onClick = { vm.disconnectThermometer(mac) }) { Text("斷開") }

Text(text = "連接狀態: ${vm.thermometerData.isConnected}")
Text(text = "溫度: ${vm.thermometerData.temperature}") // 攝氏度
when (vm.thermometerData.mode) {
    1 -> Text(text = "模式: 成人額溫")
    2 -> Text(text = "模式: 兒童額溫")
    3 -> Text(text = "模式: 耳道")
    4 -> Text(text = "模式: 物體")
    else -> Text(text = "未知模式")
}
Text(text = "電池電量: ${vm.thermometerData.batteryPercent}%")
Text(text = "最後更新時間: ${vm.thermometerData.lastUpdatedTime}")
```

**數據模型**

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



### **Oximeter（血氧儀）**

⚠️注意：連接設備前，請先掃描二維碼配對設備

**代碼示例**

```kotlin
import android.app.Application
import android.widget.Toast
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.compose.viewModel
import hk.ired.com.smartlearningsuite.BleViewModel
import com.journeyapps.barcodescanner.ScanContract
import com.journeyapps.barcodescanner.ScanOptions

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)
var mac by remember { mutableStateOf("") }

val barcodeLauncher = rememberLauncherForActivityResult(
    contract = ScanContract()
) { result ->
    result.contents?.let { scannedMac ->
        val macRegex = Regex("^([0-9A-Fa-f]{2}[:-])*([0-9A-Fa-f]{2})$")
        if (macRegex.matches(scannedMac) && scannedMac.length == 17) {
            mac = scannedMac.uppercase()
        } else {
            Toast.makeText(context, "掃描結果不是有效的 MAC 地址", Toast.LENGTH_SHORT).show()
        }
    }
}

Button(onClick = {
    val options = ScanOptions().apply {
        setDesiredBarcodeFormats(ScanOptions.QR_CODE)
        setPrompt("請掃描設備二維碼")
        setBeepEnabled(false)
        setOrientationLocked(false)
    }
    barcodeLauncher.launch(options)
}) { Text("掃描設備二維碼") }

Button(onClick = { vm.connectOximeter(mac) }) { Text("連接") }
Button(onClick = { vm.disconnectOximeter(mac) }) { Text("斷開") }

Text(text = "連接狀態: ${vm.oximeterData.isConnected}")
Text(text = "脈搏率: ${vm.oximeterData.pulse}")
Text(text = "血氧飽和度: ${vm.oximeterData.spo2}")
Text(text = "灌注指數: ${vm.oximeterData.pi}")
Text(text = "電池電量: ${vm.oximeterData.battery}%")
Text(text = "最後更新時間: ${vm.oximeterData.lastUpdatedTime}")
```

**數據模型**

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

**建議測量時長**：**10–15 秒** —— 血氧測量通常需要一定時間（10–15秒）來穩定讀取準確值，UI/用戶提示應告知被測者靜止不動。


------



### **Blood Pressure Monitor（血壓計）**

⚠️注意：連接設備前，請先掃描二維碼配對設備

**代碼示例**

```kotlin
import android.app.Application
import android.widget.Toast
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.compose.viewModel
import hk.ired.com.smartlearningsuite.BleViewModel
import com.journeyapps.barcodescanner.ScanContract
import com.journeyapps.barcodescanner.ScanOptions

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)
var mac by remember { mutableStateOf("") }

val barcodeLauncher = rememberLauncherForActivityResult(
    contract = ScanContract()
) { result ->
    result.contents?.let { scannedMac ->
        val macRegex = Regex("^([0-9A-Fa-f]{2}[:-])*([0-9A-Fa-f]{2})$")
        if (macRegex.matches(scannedMac) && scannedMac.length == 17) {
            mac = scannedMac.uppercase()
        } else {
            Toast.makeText(context, "掃描結果不是有效的 MAC 地址", Toast.LENGTH_SHORT).show()
        }
    }
}

Button(onClick = {
    val options = ScanOptions().apply {
        setDesiredBarcodeFormats(ScanOptions.QR_CODE)
        setPrompt("請掃描設備二維碼")
        setBeepEnabled(false)
        setOrientationLocked(false)
    }
    barcodeLauncher.launch(options)
}) { Text("掃描設備二維碼") }

Button(onClick = { vm.connectBpm(mac) }) { Text("連接") }
Button(onClick = { vm.disconnectBpm(mac) }) { Text("斷開") }

Text(text = "連接狀態: ${vm.bpmData.isConnected}")
Text(text = "血壓值: ${vm.bpmData.pressure}")
Text(text = "收縮壓: ${vm.bpmData.systolic}")
Text(text = "舒張壓: ${vm.bpmData.diastolic}")
Text(text = "脈搏率: ${vm.bpmData.pulse}")
Text(text = "脈搏異常: ${vm.bpmData.irregularPulse}")
Text(text = "脈搏狀態: ${vm.bpmData.pulseStatus}")
Text(text = "最後更新時間: ${vm.bpmData.lastUpdatedTime}")
```

**數據模型**

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

### **Scale（體重秤）**

> **重點**：體重秤的數據通過廣播方式傳遞，所以無連接/斷開，開啟掃描即可

⚠️注意：掃描設備前，請先掃描二維碼配對設備

**Scale 單位**：weight 返回值單位為 **千克（kg）**（UI 顯示時請註明單位）。

**代碼示例**

```kotlin
import android.app.Application
import android.widget.Toast
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.compose.viewModel
import hk.ired.com.smartlearningsuite.BleViewModel
import com.journeyapps.barcodescanner.ScanContract
import com.journeyapps.barcodescanner.ScanOptions

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)
var mac by remember { mutableStateOf("") }

val barcodeLauncher = rememberLauncherForActivityResult(
    contract = ScanContract()
) { result ->
    result.contents?.let { scannedMac ->
        val macRegex = Regex("^([0-9A-Fa-f]{2}[:-])*([0-9A-Fa-f]{2})$")
        if (macRegex.matches(scannedMac) && scannedMac.length == 17) {
            mac = scannedMac.uppercase()
        } else {
            Toast.makeText(context, "掃描結果不是有效的 MAC 地址", Toast.LENGTH_SHORT).show()
        }
    }
}

Button(onClick = {
    val options = ScanOptions().apply {
        setDesiredBarcodeFormats(ScanOptions.QR_CODE)
        setPrompt("請掃描設備二維碼")
        setBeepEnabled(false)
        setOrientationLocked(false)
    }
    barcodeLauncher.launch(options)
}) { Text("掃描設備二維碼") }

Button(onClick = { vm.scanScale(mac) }) { Text("掃描並讀取") }
Button(onClick = { vm.stopScanScale {} }) { Text("停止掃描") }

Text(text = "體重: ${vm.scaleData.weight} kg")
Text(text = "穩定體重: ${vm.scaleData.isStable}") // 是否為最終結果
Text(text = "最後更新時間: ${vm.scaleData.lastUpdatedTime}")
```

**數據模型**

```kotlin
data class ScaleData(
    val weight: Double = 0.0,
    val isStable: Boolean = false,
    val lastUpdatedTime: String = ""
)
```

------

### **Heart Rate Monitor（心率帶）**

⚠️注意：連接設備前，請先掃描二維碼配對設備

**代碼示例**

```kotlin
import android.app.Application
import android.widget.Toast
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.compose.viewModel
import hk.ired.com.smartlearningsuite.BleViewModel
import com.journeyapps.barcodescanner.ScanContract
import com.journeyapps.barcodescanner.ScanOptions

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)
var mac by remember { mutableStateOf("") }

val barcodeLauncher = rememberLauncherForActivityResult(
    contract = ScanContract()
) { result ->
    result.contents?.let { scannedMac ->
        val macRegex = Regex("^([0-9A-Fa-f]{2}[:-])*([0-9A-Fa-f]{2})$")
        if (macRegex.matches(scannedMac) && scannedMac.length == 17) {
            mac = scannedMac.uppercase()
        } else {
            Toast.makeText(context, "掃描結果不是有效的 MAC 地址", Toast.LENGTH_SHORT).show()
        }
    }
}

Button(onClick = {
    val options = ScanOptions().apply {
        setDesiredBarcodeFormats(ScanOptions.QR_CODE)
        setPrompt("請掃描設備二維碼")
        setBeepEnabled(false)
        setOrientationLocked(false)
    }
    barcodeLauncher.launch(options)
}) { Text("掃描設備二維碼") }

Button(onClick = { vm.connectHeartRateMonitor(mac) }) { Text("連接") }
Button(onClick = { vm.disconnectHeartRateMonitor(mac) }) { Text("斷開連接") }

Text("連接: ${vm.heartRateMonitorData.isConnected}")
Text("心率: ${vm.heartRateMonitorData.heartRate} bpm")
Text("電量: ${vm.heartRateMonitorData.batteryPercent}")
Text("最後更新時間: ${vm.heartRateMonitorData.lastUpdatedTime}")
```

**數據模型**

```kotlin
data class HeartRateMonitorData(
    val isConnected: Boolean = false,
    val lastUpdatedTime: String = "",
    val heartRate: Int = 0,
    val batteryPercent: Int = 0
)
```

------


### **Rope（跳繩）**

⚠️注意：連接設備前，請先掃描二維碼配對設備

**代碼示例**

```kotlin
import android.app.Application
import android.widget.Toast
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.compose.viewModel
import hk.ired.com.smartlearningsuite.BleViewModel
import com.journeyapps.barcodescanner.ScanContract
import com.journeyapps.barcodescanner.ScanOptions

val context = LocalContext.current
val vm: BleViewModel = viewModel(
    factory = ViewModelProvider.AndroidViewModelFactory.getInstance(context.applicationContext as Application)
)
var mac by remember { mutableStateOf("") }

val barcodeLauncher = rememberLauncherForActivityResult(
    contract = ScanContract()
) { result ->
    result.contents?.let { scannedMac ->
        val macRegex = Regex("^([0-9A-Fa-f]{2}[:-])*([0-9A-Fa-f]{2})$")
        if (macRegex.matches(scannedMac) && scannedMac.length == 17) {
            mac = scannedMac.uppercase()
        } else {
            Toast.makeText(context, "掃描結果不是有效的 MAC 地址", Toast.LENGTH_SHORT).show()
        }
    }
}

Button(onClick = {
    val options = ScanOptions().apply {
        setDesiredBarcodeFormats(ScanOptions.QR_CODE)
        setPrompt("請掃描設備二維碼")
        setBeepEnabled(false)
        setOrientationLocked(false)
    }
    barcodeLauncher.launch(options)
}) { Text("掃描設備二維碼") }

// 連接 / 斷開
Button(onClick = { vm.connectRope(mac) }) { Text("連接") }
Button(onClick = { vm.disconnectRope(mac) }) { Text("斷開連接") }

// 設置模式（自由 / 時間 / 計數）
Button(onClick = { vm.ropeSetMode(0, 0) }) { Text("自由模式") }
Button(onClick = { vm.ropeSetMode(1, 60) }) { Text("時間模式(60秒)") }
Button(onClick = { vm.ropeSetMode(2, 100) }) { Text("計數模式(100次)") }

// 顯示跳繩數據
Text(text = "連接狀態: ${vm.ropeData.isConnected}") // 連接狀態
Text(text = "跳繩次數: ${vm.ropeData.count}")
Text(text = "模式: ${vm.ropeData.mode}") // 0: 自由模式, 1: 時間模式, 2: 計數模式
Text(text = "狀態: ${vm.ropeData.status}") // 0: 未跳繩, 1: 跳繩中, 2: 暫停, 3: 結束
Text(text = "目標設置: ${vm.ropeData.setting}") // 其單位由模式決定 (1=秒, 2=次數)
Text(text = "電池等級: ${vm.ropeData.batteryLevel}") // 0: 0-5%, 1: 6-25%, 2: 26-50%, 3: 51-75%, 4: 76-100%
Text(text = "時間（秒）: ${vm.ropeData.time}")
Text(text = "最後更新時間: ${vm.ropeData.lastUpdatedTime}")
```

**數據模型**

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
