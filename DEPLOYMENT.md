# GitHub Pages 部署说明

## 概述

本项目已配置为部署到 GitHub Pages，可通过以下 URL 访问：
**https://kewei-cpu.github.io/**

## 部署配置

### 1. GitHub Actions 工作流

已创建 `.github/workflows/deploy.yml` 文件，包含以下功能：
- 自动在 `main` 分支推送时触发
- 安装 Node.js 20
- 安装项目依赖
- 运行 TypeScript 类型检查
- 构建生产版本
- 部署到 GitHub Pages

### 2. Vite 配置

`vite.config.ts` 已配置：
```typescript
base: '/'
```

### 3. 构建脚本

`package.json` 中的构建命令：
```bash
npm run build
```

该命令会：
1. 运行 TypeScript 类型检查
2. 构建 Vite 应用
3. 复制 `.nojekyll` 文件（防止 Jekyll 处理）

## 部署步骤

### 首次部署

1. **创建 GitHub 仓库**
   - 仓库名称：`Kewei-cpu.github.io`
   - 设置为 Public

2. **推送代码到 GitHub**
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/Kewei-cpu/Kewei-cpu.github.io.git
   git push -u origin main
   ```

3. **启用 GitHub Pages**
   - 进入仓库 Settings > Pages
   - Source 选择 "GitHub Actions"
   - 无需其他配置

4. **等待部署完成**
   - GitHub Actions 会自动运行
   - 部署完成后访问：https://kewei-cpu.github.io/

### 后续更新

只需将代码推送到 `main` 分支：
```bash
git add .
git commit -m "Update"
git push
```

GitHub Actions 会自动重新部署。

## 文件结构

```
.github/
└── workflows/
    └── deploy.yml          # GitHub Actions 工作流

public/
├── .nojekyll               # 防止 Jekyll 处理
└── emojis.json            # Emoji 数据

vite.config.ts             # Vite 配置（包含 base: '/'）
```

## 注意事项

1. **仓库名称**：必须为 `Kewei-cpu.github.io` 才能使用 GitHub Pages 的特殊域名
2. **分支**：部署基于 `main` 分支
3. **等待时间**：首次部署可能需要几分钟时间
4. **自定义域名**：如需使用自定义域名，需要在仓库根目录创建 `CNAME` 文件

## 故障排除

### 构建失败
- 检查 Actions 日志
- 确保所有依赖正确安装
- 检查 TypeScript 类型错误

### 页面空白
- 检查浏览器控制台错误
- 确认 `base: '/'` 配置正确
- 检查资源文件路径

### 404 错误
- 确认 GitHub Pages 设置中 Source 为 "GitHub Actions"
- 等待构建完成（可能需要几分钟）
