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

JS->JS Bridge->Native：
1. 将 JS 侧传递的参数（通常是 JSON 字符串）反序列化为原生可识别的数据结构
2. 将此次调用注册到一个回调表中，生成唯一的回调 ID（callbackId），以便后续将结果精确返回给对应的 JS 函数
3. 根据调用的方法名路由到对应的原生模块。

Native->JS Bridge->JS:
1. 原生层获取到位置数据后，将结果封装为 JSON 格式，携带之前的 callbackId，通过 JS Bridge 回传。
2. Android 端通过 `WebView.evaluateJavascript()`（或将 JS 代码拼接到 `loadUrl("javascript:callback(data)")` 中）将结果注入到 WebView 的 JS 执行环境中。
3. JS Bridge 根据 callbackId 找到之前注册的回调函数，执行回调，H5 页面获得位置数据并更新 UI。
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