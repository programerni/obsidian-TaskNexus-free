# TaskNexus Free v1.1.0
日期：2026-09-14

> 重构 README 支持中英文切换；移除免费版中的规划视图功能；补充 Pro 版介绍与联系方式。

## 变更

- 移除「规划视图」功能（`config/features.json` 中 `plan.free` 改为 `false`），免费版不再包含规划视图
- README.md 重写为中英文双语版本，顶部提供语言切换链接
- 补充 Pro 版功能详细介绍（20+ 项高级功能逐一说明）
- 补充联系方式：微信二维码、小红书地址、官网链接
- 更新 features.json：功能列表与 v1.1.0 一致（9 个功能模块）
- 代码裁剪：构建时自动移除规划视图相关代码

## 免费版功能（v1.1.0）

核心甘特图、快速添加、表格列展示配置、局部排产、变更记录、激活管理、快捷键、首页仪表盘、想法收集

## 验证

- 构建通过：`node scripts/build-editions.mjs --edition free --version 1.1.0`
- 产物目录：`dist/free/1.1.0/`（main.js 1429.6KB, styles.css 306.1KB, manifest.json version 1.1.0）
- features.json 确认不含 `plan`（规划视图）
- manifest.json description 确认不含「规划视图」

# TaskNexus Free v1.0.9
日期：2026-09-13

> 初始公开版本：GitHub 仓库创建 + Release 发布

## 变更

- 创建 GitHub 仓库 `programerni/obsidian-tasknexus-free`
- 推送免费版代码：main.js + styles.css + manifest.json + features.json + versions.json + CHANGELOG.md
- 编写 README.md（英文版）
- 创建 GitHub Release v1.0.9（含 zip 安装包）
