# 🎯 Emoji Guess

一个有趣的 Emoji 猜测游戏，使用 Vue 3 + Vite + TailwindCSS 构建。

## 功能特性

### 功能1：随机猜测
- 随机展示一个 Unicode emoji
- 显示对应字符数量的输入格子
- 输入 emoji 的英文名字（shortname）
- 自动过滤掉有肤色修饰符的变体

### 功能2：字母过滤猜测
- 用户选择一个字母
- 随机展示以该字母开头的 emoji
- 猜测 emoji 的名字
- 显示可用 emoji 数量

### 功能3：搜索功能
- 通过名字搜索 emoji
- 支持模糊搜索
- 点击 emoji 查看详细信息

### 功能4：全部展示
- 展示所有可用 emoji
- 按分类筛选
- 点击查看详细信息

## 技术栈

- **前端框架**: Vue 3 (Composition API)
- **构建工具**: Vite
- **类型检查**: TypeScript
- **样式**: TailwindCSS
- **部署**: GitLab Pages

## 项目设置

### 安装依赖

```bash
npm install
```

### 开发模式

```bash
npm run dev
```

### 类型检查

```bash
npm run type-check
```

### 构建生产版本

```bash
npm run build
```

### 预览生产版本

```bash
npm run preview
```

### 运行测试

```bash
npm run test:unit
```

### 代码规范检查

```bash
npm run lint
```

## 部署到 GitLab Pages

本项目已配置 GitLab CI/CD 流水线，可以自动部署到 GitLab Pages：

### 自动部署

1. 将代码推送到 GitLab 仓库
2. 确保在 `main` 分支
3. GitLab CI/CD 会自动：
   - 安装依赖
   - 运行类型检查
   - 构建应用
   - 部署到 GitLab Pages

### 访问应用

部署完成后，访问：
- `https://git.tsinghua.edu.cn/<username>/emoji-guess`

## 项目结构

```
emoji-guess/
├── public/
│   ├── emojis.json       # Emoji 数据文件
│   └── favicon.ico       # 网站图标
├── src/
│   ├── components/       # Vue 组件
│   │   ├── RandomGuess.vue
│   │   ├── LetterGuess.vue
│   │   ├── SearchEmoji.vue
│   │   └── AllEmojis.vue
│   ├── App.vue           # 主应用组件
│   └── main.ts           # 应用入口
├── .gitlab-ci.yml        # GitLab CI 配置
└── vite.config.ts        # Vite 配置
```

## 数据过滤

应用会自动过滤 emoji 数据：
- 移除所有带肤色修饰符的 emoji（如 👌🏻👌🏼👌🏽👌🏾👌🏿）
- 移除复杂的组合 emoji（如家庭组合 👨‍👩‍👧‍👦）
- 只保留基础 emoji，确保游戏体验流畅

## 开发说明

- 使用 Vue 3 Composition API
- 响应式设计，支持移动端
- 支持 TypeScript 类型检查
- 使用 TailwindCSS 进行样式设计
- 遵循 ESLint 代码规范

## 许可证

MIT
