
# 微前端“无框架”Shell/Remote/Component 架构方案与实施清单

目标：构建一个“框架无关”的前端微服务运行时与规范，让开发者可以分别基于任意框架（Vue/React/Angular/Svelte/…）开发 Shell、Remote、Component。Shell 负责装配 Remote，Remote 装配同框架的 Component；支持本地开发与远程加载并存，支持单页多框架并行运行。

---

## 1. 目标与非目标

- 目标
  - **框架无关**：核心运行时不绑定具体前端框架。
  - **组合可插拔**：Shell 动态加载远程 Remote，Remote 动态加载同框架 Component。
  - **多框架共存**：同一页面可同时运行 Vue/React/Angular 等多个 Remote。
  - **多加载模式**：支持懒加载、预加载、远程 CDN、以及本地覆盖（local override）。
  - **隔离与协作并存**：在 CSS/JS/状态/路由上有限隔离，并提供受控通信机制。
  - **生产可用**：具备版本治理、回滚、监控、灰度、权限与安全策略。

- 非目标
  - 不追求统一一个框架；不强制 SSR；不强制使用某个打包器或单一加载技术。

---

## 2. 总体架构

- Shell（组合层）
  - 负责应用启动、全局路由、远程服务发现、加载与生命周期管理、全局状态与通信总线、安全策略、监控与灰度。
- Remote（业务域前端微应用）
  - 以“远程模块”形式暴露：页面、区块、以及若干组件。Remote 只加载与自身框架一致的 Component。
- Component（同框架组件）
  - 由 Remote 内部管理或对外暴露供其他 Remote（同框架）编排。定义统一的 mount/update/unmount 契约。

- 关键横切能力
  - Loader（ESM 动态 import / Module Federation / SystemJS 兼容）、Manifest 与注册中心、依赖共享策略、隔离与沙箱、跨应用通信、版本治理、可观测性、安全与权限、Dev/CI/CD 工具链。

---

## 3. 核心运行时与契约

### 3.1 最小运行时契约（TypeScript）

```ts
export type FrameworkId = 'vue3' | 'react18' | 'angular16' | string;

export interface RuntimeContext {
  shellVersion: string;
  frameworkAdapters: Record<FrameworkId, FrameworkAdapter>;
  eventBus: EventBus;
  router: RouterAPI; // 壳路由
  registry: RemoteRegistryAPI; // 远程服务发现
  security: SecurityAPI;
  env: Record<string, string>;
  // 可扩展
}

export type Unmount = () => Promise<void> | void;

export interface ComponentContract {
  mount: (el: HTMLElement, props?: any, ctx?: RuntimeContext) => Promise<Unmount> | Unmount;
  update?: (props?: any) => void;
}

export interface RouteDef {
  path: string; // e.g. "/orders/:id"
  expose: string; // 暴露的模块名，如 "pages/OrderDetail"
}

export interface RemoteContract {
  id: string;
  version: string;
  framework: FrameworkId;
  routes?: RouteDef[];
  getComponent?: (name: string) => Promise<ComponentContract>;
  // 页面级装配（可选，若 routes 声明）
  mountRoute?: (
    routePath: string,
    el: HTMLElement,
    routeParams: Record<string, string>,
    props?: any,
    ctx?: RuntimeContext
  ) => Promise<Unmount> | Unmount;
  // 预加载钩子
  prefetch?: (hint?: any) => Promise<void>;
  // 元信息
  meta?: Record<string, any>;
}

export interface FrameworkAdapter {
  // 将“原生框架组件/应用”包裹为协议一致的 ComponentContract
  wrapComponent: (impl: any) => ComponentContract;
  // Remote 层的应用装配（如子应用根实例）
  createRemoteApp?: (entry: any, options?: any) => RemoteContract;
}
```

### 3.2 生命周期

- Shell
  - init() → register adapters → load registry → route match → load remote → mount
- Remote
  - prefetch? → mountRoute 或 getComponent+mount
- Component
  - mount(el, props, ctx) → update(props?) → unmount()

### 3.3 Manifest（远程描述文件）

```json
{
  "id": "billing-vue",
  "version": "1.4.3",
  "framework": "vue3",
  "entry": "https://cdn.example.com/billing/1.4.3/entry.js",
  "integrity": "sha256-xxxxx",
  "routes": [
    { "path": "/billing", "expose": "pages/BillingHome" },
    { "path": "/billing/invoices/:id", "expose": "pages/InvoiceDetail" }
  ],
  "components": {
    "InvoiceList": "components/InvoiceList",
    "InvoiceCard": "components/InvoiceCard"
  },
  "styles": [
    "https://cdn.example.com/billing/1.4.3/style.css"
  ],
  "shared": {
    "vue": "^3.4.0"
  },
  "assetsBase": "https://cdn.example.com/billing/1.4.3/"
}
```

---

## 4. 加载与服务发现

- 加载技术选型（优先顺序建议）
  - **ESM 动态 import（推荐）**：最简单、浏览器原生、与 CDN 友好。要求远端响应 CORS，入口为 ESM。
  - Module Federation（Webpack 生态）：
    - 优点：可声明 shared 单例，天然微前端经验丰富。
    - 缺点：对打包器耦合较强，跨打包器/框架协调成本高。
  - SystemJS（兼容遗留）：
    - 适合旧浏览器/遗留系统过渡期。
- 服务发现与注册中心
  - environment → channel（prod/staging/canary）→ index.json → per-remote manifest.json。
  - 支持版本协商、SRI 校验、健康检查、回退策略。
- 本地覆盖（Local Override）
  - Shell 可通过 query/localStorage/env 覆盖某 Remote 的 manifest.entry 指向本地 dev server（Vite/Webpack）。
  - 代理静态与 CORS 配置，保证同源或正确 CORS。

---

## 5. 隔离与集成

- JS 隔离
  - 默认“软隔离”：每个 Remote 独立运行域，避免修改 window 全局。
  - 可选“硬隔离”：iframe（针对不受信或第三方 Remote）。
- CSS 隔离
  - 优先 **Shadow DOM** 包裹 Remote/Component 根节点，避免样式串扰。
  - 统一 Design Tokens（CSS Variables）用于主题穿透；禁止全局 reset 污染。
- 依赖共享策略
  - 默认隔离各自依赖版本，避免破窗效应。
  - 白名单共享（如 React/Vue）：
    - 仅当版本 semver 兼容且标记 singleton 才共享；否则独立打包。
- 资源与基路径
  - 规范 `assetsBase`、`publicPath`；Webpack 可用 `__webpack_public_path__`，Vite 用 `import.meta.env.BASE_URL`。
- 安全
  - CSP、SRI、Integrity 校验、域隔离、权限策略（Permission Policy）。
  - 非信任 Remote 使用 iframe + sandbox + postMessage。

---

## 6. 路由与导航

- Shell 作为“顶层路由所有者”，Remote 提交路由声明（routes）。
- 路由挂载点：Shell 将匹配到的路由委派给对应 Remote 的 `mountRoute`。
- 路由隔离
  - Remote 内部可使用自身 Router，但需避免直接调用 `history.pushState` 破坏全局；通过 Shell Router API 发起导航。
- 导航协议
  - `router.navigate(to, options)`、`router.block(handler)`、`router.onChange(listener)`。
- SEO/可分享链接
  - CSR 为主；如需 SSR，采用“分布式 SSR”（各 Remote 自渲染，Shell 聚合）或 Islands（复杂度较高，非首期）。

---

## 7. 通信与状态

- 事件总线（轻量）
  - `eventBus.emit(topic, payload) / on / off`；主题命名空间化（如 `billing.invoice.created`）。
- 跨框架共享状态（谨慎）
  - 避免“全局单一 store”；使用“契约化读写 API”或只读查询服务。
- 跨 iframe 通信
  - `postMessage` + origin 校验 + schema 校验。
- 会话与认证
  - Token 传播（Header/Storage）、刷新与失效策略、CSRF 防护（SameSite/Lax/Strict）。

---

## 8. 性能策略

- 资源提示：`preload`/`prefetch`、`preconnect`、HTTP/2 push（或 103 Early Hints）。
- 代码切分：按路由与组件粒度；Remote 暴露时避免过度打包。
- 缓存：CDN 缓存键包含版本，静态资源长期缓存；HTML/manifest 短缓存。
- 并行：多个 Remote 并发加载（有上限控制）。
- 交互优化：骨架屏、流畅卸载/挂载、长任务切片（scheduler/requestIdleCallback）。
- 内存：严格执行 `unmount` 清理定时器、订阅、portal、全局监听。

---

## 9. 安全与合规

- CSP：`default-src 'self'; script-src 'self' https: 'strict-dynamic' 'nonce-...'; style-src 'self' 'unsafe-inline';`（视情况）
- SRI：对 Remote 入口与关键资源启用 `integrity`。
- Trusted Types/DOMPurify：对富文本/HTML 片段进行净化。
- 供应链：锁定依赖、SBOM、签名发布、镜像仓库。
- 权限：禁止 Remote 申请高危权限（如 clipboard-write）除非白名单。

---

## 10. 开发体验与工具

- CLI
  - `create-remote --framework vue3`
  - `dev --remote billing-vue --override http://localhost:5173`
  - `publish --channel canary`
- 开发服务器
  - 代理与 CORS 统一；HMR；Manifest 热更新。
- 调试叠加层（DevOverlay）
  - 显示当前挂载的 Remote/版本/路由；一键重载/切版本。
- 代码规范
  - ESLint/TS/Prettier；A11y/Lighthouse 预算检查。
- 模板工程
  - React/Vue/Angular 模板预置各自 Adapter 与打包配置。

---

## 11. 构建与发布

- 输出格式
  - ESM 单文件入口（推荐），或 MF remoteEntry.js。
- 打包器建议
  - Vite（库模式）/Rspack/Webpack/Rollup 均可，要求产物为 ESM 且显式导出 `createRemote()` 或默认导出 RemoteContract。
- CI/CD
  - 构建 → 单元/合约测试 → 产物签名 → 上传 CDN → 更新 registry → 金丝雀 → 全量 → 监控回滚。
- 版本治理
  - 语义化版本；兼容矩阵（Shell×Adapters×Remotes）。

---

## 12. 监控与可观测性

- 性能：FP/FCP/LCP/INP/CLS、自定义 marks（mount/unmount/prefetch）
- 错误：Remote 级 Error Boundary，上报（错误码+remoteId+version+route）
- 日志与追踪：trace-id 跨 Remote 传播，点击流、功能曝光。
- 健康：Manifest 心跳、入口加载失败重试、自动降级（占位/备用版本）。

---

## 13. 测试策略

- 合约测试（Contract Test）
  - 校验 Remote 导出符合协议；mount/unmount 幂等性；事件主题合规。
- 端到端（E2E）
  - Shell 驱动多 Remote 场景；路由切换、降级、回滚。
- 视觉回归（Visual Regression）
  - Storybook/Backstop 差异检测（在 Shadow DOM 下也稳定）。
- 性能回归
  - Bundle 体积预算、LCP/INP 阈值报警。

---

## 14. 参考实现示例

### 14.1 Shell Loader（ESM）

```ts
async function loadRemoteByManifest(manifestUrl: string, ctx: RuntimeContext): Promise<RemoteContract> {
  const manifest = await fetch(manifestUrl, { credentials: 'omit' }).then(r => r.json());
  // 校验 SRI（可选）
  if (manifest.integrity) {
    // 可结合 <link rel="modulepreload"> + SRI，或在 CDN 层校验
  }
  const mod = await import(/* @vite-ignore */ manifest.entry); // 需 CORS 允许
  const remote: RemoteContract =
    (mod.default && typeof mod.default === 'function') ? await mod.default(ctx) :
    (mod.createRemote ? await mod.createRemote(ctx) :
    (mod.remote ?? mod));
  return remote;
}
```

### 14.2 Remote 导出（框架无关）

```ts
// entry.ts
import { createVueRemote } from './frameworks/vue-adapter';

export default async function createRemote(ctx) {
  const app = await createVueRemote(ctx);
  const remote: RemoteContract = {
    id: 'billing-vue',
    version: '1.4.3',
    framework: 'vue3',
    routes: [
      { path: '/billing', expose: 'pages/BillingHome' },
      { path: '/billing/invoices/:id', expose: 'pages/InvoiceDetail' }
    ],
    async getComponent(name: string) {
      if (name === 'InvoiceCard') {
        const mod = await import('./components/InvoiceCard.vue');
        return ctx.frameworkAdapters['vue3'].wrapComponent(mod.default);
      }
      throw new Error(`Component not found: ${name}`);
    },
    async mountRoute(routePath, el, params, props, ctx2) {
      const pageExpose = routePath.includes('/invoices/')
        ? 'pages/InvoiceDetail'
        : 'pages/BillingHome';
      const mod = await import(`./${pageExpose}.vue`);
      const comp = ctx.frameworkAdapters['vue3'].wrapComponent(mod.default);
      return comp.mount(el, { ...props, params }, ctx2);
    },
    async prefetch() {
      await Promise.all([
        import('./pages/BillingHome.vue'),
        import('./components/InvoiceCard.vue')
      ]);
    }
  };
  return remote;
}
```

### 14.3 框架适配层示例

- React

```ts
// adapters/react18.ts
import React from 'react';
import { createRoot, Root } from 'react-dom/client';

export function wrapReactComponent(Component: React.ComponentType<any>) {
  let root: Root | null = null;
  let lastProps: any = null;

  const mount = (el: HTMLElement, props?: any) => {
    root = createRoot(el);
    lastProps = props ?? {};
    root.render(React.createElement(Component, lastProps));
    return () => {
      root?.unmount();
      root = null;
    };
  };

  const update = (props?: any) => {
    if (root) {
      lastProps = { ...lastProps, ...(props ?? {}) };
      root.render(React.createElement(Component, lastProps));
    }
  };

  return { mount, update };
}
```

- Vue 3

```ts
// adapters/vue3.ts
import { createApp, h } from 'vue';

export function wrapVueComponent(Component: any) {
  let app: any = null;
  let vm: any = null;
  let lastProps: any = null;

  const mount = (el: HTMLElement, props?: any) => {
    lastProps = props ?? {};
    app = createApp({ render: () => h(Component, lastProps) });
    vm = app.mount(el);
    return () => {
      app?.unmount();
      app = null; vm = null;
    };
  };

  const update = (props?: any) => {
    lastProps = { ...lastProps, ...(props ?? {}) };
    if (vm) vm.$forceUpdate?.();
  };

  return { mount, update };
}
```

- Angular（两种思路）
  - Angular Elements：将组件打包为自定义元素（Web Components），在运行时通过 `document.createElement('x-...')` 挂载。
  - 子应用 bootstrap：为每个 Remote 提供 `bootstrapModule` 到提供的容器元素，作为页面级别挂载。需要处理 Zone.js 与多实例。
  
示例（Elements 路径）：

```ts
// adapters/angular-elements.ts
export function wrapAngularElement(tagName: string) {
  let el: HTMLElement | null = null;
  return {
    mount: (container: HTMLElement, props?: any) => {
      el = document.createElement(tagName);
      Object.assign(el, props ?? {}); // 或使用属性/事件
      container.appendChild(el);
      return () => {
        el?.remove();
        el = null;
      };
    },
    update: (props?: any) => {
      if (el && props) Object.assign(el, props);
    }
  };
}
```

### 14.4 Shadow DOM 包装器（隔离样式）

```ts
export function withShadowRoot(host: HTMLElement, mode: ShadowRootMode = 'open') {
  const shadow = host.shadowRoot ?? host.attachShadow({ mode });
  return shadow;
}

// 使用时：const root = withShadowRoot(container); comp.mount(root as any, props, ctx);
```

### 14.5 事件总线

```ts
type Handler = (payload: any) => void;
const map = new Map<string, Set<Handler>>();

export const eventBus = {
  on(topic: string, h: Handler) {
    if (!map.has(topic)) map.set(topic, new Set());
    map.get(topic)!.add(h);
    return () => map.get(topic)!.delete(h);
  },
  emit(topic: string, payload?: any) {
    map.get(topic)?.forEach(h => h(payload));
  }
};
```

---

## 15. 你可能还没考虑到的问题与风险

- 依赖冲突与升级地狱
  - 同页多版本 React/Vue/Angular 的体积与内存成本大；建议共享白名单+严格版本策略。
- 样式与主题
  - 第三方 UI 库的全局样式泄漏；z-index、Portal/Overlay（Modal/Tooltip）跨 Root 冲突；建议统一 Overlay 宿主。
- 路由争用
  - Remote 内部直接调用 `history` 导致 URL 状态不一致；统一通过 Shell Router。
- Angular 的 Zone.js
  - 多实例下性能与副作用；可评估 `ngZone: 'noop'` 搭配手动变更检测或使用 Elements。
- 图标与字体
  - 多份 iconfont/字体重复加载；建议统一 Icon 方案或使用 SVG sprite。
- SSR/SEO/打印
  - 多框架 SSR 聚合复杂；首期建议 CSR；需要打印时注意 Shadow DOM 内容打印策略。
- 无障碍与国际化
  - 多 Remote 一致性难；需统一 A11y 规范与 i18n 上下文（语言、时区、货币）。
- Service Worker
  - 多 Remote 重叠 SW 作用域风险；建议统一在 Shell 层注册 SW。
- 安全
  - 第三方 Remote 的供应链风险、XSS、postMessage 注入；必须 origin 白名单+schema 校验+SRI。
- 性能与泄漏
  - 频繁 mount/unmount 的内存泄漏（事件/定时器/observer 未清理）；需要合约测试兜底。
- 打印/截图/导出
  - Shadow DOM 内容导出需特殊处理（开放光影 DOM 或复制到临时层）。

---

## 16. 建设路线图（建议）

- Phase 0：PoC
  - 核心契约、ESM Loader、Vue3 与 React18 适配器、Shadow DOM 隔离、简单路由委派、事件总线。
- Phase 1：开发体验与发布
  - CLI、本地覆盖、模板工程、Registry、CI/CD、SRI、监控埋点。
- Phase 2：规模化
  - 版本治理、灰度/金丝雀、回滚、性能预算、错误边界、兼容矩阵。
- Phase 3：增强
  - iframe 安全隔离选项、可插拔权限/数据访问层、（可选）分布式 SSR、A11y 标准化。

里程碑验收：
- M1：单页同时挂载 Vue+React Remote，并可路由切换与通信。
- M2：远程加载与本地覆盖可切换；出包规范化；监控上线。
- M3：灰度与回滚闭环；性能预算达标；合约测试覆盖>90%。

---

## 17. 决策建议（默认方案）

- 加载：ESM 动态 import + CDN；必要时对接 Module Federation。
- 隔离：Shadow DOM（默认）+ 软隔离；第三方 Remote 用 iframe。
- 依赖：默认不共享，白名单共享 React/Vue 等，需严格 semver。
- 路由：Shell 统一路由，Remote 声明路由并通过 API 导航。
- 通信：事件总线（轻量）+ 类型约束；跨 iframe 用 postMessage。
- 产物：每个 Remote 暴露 `default export createRemote(ctx)`，返回 RemoteContract。
- DX：本地覆盖、模板、DevOverlay、合约测试、CI/CD 签名+SRI。

---

## 18. 下一步可以落地的最小任务清单（1-2 周）

- 定义并发布 v0 契约（本文 3.1/3.3）。
- 实现 ESM Loader 与 Registry（支持本地覆盖）。
- 提供 Vue3/React18 适配器与示例 Remote（每种 1 个页面+1 个组件）。
- Shell 路由委派与事件总线。
- Shadow DOM 容器与样式基线（Design Tokens）。
- CI 构建上传 CDN 与 registry 更新脚本。
- 基础监控（FCP/LCP/错误上报）与 DevOverlay。

