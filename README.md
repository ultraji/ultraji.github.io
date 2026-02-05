# ultrajiのblog

## 特性

- ⚡️ 使用 Astro 5.x 构建，性能优异
- 📝 支持 Markdown 和 MDX
- 🎨 保留原博客风格
- 🌐 支持 GitHub Pages 部署
- 📱 响应式设计
- 🔍 SEO 友好
- 🚀 现代化的开发体验

## 项目结构

```
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Pages 部署配置
├── public/                     # 静态资源
├── src/
│   ├── assets/                # 图片等资源
│   ├── components/            # Astro 组件
│   │   ├── BaseHead.astro
│   │   ├── Footer.astro       # 页脚
│   │   ├── Header.astro       # 导航栏
│   │   └── ...
│   ├── content/
│   │   └── blog/              # 博客文章
│   │       └── ...
│   ├── layouts/               # 页面布局
│   ├── pages/                 # 页面路由
│   │   ├── index.astro        # 首页
│   │   ├── blog/              # 博客列表
│   │   └── about.astro        # 关于页面
│   ├── styles/                # 样式文件
│   ├── consts.ts              # 全局常量配置
│   └── content.config.ts      # Content Collections 配置
├── astro.config.mjs           # Astro 配置
├── package.json
└── README.md
```

## 快速开始

### 前置要求

- Node.js >= 18.20.8 (推荐使用 Node.js 20+)
- npm 或 yarn

### 安装 Node.js (如需要)

#### 使用 nvm (推荐)

```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# 安装 Node.js 20
nvm install 20
nvm use 20

# 验证版本
node --version  # 应该显示 v20.x.x
```

#### 或使用系统包管理器

```bash
# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 验证版本
node --version
```

### 开发流程

#### 1. 安装依赖

```bash
npm install
```

#### 2. 启动开发服务器

```bash
npm run dev
```

访问 http://localhost:4321

#### 3. 编写内容

在 `src/content/blog/` 目录下创建或编辑 Markdown 文件。

#### 4. 预览构建

```bash
npm run build
npm run preview
```

### 常用命令

```bash
# 开发
npm run dev              # 启动开发服务器

# 构建
npm run build           # 构建生产版本
npm run preview         # 预览生产版本

# 其他
npm run astro           # 运行 Astro CLI
npm run astro -- --help # 查看帮助
```

## 部署到 GitHub Pages

本项目已配置 GitHub Actions 自动部署。

### 配置步骤

1. 在 GitHub 仓库设置中，进入 Settings > Pages
2. Source 选择 "GitHub Actions"
3. 推送代码到 `main` 或 `master` 分支
4. GitHub Actions 会自动构建并部署

### 自定义域名

1. 在 `public/` 目录下创建 `CNAME` 文件，内容为你的域名
2. 在域名提供商处配置 DNS 解析

## 配置说明

### 网站基本信息

编辑 `src/consts.ts` 修改网站信息：

```typescript
export const SITE_TITLE = 'ultrajiのblog';
export const SITE_DESCRIPTION = '操千曲而后晓声，观千剑而后识器';
export const SITE_AUTHOR = 'Jack Wang';
export const SITE_EMAIL = 'ultraji@live.com';
export const GITHUB_USERNAME = 'ultraji';
export const ICP_NUMBER = '浙ICP备2026007268号';
export const ICP_URL = 'https://beian.miit.gov.cn/';
```

### 网站URL

编辑 `astro.config.mjs` 中的 `site` 配置：

```javascript
export default defineConfig({
  site: 'https://ultraji.xyz',
  // ...
});
```

## 博客文章

博客文章位于 `src/content/blog/` 目录下。

### 创建新文章

在 `src/content/blog/` 目录下创建 `.md` 或 `.mdx` 文件：

```markdown
---
title: '文章标题'
description: '文章描述'
pubDate: 2024-02-04
tags:
  - docker
  - 容器
---

文章内容...
```

### Front Matter 字段

- `title` (必需): 文章标题
- `postId`(必需): 六位数字（年月序号），对应文字URL地址
- `description` (可选): 文章描述
- `pubDate` (可选): 发布日期
- `updatedDate` (可选): 更新日期
- `heroImage` (可选): 封面图片
- `tags` (可选): 标签

## 技术栈

- [Astro](https://astro.build/) - 静态站点生成器
- [MDX](https://mdxjs.com/) - Markdown 增强
- TypeScript - 类型安全
- GitHub Actions - CI/CD

## 联系方式

- Email: ultraji@live.com
- GitHub: [@ultraji](https://github.com/ultraji)

## 更多资源

- [Astro 文档](https://docs.astro.build/)
- [Astro 示例](https://astro.build/themes/)
- [Markdown 语法](https://www.markdownguide.org/)
- [GitHub Pages 文档](https://docs.github.com/pages)
