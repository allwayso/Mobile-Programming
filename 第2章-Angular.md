# 第2章 Angular移动端信息采集和展示

> [!SUMMARY] Table of Contents
>
> - [[第2章-Angular#第2章 Angular移动端信息采集和展示|第2章 Angular移动端信息采集和展示]]
>   - [[第2章-Angular#目录|目录]]
>   - [[第2章-Angular#§1 Angular 简介|§1 Angular 简介]]
>   - [[第2章-Angular#§2 Angular 核心概念|§2 Angular 核心概念]]
>   - [[第2章-Angular#§3 Angular 环境配置与项目创建|§3 Angular 环境配置与项目创建]]
>   - [[第2章-Angular#§4 组件开发实战|§4 组件开发实战]]
>   - [[第2章-Angular#§5 路由与模板|§5 路由与模板]]
>   - [[第2章-Angular#§6 数据模型与双向绑定|§6 数据模型与双向绑定]]
>   - [[第2章-Angular#§7 服务与数据共享|§7 服务与数据共享]]
>   - [[第2章-Angular#§8 小结|§8 小结]]

---

## §1 Angular 简介

Angular 是 Google 开发的**开源前端框架**，目前架构在 TypeScript 之上。它和 AngularJS 1.x 完全不同——不再有控制器（Controller）和作用域（Scope），完全基于组件导向。Angular CLI 用于便捷创建和管理组件。

## §2 Angular 核心概念

Angular 提供组件化开发，MVC 架构支持。核心概念：

| 概念 | 说明 |
|------|------|
| **Module（模块）** | 代码组织单元，NgModule 装饰器声明 |
| **Component（组件）** | 视图 + 逻辑的最小单元，含模板和类 |
| **Template（模板）** | HTML 视图，支持 Angular 指令语法 |
| **Metadata（元数据）** | 装饰器参数，告诉 Angular 如何处理类 |
| **Data binding（数据绑定）** | `[]` 模型→视图，`()` 视图→模型，`[()]` 双向 |
| **Service（服务）** | 可注入的单实例，共享数据与功能 |
| **Directive（指令）** | 扩展 HTML 行为 |
| **Dependency injection（依赖注入）** | 构造函数自动注入服务实例 |
| **Route（路由）** | 页面间导航 |

## §3 Angular 环境配置与项目创建

基本命令：

```bash
npm install -g @angular/cli@16.2.15    # 安装 Angular CLI
ng new my-angular-app --skip-tests     # 创建项目（跳过测试）
ng serve --open                         # 启动开发服务器（端口4200）
```

项目目录结构：`src/app/` 下为应用代码，`package.json` 管理依赖。Angular 项目本身是一个 NPM 包，可用 `npm i` 安装依赖。

## §4 组件开发实战

以时空数据采集应用为例：

**创建组件**：`ng g component data-collect`，生成四个文件（.ts、.html、.scss、.spec.ts）。

**创建类/模型**：`ng g class data-collect/sitedata --skip-tests`
```typescript
export class Sitedata {
  constructor(
    public time: string | null,
    public name: string | null,
    public value: string | null,
    public x_loc: number | null,
    public y_loc: number | null
  ) { }
}
```

**创建数据存储类**：`ng g class data-collect/sites --skip-tests`
```typescript
export class Sites {
  sitedatalist: Sitedata[] = [];
  getData(): Sitedata[] { return this.sitedatalist; }
  addSiteData(sitedata: Sitedata) { this.sitedatalist.push(sitedata); }
}
```

**组件 TS 逻辑**：导入模型和数据，初始化属性，定义 `submit()` 方法。

**组件模板**：表单含采集时间、经度、纬度、属性名、属性值五个输入框，提交和重置按钮。

**样式**：用 SCSS 控制 `form-group`、`label`、`container` 布局，`margin: 5px 0`、`display: inline-block` 等。

## §5 路由与模板

**路由配置**（app-routes.ts）：定义路径映射到组件
```typescript
const routes: Routes = [
  { path: 'data-collect', component: DataCollectComponent },
  { path: 'data-show', component: DataShowComponent }
];
```

**导航**：`app.component.html` 中使用 `<a routerLink="/data-collect">` 实现路由跳转，`routerLinkActive="active"` 高亮当前页。

**动态显示**：用 `*ngIf` 条件渲染、`*ngFor` 遍历列表、`{{ }}` 插值表达式。

## §6 数据模型与双向绑定

**数据绑定三种方式**：

- **单向（模型→视图）**：`[value]="sitedata.time"`——属性绑定
- **单向（视图→模型）**：`(input)="sitedata.time=$any($event.target).value"`——事件绑定
- **双向绑定**：`[(ngModel)]="sitedata.time"`——需在 `app.module.ts` 中引入 `FormsModule`

**列表展示模板**：
```html
<tr *ngFor="let sitedata of sitedatalist">
  <td>{{ sitedata.time }}</td>
  <td>{{ sitedata.name }}</td>
  ...
</tr>
```

表格样式通过 SCSS 控制：边框 `1px solid grey`、间距 `padding: 5px`、奇偶行交替背景色（`nth-child(odd/even)`）。

## §7 服务与数据共享

服务（Service）可声明为**单实例**，在不同组件间共享数据：
```bash
ng g service sitedataservice --skip-tests
```

**两种注入方式**：
1. `@Injectable({ providedIn: 'root' })`——推荐
2. 在 `app.module.ts` 的 `providers` 数组中声明

**数据采集组件注入服务**：
```typescript
constructor(public sitedataserviceservice: SitedataserviceService) { }
submit() {
  this.sitedataserviceservice.addSiteData(this.sitedata);
}
```

**数据展示组件注入服务**：在 `ngOnInit()` 中调用 `getSiteDataList()` 获取数据并绑定到模板。

**导航栏动态计数**：`{{sitedataserviceservice.getSiteDataList().length}}` 实时显示已采集条目数。

## §8 小结

Angular 核心能力总结：模块（module）、组件（component）、模板（template）、类（class）、服务（service）、指令（directive）、SCSS 样式、路由导航、依赖注入。

后续可扩展：持久化存储、时空信息自动填充、删除功能等。

> 课后作业：用 Angular 编写时空数据采集应用，包含数据录入表单和结果展示功能，打包 src 目录和截图提交。