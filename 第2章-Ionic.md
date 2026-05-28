# 第2章 Ionic混合编程技术

> [!SUMMARY] Table of Contents
>
> - [[第2章-Ionic#§1 Ionic 概述|§1 Ionic 概述]]
>   - [[第2章-Ionic#Ionic 是什么|Ionic 是什么]]
>   - [[第2章-Ionic#安装与环境|安装与环境]]
>   - [[第2章-Ionic#工程创建与结构|工程创建与结构]]
> - [[第2章-Ionic#§2 Ionic 常用基础|§2 Ionic 常用基础]]
>   - [[第2章-Ionic#UI 组件|UI 组件]]
>   - [[第2章-Ionic#创建组件与页面|创建组件与页面]]
>   - [[第2章-Ionic#数据绑定与事件|数据绑定与事件]]
>   - [[第2章-Ionic#属性与结构指令|属性与结构指令]]
> - [[第2章-Ionic#§3 Ionic + Angular 进阶|§3 Ionic + Angular 进阶]]
>   - [[第2章-Ionic#APP 项目配置|APP 项目配置]]
>   - [[第2章-Ionic#路由和导航|路由和导航]]
>   - [[第2章-Ionic#页面生命周期事件|页面生命周期事件]]
> - [[第2章-Ionic#§4 Ionic + Cordova 实现 GPS 数据采集|§4 Ionic + Cordova 实现 GPS 数据采集]]
> - [[第2章-Ionic#§5 Ionic + Cordova 实现相机照片采集|§5 Ionic + Cordova 实现相机照片采集]]
> - [[第2章-Ionic#§6 Ionic + Cordova 时空数据采集|§6 Ionic + Cordova 时空数据采集]]
>   - [[第2章-Ionic#百度地图集成|百度地图集成]]
>   - [[第2章-Ionic#定位服务集成|定位服务集成]]
> - [[第2章-Ionic#§7 调试与测试|§7 调试与测试]]
> - [[第2章-Ionic#课后自学参考|课后自学参考]]

---

## §1 Ionic 概述

### Ionic 是什么

Ionic 是一个**开源的移动端混合开发框架**，基于 Angular 和 Cordova，使用 Web 技术（HTML/CSS/TS）开发跨平台移动应用（Android、iOS 等）。

Ionic 提供了丰富的**UI 组件库**（按钮、列表、卡片、导航栏等），可快速构建接近原生体验的移动端界面。

### 安装与环境

```bash
npm install -g @ionic/cli          # 安装 Ionic CLI
npm install -g cordova             # Cordova 是底层打包工具
ionic start myApp blank            # 创建空白工程（模板可选 tabs/sidemenu/blank）
```

### 工程创建与结构

```bash
ionic start myApp blank --type=angular    # 选择 Angular 项目类型
cd myApp
ionic serve                               # 在浏览器中预览
```

工程目录说明：

| 目录/文件 | 说明 |
|-----------|------|
| `src/app/` | 应用主模块 |
| `src/app/app.module.ts` | 应用的根模块定义（引入组件和插件） |
| `src/app/app-routing.module.ts` | 路由配置 |
| `src/app/pages/` | 页面组件存放位置 |
| `src/index.html` | 主入口 HTML |
| `angular.json` | Angular 项目配置 |
| `config.xml` | Cordova 配置文件 |

---

## §2 Ionic 常用基础

### UI 组件

Ionic 提供了丰富的 UI 组件，常用的有：

| 组件 | 标签 | 示例 |
|------|------|------|
| 标题栏 | `<ion-header><ion-toolbar><ion-title>` | |
| 按钮 | `<ion-button>` | `<ion-button expand="block" (click)="fn()">按钮</ion-button>` |
| 图标 | `<ion-icon>` | `<ion-icon name="add"></ion-icon>` |
| 输入框 | `<ion-input>` | `<ion-input [(ngModel)]="name" placeholder="请输入">` |
| 列表 | `<ion-list><ion-item>` | |
| 内容区 | `<ion-content>` | `<ion-content class="ion-padding">` |
| 卡片 | `<ion-card>` | 含 `<ion-card-header>` + `<ion-card-content>` |

### 创建组件与页面

```bash
ionic generate page PageName        # 新建页面（自动添加路由）
ionic generate component CompName   # 新建组件
ionic generate provider ServiceName # 新建服务（provider）
```

### 数据绑定与事件

```html
<!-- 双向绑定 -->
<ion-input [(ngModel)]="username"></ion-input>

<!-- 事件绑定 -->
<ion-button (click)="handleClick()">点击</ion-button>

<!-- 插值表达式 -->
<p>{{ message }}</p>
```

### 属性与结构指令

- `*ngIf`：条件渲染，`<div *ngIf="isVisible">显示内容</div>`
- `*ngFor`：循环渲染，`<ion-item *ngFor="let item of list">{{ item.name }}</ion-item>`
- `[ngStyle]`：动态样式，`[ngStyle]="{'color': isActive ? 'red' : 'blue'}"`
- `[ngClass]`：动态类名，`[ngClass]="{'active': isActive}"`

---

## §3 Ionic + Angular 进阶

### APP 项目配置

`src/app/app.module.ts` 是应用的根模块，需要在此注册：
- 页面组件
- 插件 Provider（如 Geofence、Diagnostic、Toast 等）
- 路由模块

### 路由和导航

`src/app/app-routing.module.ts` 定义路由映射：

页面导航方式：
- 通过 `[routerLink]` 属性
- 通过 `Router` 服务的 `navigate()` 方法编程式导航
- 通过 `navPush` 指令进行导航栈操作（老版 Ionic）

### 页面生命周期事件

Angular 页面常用生命周期钩子：

| 钩子 | 触发时机 |
|------|---------|
| `ngOnInit()` | 组件初始化时 |
| `ngAfterViewInit()` | 视图初始化完成后 |
| `ionViewDidEnter()` | Ionic 页面完全进入后 |
| `ionViewWillLeave()` | Ionic 页面即将离开时 |

---

## §4 Ionic + Cordova 实现 GPS 数据采集

1. 安装 GPS 插件：

```bash
ionic cordova plugin add cordova-plugin-geolocation --variable GPS_REQUIRED="false"
npm install @ionic-native/geolocation
```

2. 在 `app.module.ts` 中注册：

```typescript
import { Geolocation } from '@ionic-native/geolocation/ngx';
// providers 中添加 Geolocation
```

3. 在页面中使用：

```typescript
import { Geolocation } from '@ionic-native/geolocation/ngx';

constructor(private geolocation: Geolocation) {}

getCurrentCoordinates() {
  this.geolocation.getCurrentPosition().then((resp) => {
    this.latitude = resp.coords.latitude;
    this.longitude = resp.coords.longitude;
  }).catch((error) => {
    console.log('Error getting location', error);
  });
}
```

4. 页面模板示例（tab3.page.html）：

```html
<ion-header>
  <ion-toolbar>
    <ion-title>定位测试</ion-title>
  </ion-toolbar>
</ion-header>
<ion-content class="ion-padding">
  <ion-button expand="block" (click)="getLocation()">获取当前位置</ion-button>
  <p>状态：{{ msg }}</p>
  <p>纬度：{{ lat }}</p>
  <p>经度：{{ lng }}</p>
</ion-content>
```

---

## §5 Ionic + Cordova 实现相机照片采集

1. 安装相机插件：

```bash
ionic cordova plugin add cordova-plugin-camera
npm install @ionic-native/camera
```

2. 在 `app.module.ts` 中注册 `Camera`。

3. 使用示例：

```typescript
import { Camera, CameraOptions } from '@ionic-native/camera/ngx';

constructor(private camera: Camera) {}

takePhoto() {
  const options: CameraOptions = {
    quality: 50,
    destinationType: this.camera.DestinationType.DATA_URL,
    encodingType: this.camera.EncodingType.JPEG,
    mediaType: this.camera.MediaType.PICTURE
  };
  this.camera.getPicture(options).then((imageData) => {
    this.myImage = 'data:image/jpeg;base64,' + imageData;
  }, (err) => {
    console.log(err);
  });
}
```

---

## §6 Ionic + Cordova 时空数据采集

本节实现百度地图集成 + 百度定位插件，在 Ionic 中完成时空数据（位置 + 地图）采集。

### 百度地图集成

创建百度地图页面：

```bash
ionic generate page baidu-map
```

在 `home.html` 中添加导航按钮：

```html
<button ion-button navPush="BaiduMapPage">map</button>
```

在 `index.html` 中引入百度地图 JS API（注意 AK 必须为**浏览器端**类型）：

```html
<script type="text/javascript" src="http://api.map.baidu.com/api?v=2.0&ak=浏览器端AK"></script>
<script type="text/javascript" src="http://api.map.baidu.com/library/TextIconOverlay/1.2/src/TextIconOverlay_min.js"></script>
<script type="text/javascript" src="http://api.map.baidu.com/library/MarkerClusterer/1.2/src/MarkerClusterer_min.js"></script>
```

TS 中引入 JS 库的方式：

- 方法1：`var Map: any = require('/pages/Map');`
- 方法2：`import * as Map from '/pages/Map';`
- 方法3：`import Map from '/pages/Map';` + `tsconfig.json` 配置 `"allowJs": true`

在 `baidu-map.html` 添加地图容器：

```html
<ion-content padding>
  <div #map id="map_container"></div>
</ion-content>
```

在 `baidu-map.scss` 设置样式：

```scss
#map_container { width: 100%; height: 100%; }
```

在 `baidu-map.ts` 中初始化地图：

```typescript
import { Component, ViewChild, ElementRef } from '@angular/core';
declare var BMap;

@Component({...})
export class BaiduMapPage {
  @ViewChild('map') map_container?: ElementRef;
  map: any;
  marker: any;
  lat = 30.52;   // 默认纬度
  lon = 114.38;  // 默认经度

  ionViewDidEnter() {
    this.map = new BMap.Map(this.map_container.nativeElement);
    let point = new BMap.Point(this.lon, this.lat);
    this.map.centerAndZoom(point, 14);
    this.map.enableScrollWheelZoom(true);
    this.map.addControl(new BMap.ScaleControl({
      offset: new BMap.Size(5, 5),
      anchor: BMAP_ANCHOR_TOP_RIGHT
    }));
  }
}
```

> 在 Ionic 工程目录下可通过 `ionic serve` 在浏览器中预览。

### 定位服务集成

创建 NativeService provider：

```bash
ionic g provider NativeService
```

安装所需插件和 npm 包：

```bash
# 诊断权限插件
ionic cordova plugin add cordova.plugins.diagnostic
npm install @ionic-native/diagnostic

# Toast 提示插件
ionic cordova plugin add cordova-plugin-x-toast
npm install @ionic-native/toast

# 百度定位插件（注意传入移动端 AK）
cordova plugin add hewz.plugins.baidu-location --variable API_KEY="xx"

# GPS 定位（Android 端使用）
cordova plugin add cordova-plugin-geolocation --variable GPS_REQUIRED="false"
npm install @ionic-native/geolocation
```

在 `app.module.ts` 中注册 provider：

```typescript
import { Diagnostic } from '@ionic-native/diagnostic';
import { Toast } from '@ionic-native/toast';
import { Geolocation } from '@ionic-native/geolocation/ngx';
// providers: [..., Diagnostic, Toast, Geolocation]
```

---

## §7 调试与测试

- 浏览器端预览：`ionic serve`
- Android 真机/模拟器运行：`ionic cordova run android`
- 远程调试：使用 Chrome 输入 `chrome://inspect/#devices`，或 Edge 浏览器进行 WebView 调试
- 若出现 `events.js:174 throw er` 错误，尝试重新安装 `ionic@4`

---

## 课后自学参考

Ionic 教程：https://www.runoob.com/ionic/ionic-tutorial.html