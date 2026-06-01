## Q1 Webview 和原生层是什么？JS Bridge的工作机制是什么？Hybrid Programming的请求返回机制是什么？

Hybrid programming 框架：
```
┌──────────────────────────────────┐
│           应用代码                │  
│  ┌──────────┬─────────────────┐  │
│  │  H5 内容  │  原生层（Java/   │  │  ← 这才是"原生层"
│  │ (WebView) │  Kotlin 代码）   │  │     由开发者编写的 App 代码
│  └──────────┴─────────────────┘  │
├──────────────────────────────────┤
│       Android Framework          │  ← SDK/API 层（android.location,
│       (Java API 封装层)           │      android.camera, ContentProvider 等）
├──────────────────────────────────┤
│       Android Runtime (ART)      │  ← 虚拟机，运行 Java/Kotlin 字节码
│       + 系统服务                   │     (ActivityManager, PackageManager...)
├──────────────────────────────────┤
│       HAL (硬件抽象层)            │  ← 让上层代码不依赖具体硬件品牌
├──────────────────────────────────┤
│       Linux Kernel               │  ← 驱动、内存管理、进程调度、网络协议栈
│       (真正的操作系统内核)          │
├──────────────────────────────────┤
│       硬件 (CPU/GPU/GPS/...)     │
└──────────────────────────────────┘

```
### A1.1 Webview 和 原生层

WebView 是一个**完整的浏览器引擎**，不是一段 JS 代码，也不是一个桥接工具。它由操作系统或应用框架提供，本质上是一个可嵌入原生应用的"迷你浏览器"。WebView 内部包含多个子系统：

| 子系统                          | 职责                                 |
| ---------------------------- | ---------------------------------- |
| **渲染引擎**（Webkit/Blink）       | 解析 HTML/CSS，绘制页面像素                 |
| **JS 引擎**（V8/JavaScriptCore） | 解释执行 JavaScript 代码                 |
| **网络栈**（即"网络层"）              | 真正负责发送 HTTP 请求、建立 TCP 连接、处理 TLS 加密 |
| **存储层**                      | 管理 Cookie、LocalStorage、Cache 等     |
### A1.2 JS Bridge 

Js Bridge 指的就是 Webview 中的 JS 引擎与原生层之间的交互方式

JS->原生层：[[第2章-H5交互调用#§2 JS 调用原生应用]]
原生层->JS:[[第2章-H5交互调用#§3 原生应用调用 JS]]
### A1.3 Hybrid的请求链路

请求可以分为两种，一种调用原生系统（如android）接口，一种是向服务器发起请求

 __H5 调原生能力__：H5 (JS) → JS Bridge → Native (Android/iOS) → 硬件API → Native → JS Bridge → H5 回调

__H5 调远程服务__：
```
H5 层 JS（调用 fetch）
    ↓
WebView JS 引擎（解释执行）
    ↓
WebView 网络栈（发起真实 HTTP/TCP 连接，处理 DNS 解析、TLS 握手、数据收发）
    ↓
互联网 → 远程服务器

```

## Q2 回调函数和 promise 机制是什么

### A2.1 同步回调、异步回调和回调地狱

回调函数：__把一个函数 A 作为参数传给另一个函数 B，等 B 在某个时机回头调用 A__。B 叫"高阶函数"，A 叫"回调函数"

**同步回调**示例：
```js
// 定义一个高阶函数：它接收另一个函数作为参数
function greet(name, callback) {
    console.log("开始执行 greet 函数");
    callback(name);
    //这里callback就是回调函数，在这里只是占位，实际上callback=(name) => {console.log("你好, " + name + "! 这是在回调里执行的。");
    console.log("greet 函数结束");
}

// 调用 greet，把箭头函数作为回调传进去
greet("张三", (name) => {
    console.log("你好, " + name + "! 这是在回调里执行的。");
});

// 输出顺序：
// 开始执行 greet 函数
// 你好, 张三! 这是在回调里执行的。
// greet 函数结束
```

**setTimeout**的第一个参数是回调函数，第二个参数是延迟时间
由于Javascript是非抢占式单线程，而我们又有异步需求，所以使用事件循环+setTimeout来实现异步，**异步机制**如下：
1. 程序正常运行
2. 遇到setTimeout时，把回调函数和延迟事件交给定时器系统，程序跳过回调函数向下运行
3. 定时器系统发现延迟时间到了，将回调函数放回任务队列，等待执行
4. 主线程空闲时，事件循环执行任务队列中的任务，此时回调函数被执行

**异步回调**示例：
```js
console.log("1. 开始请求数据...");

// setTimeout 是浏览器/Node 提供的异步 API
// 第一个参数是回调函数，第二个参数是延迟毫秒数
setTimeout(() => {
    // 这个回调不会立即执行，2 秒后才被调用
    console.log("3. 数据到了！模拟服务器返回：{ name: '张三', age: 25 }");
}, 2000);

console.log("2. 请求已发出，继续干别的事...");

// 实际输出顺序：
// 1. 开始请求数据...
// 2. 请求已发出，继续干别的事...
// （等 2 秒...）
// 3. 数据到了！模拟服务器返回：{ name: '张三', age: 25 }

```

**回调地狱**：如果任务之间有依赖关系，嵌套函数会越来越深
回调地狱示例：
```js
// 三层嵌套——"回调地狱"
login("zhangsan", (token) => {
    getUserInfo(token, (userInfo) => {
        getOrders(userInfo.id, (orders) => {
            console.log("最终拿到订单:", orders);
            // 如果还有下一步，还得再套一层……
        });
    });
});
```

此时错误处理极其麻烦，只要有一步出错就无法进行
### A2.2 Promise 机制

Promise 是 ES6 引入 的**专门解决回调地狱的异步处理方案**。它把异步操作包装成一个对象，这个对象有三种状态：

- __pending（等待中）__：还没出结果
- __fulfilled（已成功）__：`resolve(value)` 被调用
- __rejected（已失败）__：`reject(error)` 被调用

生产者和消费者分离是promise的核心思想：

- __生产者__（`new Promise(...)`）：负责执行异步任务，成功后叫 `resolve`，失败后叫 `reject`
- __消费者__（`.then()` / `.catch()`）：不关心任务怎么完成，只管"拿到结果后做什么"

生产者、消费者示例：
```js
// ========== 生产者：创建 Promise ==========
function fetchData() {
    return new Promise((resolve, reject) => {
        // 这个函数体内的代码立即同步执行
        console.log("发起网络请求...");

        setTimeout(() => {
            // 模拟：80% 概率成功，20% 概率失败
            const success = Math.random() > 0.2;

            if (success) {
                // 成功：调用 resolve，把数据传出去
                resolve({ status: 200, data: "服务器返回的数据" });
            } else {
                // 失败：调用 reject，把错误传出去
                reject("网络请求失败，状态码 500");
            }
        }, 2000);  // 模拟 2 秒网络延迟
    });
}

// ========== 消费者：使用 Promise ==========
const promise = fetchData();  // 得到的是一个 Promise 对象（状态 pending）

promise
    .then((result) => {   // 对应 resolve：拿到成功结果
        console.log("成功！数据：", result.data);
    })
    .catch((error) => {   // 对应 reject：拿到失败原因
        console.log("失败！原因：", error);
    });

// 以上输出（成功时）：
// 发起网络请求...
// （等 2 秒...）
// 成功！数据：服务器返回的数据

```

此时的嵌套结构被展平：
```js
// ========== 链式调用：扁平化，不再嵌套 ==========
login("zhangsan")
    .then(token => getUserInfo(token))     // 第一个 .then 返回新的 Promise
    .then(userInfo => getOrders(userInfo))  // 第二个 .then 等上一个完成才执行
    .then(orders => {
        console.log("最终订单数据:", orders);
    })
    .catch(error => {
        // 任何一步出问题，都会跳到这里统一处理
        console.log("流程中某一步出错:", error);
    });
```

## Q3 NPM包是什么，里面各个组件的作用是什么

### A3.1 NPM 包的结构

NPM 包是一个遵循 package.json 规范的目录，包含：元数据（package.json）+ 自己编写并导出的 JS 模块（通过 exports/import 暴露函数和对象）+ 依赖声明（dependencies/devDependencies 字段，记录需要哪些第三方包） + 实际下载到本地的依赖副本（node_modules/ 目录，不参与发布）。

```
my-package/                  ← 这才是"一个包"：一个完整的目录
├── package.json             ← 包的身份证：定义名字、版本、入口文件、依赖
├── index.js                 ← 主模块（包的默认入口）
├── lib/                     ← 其他功能模块
│   ├── utils.js
│   └── helper.js
├── test/                    ← 单元测试
│   └── test.js
├── doc/                     ← 文档
└── node_modules/            ← 这个包依赖的其他包的实际代码
    ├── lodash/
    ├── axios/
    └── ...

```

### A3.2 NPM 的导出和使用

以ES6为例

index.js中规定导出的函数：
```javascript
// index.js
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
export default { add, subtract };  // 默认导出
```

使用时从包中import对应函数：
```javascript
import { add, subtract } from 'my-package';
add(3, 5);  // 8
```

### A3.3 NPM 的上传和下载

下载流程：
```
终端输入
    ↓
① 解析命令与参数
    ↓
② 查询 registry，获取包元数据
    ↓
③ 解析依赖树，确定所有需要下载的包及版本
    ↓
④ 下载包到本地缓存（~/.npm/_cacache/）
    ↓
⑤ 从缓存解压到当前项目的 node_modules/
    ↓
⑥ 更新 package.json 和 package-lock.json

```

上传流程：
```
npm publish
    ↓
① 本地验证与打包准备
    ↓
② 向 registry 发起 PUT 请求，上传 tgz 压缩包
    ↓
③ registry 验证包名冲突、版本号、权限
    ↓
④ 存储并索引，包对外可用
    ↓
⑤ 返回成功，终端显示 +包名@版本号

```

## Q4 Angular 和普通 H5 的关系是什么? Angular 内部组件的功能是什么？
### A4.1 Angular 和 H5 的关系

Angular 是造在 H5（HTML/CSS/JS）地基之上的前端框架，底层仍是标准 Web 技术，但通过 TypeScript + 组件化 + 指令系统 + 数据绑定，把原始 H5 变成了一整套工业化开发体系

具体表现：
- Angular 的 __Template（模板）__ 就是 HTML 文件，但它被扩展了——加入了 `*ngIf`、`*ngFor`、`[property]="value"`、`(event)="handler()"` 等 Angular 特有的指令语法（笔记第 31 行）。
- Angular 的 __样式__ 就是 CSS/SCSS（笔记第 82 行），完全标准 H5 样式。
- Angular 的 __组件类__ 用 TypeScript 编写，最终被 Angular 编译器（ngc）编译为标准 JavaScript，在浏览器中执行。

ts为js的超集，scss为css的超级，通过ngc编译转化：
- .ts编译为.js
- .scss编译为.css
### A4.2 angular 内部架构

完整angular目录树及各文件功能：
```
my-angular-app/                          ← 项目根目录（本身就是一个 NPM 包）
│
├── 📁 node_modules/                     ← 【依赖实体代码】
│   ├── @angular/                          npm install 时自动下载的所有第三方包
│   ├── rxjs/                              包括 Angular 核心、RxJS、TypeScript 等
│   └── ...                                不会被提交到 Git，由 package.json 描述即可还原
│
├── 📁 src/                              ← 【源码目录——你写代码的地方】
│   │
│   ├── 📁 app/                          ← 【应用主模块——所有组件和服务都在这里】
│   │   ├── app.component.ts                根组件：业务逻辑（TypeScript 类）
│   │   ├── app.component.html              根组件：视图模板（HTML + Angular 指令）
│   │   ├── app.component.scss              根组件：私有样式（仅作用于本组件）
│   │   ├── app.component.spec.ts           根组件：单元测试文件
│   │   ├── app.module.ts                   根模块：声明组件、导入外部模块、注册服务
│   │   ├── app-routing.module.ts           路由模块：定义 URL 路径 → 组件的映射
│   │   │
│   │   ├── 📁 data-collect/             ← 【示例：你用 ng g component 创建的自定义组件】
│   │   │   ├── data-collect.component.ts       采集组件：业务逻辑
│   │   │   ├── data-collect.component.html     采集组件：表单模板
│   │   │   ├── data-collect.component.scss     采集组件：私有样式
│   │   │   └── data-collect.component.spec.ts  采集组件：单元测试
│   │   │
│   │   ├── 📁 data-show/                ← 【示例：展示数据的自定义组件】
│   │   │   └── ...（同上四个文件）
│   │   │
│   │   ├── sitedata.ts                  ← 【数据模型类】定义采集数据的结构（字段+类型）
│   │   ├── sites.ts                     ← 【数据存储类】维护数组，提供增删查方法
│   │   └── sitedataservice.service.ts   ← 【共享服务】跨组件的数据池，单例注入
│   │
│   ├── 📁 assets/                       ← 【静态资源】图片、字体、JSON 配置文件
│   │   └── .gitkeep                        保证空目录也能被 Git 追踪
│   │
│   ├── 📁 environments/                 ← 【环境配置】不同环境用不同的变量值
│   │   ├── environment.ts                  开发环境：API 地址指向本地
│   │   └── environment.prod.ts             生产环境：API 地址指向线上服务器
│   │
│   ├── favicon.ico                      ← 浏览器标签页上的小图标
│   ├── index.html                       ← 【单页应用的唯一 HTML 文件】
│   │                                        所有页面都在这个文件里动态渲染
│   │                                        <app-root></app-root> 是 Angular 的挂载点
│   ├── main.ts                          ← 【应用入口】引导启动 AppModule
│   │                                        platformBrowserDynamic().bootstrapModule(AppModule)
│   ├── polyfills.ts                     ← 【浏览器兼容补丁】让老浏览器支持新特性
│   └── styles.scss                      ← 【全局样式】作用于整个应用的所有组件
│
├── .editorconfig                        ← 编辑器配置（缩进风格、字符集等，团队统一）
├── .gitignore                           ← Git 忽略规则（node_modules/ 等不提交）
├── angular.json                         ← 【Angular CLI 工程配置】
│                                            定义项目名、源码目录、构建选项、端口号等
├── karma.conf.js                        ← 单元测试运行器配置（Karma）
├── package.json                         ← 【包描述文件】项目名、版本、依赖声明、scripts 脚本
├── package-lock.json                    ← 【依赖版本快照】锁定每个包的精确版本号
├── tsconfig.json                        ← 【TypeScript 编译配置】
│                                            目标 JS 版本、严格模式、路径别名等
├── tsconfig.app.json                    ← 应用代码专用的 TS 配置（继承 tsconfig.json）
└── tsconfig.spec.json                   ← 测试代码专用的 TS 配置

```

精简的三个层次：
```
第一层 — 外部壳
├── package.json           → "我是什么包、我需要谁"（依赖声明）
├── angular.json           → "这个 Angular 项目怎么编译、怎么启动"
└── node_modules/          → "依赖包的实体代码都在这里"

第二层 — 入口与全局
├── src/index.html         → 唯一 HTML 页面，<app-root> 是挂载点
├── src/main.ts            → 启动 AppModule 的入口脚本
├── src/styles.scss        → 全局样式
└── src/app/app.module.ts  → 根模块：声明所有组件、导入外部模块

第三层 — 应用代码（src/app/ 下）
├── app.component.*        → 根组件（外壳框架，通常放导航栏 + <router-outlet>）
├── *.component.*          → 每个自定义组件 = .ts(逻辑) + .html(模板) + .scss(样式)
├── *.service.ts           → 跨组件共享数据与业务逻辑
├── *.ts                   → 数据模型类（定义字段结构）
└── app-routing.module.ts  → URL ↔ 组件的路由映射表

```

## Q5 JS 和 原生层 之间各种交互方式的原理是什么？

### A5.1 JS->原生层

[[第2章-H5交互调用#§2 JS 调用原生应用]]中给出了三种方法 ，这里结合代码详细解释

1. 对象映射：在初始化webview的时候，将一个java对象（经过`@JavascriptInterface` 注释允许被JS调用）直接注入 JS 全局作用域（指通过`webView.addJavasriptInterface`函数创建window对象），JS 可以操作这个对象，但是实际上触发的是原生层中java对象

```
┌─ H5 (WebView 中的 JS) ──────────────────────────────────┐
│                                                          │
│  window.launcher.showToast("你好")                       │  ← JS 以为在调 JS 方法
│       │                                                  │
└───────┼──────────────────────────────────────────────────┘
        │  WebView 内部拦截了这个"假调用"
        │  通过 JNI 跨语言调用
        ▼
┌─ 原生层 (Java/Kotlin) ──────────────────────────────────┐
│                                                          │
│  class JsInterface {                                    │
│    @JavascriptInterface                                  │  ← 这个注解就是"允许 JS 调我"
│    public void showToast(String msg) {                  │
│      Toast.makeText(context, msg, ...).show();          │  ← 真正执行原生功能
│    }                                                     │
│  }                                                       │
│                                                          │
│  // WebView 初始化时                                   │
│  webView.addJavascriptInterface(                         │
│    new JsInterface(this),  // 创建原生对象               │
│    "launcher"              // 映射为 JS 中的 window.launcher
│  );                                                      │
└──────────────────────────────────────────────────────────┘

```

2. URL拦截：JS 不能调用原生方法，但是通过URL跳转可以触发webView导航。原生层在webView中设置一个接受URL的函数作为门卫（示例中的`shouldOverrideUrlLoading`函数），这个门卫在接受特殊格式的URL（示例中的`js://xxxx`）时并不执行页面跳转，而是解析里面的信息并执行操作；当接受正常格式的URL时，则进行页面跳转

```
┌─ H5 ────────────────────────────────────────────────────┐
│                                                          │
│  // JS 构造一条特殊的 URL                               │
│  window.location.href = "js://showToast?msg=你好&duration=long";
│       │                                                  │
│       │  这个 URL 要离开 WebView 去加载                   │
└───────┼──────────────────────────────────────────────────┘
        │
        ▼
┌─ 原生层的 WebViewClient ────────────────────────────────┐
│                                                          │
│  @Override                                               │
│  public boolean shouldOverrideUrlLoading(               │
│      WebView view, String url) {                        │
│                                                          │
│    if (url.startsWith("js://")) {   // 发现了！是我们约定的协议 │
│      // 解析 URL：js://showToast?msg=你好&duration=long │
│      String method = url.substring(5, url.indexOf("?"));// "showToast"
│      String params = url.substring(url.indexOf("?")+1); // "msg=你好&duration=long"
│                                                        │
│      // 根据方法名执行对应逻辑                          │
│      if (method.equals("showToast")) {                 │
│        String msg = parseParam(params, "msg");         │
│        Toast.makeText(context, msg, ...).show();       │
│      }                                                  │
│      return true;  // ← 关键！返回 true 表示"我处理了，别真导航" │
│    }                                                    │
│    return false; // 普通 URL 正常加载                    │
│  }                                                      │
└──────────────────────────────────────────────────────────┘

```

3. 对话框拦截：JS有三个内置ui api函数（alert，confirm和prompt），创建webView时将这三个函数暴露给原生层。当JS发出带特殊的message的prompt请求时，执行解析并执行特定操作，这个方法可以有返回值

```
┌─ H5 ─────────────────────────────────────────────────────────┐
│                                                               │
│  // JS 调用内置 prompt()，但它不是真弹窗                       │
│  var result = prompt("native:getLocation", "");               │
│  // prompt(消息, 默认值) —— 这里的"消息"就是我们要传的指令     │
│  // 返回值就是原生层回传的数据                                  │
│       │                                                       │
└───────┼───────────────────────────────────────────────────────┘
        │  WebView 内核检测到 prompt() 被调用
        │  触发 WebChromeClient.onJsPrompt() 回调
        ▼
┌─ 原生层的 WebChromeClient ───────────────────────────────────┐
│                                                               │
│  @Override                                                    │
│  public boolean onJsPrompt(WebView view, String url,          │
│      String message,        // "native:getLocation"           │
│      String defaultValue,   // ""（不用）                     │
│      JsPromptResult result) // 用来回传给 JS 的通道            │
│  {                                                            │
│    if (message.startsWith("native:")) {   // 发现暗号！       │
│      String cmd = message.substring(7);  // "getLocation"     │
│      // 执行原生操作...                                       │
│      String response = getNativeLocation();  // "经度:120, 纬度:30" │
│                                                               │
│      result.confirm(response); // ← 把结果传回给 JS 的变量！   │
│      return true;  // 返回 true 表示"我处理了，别弹系统对话框"  │
│    }                                                           │
│    return false; // 不是暗号，正常弹窗                          │
│  }                                                             │
└───────────────────────────────────────────────────────────────┘
        │
        │  result.confirm(response) 把数据传回
        ▼
┌─ H5 ─────────────────────────────────────────────────────────┐
│  var location = prompt("native:getLocation", "");             │
│  console.log(location);  // "经度:120, 纬度:30"               │
│  ← JS 同步拿到了原生的返回结果！                               │
└───────────────────────────────────────────────────────────────┘

```

**三者的对比总览**:

|维度|方法1：对象映射|方法2：URL 拦截|方法3：弹窗拦截|
|---|---|---|---|
|JS 端语法|`window.launcher.xxx()`|`location.href = 'js://...'`|`prompt('native:xxx')`|
|JS 能收到返回值|✅ 直接 return|❌ 需原生反向调 JS|✅ prompt 可同步接收|
|传递数据量|中等|受 URL 长度限制|受字符串限制|
|安全性（Android 4.2-）|❌ 有漏洞|✅ 安全|✅ 安全|
|适合大规模方法数|✅ 一次注入，多个方法|⚠️ 需自己维护 case 分支|⚠️ 所有调用走同一个回调|
|实际项目常用度|⭐⭐⭐⭐⭐ (最常用)|⭐⭐⭐|⭐⭐|
### A5.2 原生层->JS

1. evaluateJavaScript ✔
2. loadUrl ×

## Q6 写完代码如何打包、发布、测试？为什么需要apk签名？

[[第2章-H5交互调用#环境与打包工具]]中的表格并不是很明确，这里稍微解释一下

### A6.1 各个工具的作用

| 工具                    | 用途                            |
| --------------------- | ----------------------------- |
| **Android SDK + JDK** | 环境准备：前者编译android代码，后者编译java代码 |
| **Gradle**            | 打包工具：形成.apk文件                 |
| **emulator**          | 虚拟机：模拟移动端                     |
| **adb**               | 调试桥：用于连接移动端进行调试               |

### A6.2 apk签名

APK签名的作用：

- __身份认证__：证明"这个 APK 就是我开发的，没有被别人篡改过"。应用商店通过签名判断更新包是否来自同一开发者，签名不对就无法覆盖安装旧版本。

- __完整性校验__：安装时 Android 系统会验证 APK 内文件的签名是否一致，任何一个文件被篡改都会导致签名验证失败，安装直接拒绝。

- __权限信任__：同一签名的应用可以共享数据（如果声明了 `sharedUserId`），系统层面实现应用间互信。

APK签名的流程：
1. kaytool生成密钥库
2. apksigner使用密钥库给apk签名
3. apksigner验证签名是否成功

具体代码示例：
```
# 步骤1：生成密钥库（keystore）——只在第一次发布时需要，以后用同一个
keytool -genkey \
  -alias helloalias \           # 别名：这把密钥的名字，一个 keystore 里可以存多把
  -keyalg RSA \                 # 密钥算法：RSA 2048 位，目前行业标准
  -keysize 2048 \
  -validity 36500 \             # 有效期：36500 天 ≈ 100 年（基本永久）
  -keystore hello.keystore      # 输出文件：密钥库文件，妥善保管，丢了就永远发不了更新

# 执行后会交互式提问：
# 姓名、组织、城市、国家等 → 这些信息会被编入证书
# 需要设置密钥库密码和密钥密码

# 步骤2：用生成的密钥签名 APK
apksigner sign \
  --ks hello.keystore \                    # 指定 keystore 文件
  --out app-release-signed.apk \           # 输出：已签名的 APK
  app-release-unsigned.apk                 # 输入：未签名的 APK

# 步骤3：验证签名是否成功
apksigner verify -v app-release-signed.apk
# 输出类似：
# Verifies
# Verified using v1 scheme (JAR signing): true
# Verified using v2 scheme (APK Signature Scheme v2): true
# Verified using v3 scheme (APK Signature Scheme v3): true

```

## Q7 cordova 中 plugin.xml 的作用是什么

### A7 从 plugin.xml 看 cordova

plugin.xml框架：
``` 
┌───────────────────────────────────────────────────────┐
│                   plugin.xml                          │
│                                                       │
│  ① 备案：告诉 Cordova 这个插件叫什么、有哪些 JS 方法   │
│     → <js-module> + <clobbers>                        │
│                                                       │
│  ② 接线：把 JS 方法名和原生类关联在一起                │
│     → <feature name="xxx"> + <param name="android-package" value="..."/> │
│                                                       │
│  ③ 搬运：把插件的源文件从插件目录复制到平台工程对应位置    │
│     → <source-file src="..." target-dir="..." />      │
│     → <config-file target="..." parent="...">         │
│                                                       │
└───────────────────────────────────────────────────────┘

```

plugin.xml代码示例：
```xml
<plugin id="com.testdialog" version="1.0.0"
        xmlns="http://apache.org/cordova/ns/plugins/1.0">

    <!-- ============ 第一部分：JS 端注册 ============ -->
    <!-- 告诉 Cordova：生成 cordova_plugins.js 时加入这个模块 -->
    <js-module src="www/TestDialog.js" name="TestDialog">
        <!-- 前端通过这个名字调用 -->
        <clobbers target="cordova.plugins.TestDialog" />
        <!--          ↑                               -->
        <!--  生成的全局对象路径                       -->
        <!--  相当于 window.cordova.plugins.TestDialog -->
    </js-module>
    <!-- 效果：cordova.js 会自动把 www/TestDialog.js 的内容挂到 cordova.plugins.TestDialog 上 -->


    <!-- ============ 第二部分：Android 平台配置 ============ -->
    <platform name="android">

        <!-- 2a. 接线：告诉 Cordova 原生实现类在哪里 -->
        <config-file target="res/xml/config.xml" parent="/*">
            <!-- feature name 必须和 js-module 的 name 一致，否则匹配不上！ -->
            <feature name="TestDialog">
                <param name="android-package" value="com.testdialog.TestDialog" />
                <!--                      ↑ 完整包名.类名                       -->
            </feature>
        </config-file>

        <!-- 2b. 搬运：把 Java 源文件复制到 Android 工程 -->
        <source-file
            src="src/android/TestDialog.java"
            target-dir="src/com/testdialog" />
        <!-- 插件目录: plugins/xxx/src/android/TestDialog.java           -->
        <!-- 复制到:   platforms/android/.../src/com/testdialog/TestDialog.java -->

    </platform>

</plugin>

```

对应的映射关系：
```
┌────────────────────────────────────────────────────────────────┐
│                     plugin.xml 映射关系图                       │
│                                                                │
│  [JS 调用时]                                                   │
│  cordova.plugins.TestDialog.coolMethod("hello", succ, err)     │
│       │                                                        │
│       │ clobbers target="cordova.plugins.TestDialog"           │
│       ▼                                                        │
│  www/TestDialog.js                                             │
│       │                                                        │
│       │ cordova.exec(succ, err, "TestDialog", "coolMethod",..) │
│       ▼                                                        │
│  exec() 查找 feature name="TestDialog"                         │
│       │                                                        │
│       │ param android-package="com.testdialog.TestDialog"      │
│       ▼                                                        │
│  com.testdialog.TestDialog.execute("coolMethod", args, cb)     │
│       │                                                        │
│       │ 找到这个文件是因为 source-file 把它搬到了正确位置         │
│       ▼                                                        │
│  target-dir="src/com/testdialog" → com.testdialog.TestDialog   │
│                                                                │
└────────────────────────────────────────────────────────────────┘

```

## Q8  CORS 的作用是什么

### A8.1 CORS 的目的

[[第3章-服务端#CORS 跨域处理]]：限制网页从不同源（域名、协议、端口）请求资源

### A8.2 CORS 的处理方式

__处理方式__（笔记第 117-126 行）：

```jsp
response.setHeader("Access-Control-Allow-Origin", "*");       // 允许所有来源
response.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS");
response.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");

// OPTIONS 预检请求——浏览器先发一个 OPTIONS 探路
if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
    response.setStatus(HttpServletResponse.SC_OK);
    return;  // 直接返回，不需处理业务逻辑
}
```

__执行流程__：

```javascript
浏览器                    JSP 服务端
  │                         │
  │──── OPTIONS 预检 ──────→│  ① 先问：允许 POST 吗？
  │←─── CORS 头（200）──────│  ② 答：允许，所有来源都行
  │                         │
  │──── POST + JSON body ──→│  ③ 发送真正的请求
  │←─── JSON 响应 ──────────│  ④ 返回数据
```

如果 JSP 没设置 CORS 头，浏览器会在第二步直接报错 `blocked by CORS policy`。

## Q9 数据处理在哪里进行？

### A9 数据库和后端的联动

用数据库存储函数（Stored Function）封装数据处理逻辑，而不是把 SQL 写在 JSP 里

并且需要注意，当采取统一分发的时候，可以通过一个url+不同param来对应不同的数据库函数