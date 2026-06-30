# openEuler 网站语义化标签优化方案

## 一、现状诊断（数据来自实际抓取）

| 维度 | 现状 | 问题严重度 |
|------|------|-----------|
| 语义标签 | 仅 `header`×1、`main`×1，无 `nav`/`article`/`section`/`aside`/`footer` | 严重 |
| 标题层级 | h2×5、h3×1、h4×5，**无 h1**，h2→h4 跳级 | 严重 |
| div 滥用 | 270 个 div，核心内容区全用 div 包裹 | 严重 |
| ARIA | 仅 5 个 `role=separator`，无 `aria-label` | 中等 |
| 图片 alt | 45 张图仅 20 张有 alt，25 张缺失 | 中等 |
| Schema | 2 块 JSON-LD（Organization + SoftwareApplication），**内容完全重复** | 轻微 |
| SSR | SSR 可见文本仅 1304 字符，大量内容依赖 JS 渲染 | 中等 |

## 二、核心问题拆解

### 问题1：无 h1，标题层级断裂

```
当前：h2(面向数字基础设施的开源操作系统) → h4(技术白皮书/安全中心/迁移专区/活动专区)
      h2(开发者日历) → 无子标题
      h2(用户案例) → 无子标题
      h2(社区动态) → 无子标题
```

AI 爬虫依赖 h1 判断页面主题，h2→h4 跳级导致 AI 无法判断层级关系。

### 问题2：div-soup，零 section/article

首页 6 大内容区（Banner、快速入口、特性介绍、日历、案例、动态）全部用 `<div>` 堆叠，AI 无法区分"这是导航区""这是主内容""这是侧边栏"。

### 问题3：25 张图片缺 alt

Banner 图、特性图标、案例图等均无 alt，AI 无法理解图片内容。

## 三、优化方案（按优先级排序）

### P0：标题层级修复（1天）

```html
<!-- 修复前 -->
<div class="home-banner">
  <div class="banner-title">面向数字基础设施的开源操作系统</div>
</div>
<div class="home-quick-entry">
  <h4>技术白皮书</h4>
  <h4>安全中心</h4>
</div>

<!-- 修复后 -->
<section class="home-banner" aria-label="首页横幅">
  <h1>面向数字基础设施的开源操作系统</h1>
</section>
<nav class="home-quick-entry" aria-label="快速入口">
  <h2>快速入口</h2>
  <ul>
    <li><a href="...">技术白皮书</a></li>
    <li><a href="...">安全中心</a></li>
  </ul>
</nav>
```

**规则**：
- 全站有且仅有 1 个 h1，为页面核心主题
- h1 → h2 → h3 逐级递进，禁止跳级
- 快速入口/功能入口用 `nav` + h2

### P1：语义标签替换 div（2-3天）

按内容区逐一替换：

| 内容区 | 当前 | 优化为 | 理由 |
|--------|------|--------|------|
| 顶部导航 | `<header>` + div | `<header>` + `<nav aria-label="主导航">` | 导航区语义化 |
| Banner | `<div class="home-banner">` | `<section aria-label="首页横幅">` | 独立内容块 |
| 快速入口 | `<div>` + `<h4>` | `<nav aria-label="快速入口">` + `<h2>` | 导航型内容 |
| 特性介绍 | `<div>` | `<section aria-label="核心特性">` + `<article>` 每个特性 | 独立内容块 |
| 开发者日历 | `<div>` | `<section aria-label="开发者日历">` | 独立内容块 |
| 用户案例 | `<div>` | `<section aria-label="用户案例">` + `<article>` 每个案例 | 每个案例是独立条目 |
| 社区动态 | `<div>` | `<section aria-label="社区动态">` | 独立内容块 |
| 友好社区 | `<div>` | `<aside aria-label="友好社区">` | 辅助/关联内容 |
| 底部 | 无 footer | `<footer>` | 页脚语义 |
| 侧边栏/推荐 | `<div>` | `<aside>` | 辅助内容 |

**VitePress 组件改造要点**：
- 修改 `TheHome.vue` 组件，将外层 div 改为 section
- 每个 Feature Card 用 `<article>` 包裹
- 日历/动态列表项用 `<article>` 包裹

### P2：图片 alt 补全（1天）

```html
<!-- 修复前 -->
<img src="/assets/pc.EU0VZA2W.jpg" class="o-figure-img">

<!-- 修复后：有信息量的图片 -->
<img src="/assets/pc.EU0VZA2W.jpg" class="o-figure-img"
     alt="openEuler 2025社区年报横幅">

<!-- 装饰性图片 -->
<img src="data:image/png;base64,..." class="circle" alt="" role="presentation">
```

**规则**：
- 有信息量的图片：写具体描述（Banner/案例/特性图）
- 图标类 SVG：`alt="功能名称"` 或 `aria-hidden="true"`
- 纯装饰：`alt=""` + `role="presentation"`

### P3：ARIA 增强（1天）

```html
<!-- 导航 -->
<nav aria-label="主导航">
<nav aria-label="快速入口">
<nav aria-label="面包屑" aria-describedby="breadcrumb-desc">

<!-- 内容区 -->
<section aria-label="核心特性">
<section aria-label="开发者日历">
<article aria-label="openEuler 24.03 LTS SP3 发布">

<!-- 轮播 -->
<div class="o-carousel" role="region" aria-label="首页轮播" aria-roledescription="轮播">

<!-- 按钮/链接 -->
<a href="/zh/download/" aria-label="下载 openEuler 操作系统">
```

### P4：Schema 去重 + 补充（0.5天）

当前问题：两块 JSON-LD 内容完全重复。优化为保留1块并补充 WebSite + SearchAction：

```json
[
  {
    "@type": "WebSite",
    "name": "openEuler社区官网",
    "url": "https://www.openeuler.openatom.cn/zh/",
    "potentialAction": {
      "@type": "SearchAction",
      "target": "https://www.openeuler.openatom.cn/zh/search/?q={search_term_string}",
      "query-input": "required name=search_term_string"
    }
  },
  {
    "@type": "Organization",
    "name": "openEuler社区",
    "alternateName": "openEuler",
    "url": "https://www.openeuler.openatom.cn/zh/",
    "logo": "https://www.openeuler.openatom.cn/img/for-cdn/logo.png",
    "description": "openEuler是一个开源、免费的 Linux 发行版平台，通过开放的形式与全球的开发者共同构建一个开放、多元和架构包容的软件生态体系。",
    "contactPoint": {
      "@type": "ContactPoint",
      "email": "contact@openeuler.io",
      "contactType": "customer service"
    },
    "parentOrganization": {
      "@type": "Organization",
      "name": "开放原子开源基金会",
      "alternateName": "OpenAtom Foundation"
    }
  },
  {
    "@type": "SoftwareApplication",
    "name": "openEuler",
    "applicationCategory": "Operating System",
    "operatingSystem": "Linux",
    "offers": {
      "@type": "Offer",
      "price": "0",
      "priceCurrency": "CNY",
      "description": "openEuler社区版完全免费开源"
    },
    "description": "面向数字基础设施的开源操作系统，支持服务器、云计算、边缘计算、嵌入式四大场景，支持ARM、x86、RISC-V、LoongArch、PowerPC、SW-64六种处理器架构。",
    "softwareVersion": "24.03 LTS SP3",
    "downloadUrl": "https://www.openeuler.openatom.cn/zh/download/",
    "license": "https://license.coscl.org.cn/MulanPSL2"
  }
]
```

## 四、落地节奏

| 阶段 | 内容 | 产出 |
|------|------|------|
| 第1周 | P0 标题层级 + P1 语义标签 | 首页 DOM 改造完成 |
| 第2周 | P2 alt 补全 + P3 ARIA | 全站 alt/ARIA 合规 |
| 第3周 | P4 Schema + 内页推广 | Schema 去重补充，核心内页同步改造 |
| 第4周 | 验收 | Lighthouse 语义分≥95、h 层级零跳级、alt 覆盖率 100% |

## 五、验收标准

| 指标 | 目标 | 度量方式 |
|------|------|---------|
| h1 唯一性 | 每页有且仅有 1 个 h1 | 脚本扫描 |
| h 层级 | 零跳级 | Lighthouse + 自定义脚本 |
| 语义标签 | nav/article/section/aside/footer 覆盖所有内容区 | 代码审查 |
| alt 覆盖率 | 100% | 脚本扫描 |
| ARIA | 所有交互组件有 aria-label | axe-core 审计 |
| Lighthouse 语义分 | ≥95 | Lighthouse CI |

## 六、VitePress 层面关键改造点

openEuler 基于 VitePress v1.6.4，语义化改造需关注：

1. **主题组件**：修改 `TheHome.vue`，将 div 骨架替换为语义标签
2. **布局组件**：`Layout.vue` 中补 `<nav>`、`<footer>`
3. **Markdown 渲染**：确保文档页自动生成 h1（来自 frontmatter title）
4. **构建钩子**：在 `config.ts` 中用 `transformHead` 补 h1
5. **SSR 输出**：确认 VitePress SSG 模式下语义标签正确输出到静态 HTML
