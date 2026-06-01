# 第2章 Cordova本地资源调用

> [!SUMMARY] Table of Contents
>
> - [[第2章-Cordova#第2章 Cordova本地资源调用|第2章 Cordova本地资源调用]]
>   - [[第2章-Cordova#§1 Cordova 基本原理|§1 Cordova 基本原理]]
>     - [[第2章-Cordova#为什么需要 Cordova|为什么需要 Cordova]]
>     - [[第2章-Cordova#Cordova 是什么|Cordova 是什么]]
>     - [[第2章-Cordova#Cordova 调用三角形|Cordova 调用三角形]]
>     - [[第2章-Cordova#工程目录结构|工程目录结构]]
>   - [[第2章-Cordova#§2 Cordova 插件开发|§2 Cordova 插件开发]]
>     - [[第2章-Cordova#创建插件|创建插件]]
>     - [[第2章-Cordova#插件目录结构|插件目录结构]]
>     - [[第2章-Cordova#pluginxml 配置|pluginxml 配置]]
>   - [[第2章-Cordova#§3 Cordova 插件应用|§3 Cordova 插件应用]]
>     - [[第2章-Cordova#安装插件|安装插件]]
>     - [[第2章-Cordova#JS 端（TestDialogjs）|JS 端]]
>     - [[第2章-Cordova#原生端（TestDialogjava）|原生端]]
>     - [[第2章-Cordova#页面调用|页面调用]]
>     - [[第2章-Cordova#执行流程|执行流程]]
>   - [[第2章-Cordova#§4 GPS 插件（cordova-plugin-geolocation）|§4 GPS 插件]]
>     - [[第2章-Cordova#使用步骤|使用步骤]]
>     - [[第2章-Cordova#模拟定位|模拟定位]]
>   - [[第2章-Cordova#§5 百度地图移动应用服务|§5 百度地图移动应用服务]]
>     - [[第2章-Cordova#申请 AK|申请 AK]]
>     - [[第2章-Cordova#数字签名基础|数字签名基础]]
>     - [[第2章-Cordova#数字证书管理|数字证书管理]]
>   - [[第2章-Cordova#§6 百度定位服务插件|§6 百度定位服务插件]]
>     - [[第2章-Cordova#使用步骤-1|使用步骤]]
>     - [[第2章-Cordova#调试方法|调试方法]]
>   - [[第2章-Cordova#总结|总结]]
>   - [[第2章-Cordova#课后作业|课后作业]]

---

## §1 Cordova 基本原理

### 为什么需要 Cordova

回顾混合编程中直接通过 WebView 实现 H5 与原生交互的方式存在以下问题：

- 接口定义、实现、URL 解析、弹窗解析均需手动处理，**比较麻烦**
- 不同平台的方法存在差异
- 不是所有人都会原生编程
- **缺少标准**，不具有多人合作的可能

Cordova 提供了一个**框架和标准**，使开发者完全不需要考虑调用关系细节，同时基于该标准开发的插件可被他人共享使用。

### Cordova 是什么

Cordova 使用 HTML、CSS 和 JS 构建移动应用，它是一个**容器**，用于将 Web 应用程序与本机移动功能连接，生成完整的 APP 应用。

- 使用 JS 编写，不需要学习平台特定语言
- Cordova 也可作为**插件**（plugin）在其他 H5 应用中引入并和本地资源交互
- Cordova 插件由 JS + 原生代码组成，打包时整体编译
- 需要 Android SDK 或 iOS Xcode

常用原生功能示例：GPS 定位、电池余量、系统信息、摄像头、联系人列表、本地文件、网络信息、状态栏等。

创建 Cordova 工程：
```bash
cordova create CordovaApp com.cordova.app CordovaAppTest
```

### Cordova 调用三角形

```
TS/JS → 插件 JS/TS → 插件 Native → TS/JS
```

插件管理通过 NPM：`npm publish /path/to/your/plugin`，`cordova plugin add cordova-plugin-device`。

学习内容：
- Cordova 开发移动应用
- Cordova 插件开发（供别人使用）
- 使用现有的 Cordova 插件

### 工程目录结构

- `platforms\android\app\src\main\assets\www`——添加平台时 `www` 目录复制到这里，同时生成 `cordova.js`、`cordova_plugins.js`
- `config.xml` 中 `<content src="index.html" />` 指定入口 HTML 文件
- `plugins` 目录下所有在 `package.json` 的 `cordova` 域中引用的插件一同加载到平台中

---

## §2 Cordova 插件开发

### 创建插件

```bash
npm install -g plugman                          # 安装 plugman 工具
npm init                                        # 先创建 npm 包
plugman create --name TestDialog --plugin_id com.testdialog --plugin_version 1.0.0
cd TestDialog
plugman platform add --platform_name android
plugman platform add --platform_name ios
```

### 插件目录结构

| 目录/文件 | 说明 |
|-----------|------|
| `src/` | 各平台源码（Android、iOS 等） |
| `www/` | JS 调用 native 代码的接口文件 |
| `plugin.xml` | 插件配置文件 |
| `package.json` | npm 包信息 |

### plugin.xml 配置

`plugin.xml` 遵循 Apache Cordova 规范：

- **clobbers**：前端通过它调用 `www/TestDialog.js` 的公开方法（如 `cordova.plugins.TestDialog`）
- **feature**：定义服务名（name）
- **param**：定义各平台的底层实现类
  - `ios-package`：iOS 平台下的底层实现类名
  - `android-package`：Android 平台下的 `包名.类`
- `<source-file>`：指定源码文件位置和目标目录
- `<config-file>`：向平台配置中插入内容

---

## §3 Cordova 插件应用

### 安装插件

```bash
cordova plugin add ../TestDialog
```

安装时自动执行：复制到 `node_modules`、`plugins` 目录，将 JS 部分复制到平台 `www` 目录，Java 源码复制到平台 `java` 目录。

`cordova plugin list` 列出已安装的插件。

### JS 端（TestDialog.js）

```javascript
var exec = require('cordova/exec');
var TestDialog = {
  coolMethod: function(arg0, success, error) {
    exec(success, error, "TestDialog", "coolMethod", [arg0]);
  },
  addSum: function(args, success, error) {
    exec(success, error, "TestDialog", "sum", args);
  }
};
module.exports = TestDialog;
```

**exec 参数说明**：

| 参数位置 | 含义 | 示例 |
|---------|------|------|
| 参数1 | 成功回调 | `success` |
| 参数2 | 失败回调 | `error` |
| 参数3 | 服务名（与 plugin.xml 中 `js-module name` 一致） | `"TestDialog"` |
| 参数4 | Action 动作名（原生根据此字符串调度不同方法） | `"coolMethod"` |
| 参数5 | 参数，必须是数组，可以 JSON 数据 | `[arg0]` 或 `[{key:value}]` |

调用方式：`cordova.plugins.TestDialog.coolMethod(arg0, success, error)`，其中 `cordova.plugins` 来源于 `cordova.js`。

### 原生端（TestDialog.java）

```java
public class TestDialog extends CordovaPlugin {
  @Override
  public boolean execute(String action, JSONArray args,
      CallbackContext callbackContext) throws JSONException {
    if (action.equals("coolMethod")) {
      String message = args.getString(0);
      this.coolMethod(message, callbackContext);
      return true;
    } else if (action.equals("sum"))
      return this.sum(args, callbackContext);
    return false;
  }
  private void coolMethod(String message, CallbackContext callbackContext) {
    if (message != null && message.length() > 0)
      callbackContext.success(message);
    else
      callbackContext.error("Expected one non-empty string argument.");
  }
  private boolean sum(JSONArray args, CallbackContext callbackContext) {
    int x = args.getInt(0); int y = args.getInt(1);
    callbackContext.success(x + y);
    return true;
  }
}
```

`execute` 方法接收 3 个参数：`String action`（动作名）、`JSONArray args`（JS 传入的参数数组）、`CallbackContext callbackContext`（回调）。

JS 入参如果是 `["value1", 10, "value3"]`，则原生端通过 `args.getString(0)`、`args.getInt(1)` 获取；如果是 JSON 数组，则通过 `JSONArray` 和 `JSONObject` 解析。

### 页面调用

```html
<script type="text/javascript" src="cordova.js"></script>
<script type="text/javascript" src="js/index.js"></script>
<div class="app">
  <h1>Apache Cordova</h1>
  <button onclick="coolMethod('cool')">Click Me</button>
  <button onclick="sum(2,3)">Call sum</button>
</div>
```

index.js 中定义：
```javascript
function success(msg) { alert(msg); }
function error(msg) { alert(msg); }
function coolMethod(x) {
  cordova.plugins.TestDialog.coolMethod(x, success, error);
}
function sum(x, y) {
  cordova.plugins.TestDialog.addSum([x, y], success, error);
}
```

运行：`cordova run android`

### 执行流程

1. JS 方法 → 2. `cordova.exec()`（第四个参数对应原生方法名） → 3. 原生 `execute()` → 4. 原生方法调用 → 5. 回调 `success` 或 `error`

Cordova 提供的核心价值：**规范 + 跨平台**（Android、iOS、Windows 等）。

---

## §4 GPS 插件（cordova-plugin-geolocation）

### 使用步骤

```bash
cordova create geolocation                    # 创建工程
cordova plugin add cordova-plugin-geolocation  # 添加插件
cordova platform add android
cordova platform add browser
```

**index.html** 核心代码：
```javascript
document.addEventListener("deviceready", onDeviceReady, false);
function onDeviceReady() {
  navigator.geolocation.getCurrentPosition(onSuccess, onError, {
    enableHighAccuracy: true,
    timeout: 5000,
    maximumAge: 0
  });
}
```

**index.js 回调**：
```javascript
var onSuccess = function(position) {
  document.getElementById('loc').innerHTML =
    'Latitude: ' + position.coords.latitude + '\n' +
    'Longitude: ' + position.coords.longitude + '\n' +
    'Altitude: ' + position.coords.altitude + '\n' +
    'Accuracy: ' + position.coords.accuracy + '\n' +
    'Speed: ' + position.coords.speed + '\n' +
    'Timestamp: ' + position.timestamp;
};
function onError(error) { console.log(JSON.stringify(error)); }
```

### 模拟定位

```bash
adb emu geo fix 116.397 39.908  # 模拟定位到北京天安门
```

遇到黑屏或无法连接 adb，可先执行 `adb kill-server && adb start-server`。

---

## §5 百度地图移动应用服务

### 申请 AK

百度地图 API 申请地址：`https://lbsyun.baidu.com/apiconsole/key#/home`

注意需要申请不同类型的 AK（移动端申请 Android SDK / iOS SDK），申请时需提供**数字指纹**（安全码）。

### 数字签名基础

**数字签名技术**基于公钥密码体制，用于验证信息的完整性和真实性（如 APK、IPA 的签名）。

签名作用：
- **应用身份识别**：每个安装文件有唯一签名（身份证）
- **保证应用完整性**：文件被修改则签名失效
- **应用升级校验**：检查新旧版本签名是否一致
- **权限授予**：某些系统级权限（如百度地图服务）基于签名

签名机制：密钥库文件（keystore）含私钥和公钥。开发者用**私钥**签名，生成 `CERT.SF`（签名文件）；安装时设备用**公钥**验证 `CERT.RSA`。

签名两步：1）从内容算摘要（哈希算法），2）从摘要明文到摘要密文（私钥 + 加密算法）。

摘要算法：将任意长度文本得到固定长度的消息摘要（digest）。常用 MD5、SHA1、SHA256/512。难以从结果反推源，但易于从摘要验证源合法性。

### 数字证书管理

生成发布版数字证书：
```bash
keytool -genkeypair -alias helloalias -keyalg RSA -keysize 2048 \
  -validity 36500 -keystore hello.keystore \
  -storepass android -keypass android \
  -dname "CN=Zhang San, OU=Dev, O=MyCompany, L=Shanghai, ST=Shanghai, C=CN"
```

| 字段 | 含义 |
|------|------|
| CN | Common Name（通用名称） |
| OU | Organizational Unit（组织单位） |
| O | Organization（组织/公司） |
| L | Locality（城市/地区） |
| ST | State or Province（省/州） |
| C | Country（国家代码，两位字母） |

签名 APK：
```bash
apksigner sign --ks hello.keystore --out app-release-signed.apk app-release-unsigned.apk
```

查看证书指纹：
```bash
keytool -list -v -keystore debug.keystore -alias androiddebugkey -storepass android -keypass android
```

> `debug.keystore` 默认在 `C:\Users\<用户名>\.android` 目录下，调试版密码为 `android`。

百度地图 JS 服务：`https://lbsyun.baidu.com/jsdemo.htm`，申请浏览器端 AK，将源码中密钥替换后可用。

---

## §6 百度定位服务插件（hewz.plugins.baidu-location）

该插件允许在 APP 中获取硬件定位信息。

### 使用步骤

```bash
cordova create baidu_hewz com.test.map baidumaptest
cordova plugin add hewz.plugins.baidu-location \
  --variable API_KEY="移动端Android/iOS SDK类型的AK"
# 修改 index.html 和 index.js
cordova platform add android@14
cordova run android  # 必须在真机运行，虚拟机得不到定位结果
```

> 注意手动开启 APP 的定位权限。

**index.html** 调用示例：
```javascript
document.addEventListener("deviceready", onDeviceReady, false);
function onDeviceReady() {
  baidu_location.getCurrentPosition(onSuccess, onError, {
    enableHighAccuracy: true, timeout: 5000, maximumAge: 0
  });
}
```

**index.js** 回调：
```javascript
var onSuccess = function(position) {
  var output = JSON.stringify(position);
  document.getElementById('loc').innerHTML = output;
};
function onError(error) { console.log(JSON.stringify(error)); }
```

### 调试方法

通过 Chrome 或 Edge 远程调试：输入 `chrome://inspect` 或 `edge://inspect` 列出连接的设备。

若出现定位错误，应检查并修改应用的定位权限。

---

## 总结

使用 Cordova Plugin 获取移动设备空间定位功能的一般流程：
1. 安装插件 → 2. 配置插件 → 3. 使用插件

地图作为移动应用底图，可结合高德、百度、腾讯等地图服务 + Cordova Plugin 进行地图应用开发（如在地图上显示当前位置）。

---

## 课后作业

开发并实践 Cordova 插件，用 H5 显示界面元素，用原生代码实现数值计算，完成移动计算器（支持 + - × ÷，含参数1、参数2输入框和结果展示）。