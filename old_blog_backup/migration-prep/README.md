# Blog Migration Preparation

本文件夹包含从当前 Fuwariblog 主题迁移到新 Astro 主题所需的所有资料。

## 目录结构

```
migration-prep/
├── 01-content/               ← 所有文章（直接从旧项目复制）
│   ├── posts/
│   │   ├── lecture-notes-zh/   # CS110 讲义（24 篇）
│   │   ├── slides-zh/          # CS110 课件（20 篇）
│   │   ├── translated_markdown/# CS106L 笔记（17 篇）
│   │   └── xv6-riscv-book/     # xv6 OS 书籍（9 章）
│   └── spec/
│       └── about.md            # 关于页面
│
├── 02-assets/                ← 图片和静态资源
│   ├── images/                 # 文章内图片
│   └── favicon/                # 网站图标
│
├── 03-config/                ← 个人配置（直接复用）
│   ├── config.ts               # 站点名称、个人信息、导航栏、主题色
│   └── musicConfig.ts          # 音乐播放器配置
│
├── 04-plugins/               ← 自定义插件（必须保留）
│   ├── expressive-code/        # 代码块复制按钮、语言标识徽章
│   ├── rehype-component-admonition.mjs  # :::tip :::warning 提示框
│   ├── rehype-component-github-card.mjs # GitHub 仓库卡片
│   ├── remark-directive-rehype.js       # ::: 指令解析器
│   ├── remark-excerpt.js                # 文章摘要提取
│   └── remark-reading-time.mjs          # 阅读时间估算
│
├── 05-custom-components/     ← 自定义组件（非主题自带）
│   ├── MusicManager.astro      # 音乐播放器管理器
│   ├── Music.astro             # 音乐播放器 UI
│   ├── Calendar.astro          # 日历组件
│   └── ArchivePanel.svelte     # 归档面板
│
├── 06-reference/             ← 参考文件（理解旧架构用）
│   ├── types/                  # TypeScript 类型定义
│   ├── constants/              # 主题常量
│   ├── utils/                  # 工具函数
│   ├── i18n/                   # 多语言翻译
│   ├── styles/                 # 旧主题样式表
│   ├── layouts/                # 旧布局组件
│   ├── pages/                  # 旧页面路由
│   └── components/             # 旧主题组件
│
└── 07-project-config/        ← 项目配置文件
    ├── package.json            # 依赖清单
    ├── astro.config.mjs        # Astro 配置（集成、插件、Markdown）
    ├── tailwind.config.cjs     # Tailwind CSS 配置
    ├── tsconfig.json           # TypeScript 配置
    └── pnpm-lock.yaml          # 依赖锁定文件
```

## 迁移步骤

### Step 1: 创建新 Astro 项目
```bash
pnpm create astro@latest
```

### Step 2: 安装必要依赖
参考 `07-project-config/package.json` 中的 dependencies，关键依赖：

| 类别 | 包名 | 用途 |
|------|------|------|
| Markdown | `remark-directive`, `remark-math`, `rehype-katex`, `rehype-slug`, `rehype-autolink-headings` | ::: 指令、数学公式、标题链接 |
| 代码高亮 | `astro-expressive-code`, `@expressive-code/*` | 代码块语法高亮 |
| UI | `@astrojs/svelte`, `tailwindcss`, `@iconify/svelte` | 组件框架、样式、图标 |
| 功能 | `@swup/astro`, `photoswipe`, `pagefind`, `overlayscrollbars` | 页面过渡、图片灯箱、搜索、滚动条 |

### Step 3: 迁移内容
```bash
cp -r 01-content/posts/*  新项目/src/content/posts/
cp -r 01-content/spec/*   新项目/src/content/spec/
cp -r 02-assets/images/*  新项目/src/assets/images/
cp -r 02-assets/favicon/* 新项目/public/favicon/
```

### Step 4: 迁移插件
```bash
cp -r 04-plugins/* 新项目/src/plugins/
```
在 `astro.config.mjs` 中注册这些插件（参考 `07-project-config/astro.config.mjs`）。

### Step 5: 迁移配置
- `03-config/config.ts` → 合并到新主题的配置文件
- `03-config/musicConfig.ts` → 如果保留音乐功能则复制

### Step 6: 迁移自定义组件
- `05-custom-components/` → 放到新项目的 `src/components/` 下
- 根据需要调整路径引用

### Step 7: 适配样式
- `06-reference/styles/` 中的自定义样式（markdown.css, main.css 等）
- 将自定义样式合并到新主题的样式系统中

## 注意事项

1. **:::tip 指令格式**: 所有文章中的 `:::tip` 必须写成 `:::tip`（无空格），否则不渲染
2. **图片路径**: 文章中的图片引用使用相对路径 `(image/xxx.png)`
3. **YAML frontmatter**: 标题含冒号的必须加引号 `title: "第1章：标题"`
4. **One Dark Pro**: 代码块主题设置为 `one-dark-pro`
5. **内容集合**: 参考 `07-project-config/tsconfig.json` 中的路径别名配置
