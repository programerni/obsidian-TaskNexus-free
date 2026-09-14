# TaskNexus v1.0.9
日期：2026-09-13

> 新增（CT-20260819-10）：统计面板过渡版——图表按业务类型装进子 Tab，统一视觉/间距/颜色/图例/卡片层级。

## 变更

- `src/stats.ts`：统计面板按业务类型拆分为 3 个子 Tab（📋 项目总览 / 📈 任务统计 / 👥 人员统计），现有图表全部保留、仅重新分组：总览=指标卡片+项目任务统计+按文件夹统计+风险提醒；任务=近7天/近1月完成趋势+优先级分布+按月统计+月度趋势；人员=任务负载+完成率排行+未指派提醒
- `src/view.ts`：补全子 Tab 切换交互（点击 Tab 按钮切换对应面板，active 态高亮），不新增任何统计指标
- `src/styles.css`：统一卡片层级（圆角8px/内边距16px/轻微阴影/标题14px加粗带色条）、指标卡片数值20px、图例统一12px；统一语义色——已完成=绿、进行中·新建=蓝、待处理=灰蓝、逾期·风险=红、预警=橙、次要=灰；修复「新建」柱与图例颜色不一致（原橙色柱改蓝色）；人员负载超人均=橙色预警、有逾期=红色风险分色显示；优先级「低」由绿色改灰色（避免与已完成绿色语义冲突）；补全缺失的组件样式（子Tab栏、卡片容器、风险提醒卡片/芯片、完成率排行、空态、近1月曲线图）
- `manifest.json` / `versions.json` / `CHANGELOG.md`：版本升至 6.13.39

## 验证

- 构建通过：`node esbuild.config.mjs production`；`tsc --noEmit` 无本次改动新增类型错误（仓库既有错误与本次无关）
- UI 截图验证：`tools/stats-panel-ui/` 真实 styles.css 渲染统计面板，三个 Tab 逐一切换截图（DOM 断言：Tab 点击切换生效、非激活面板隐藏、卡片容器/图例/标题样式生效、无横向溢出、无重叠溢出），OCR 结构检查识别到子 Tab 文案与各图表标题
- 验证位置：测试库 `ob_plan/test_vault`（dev v6.13.39，main.js/styles.css/manifest.json shasum 与工作区构建产物一致）
- 用户验证步骤：打开测试库 → 重载插件（设置→第三方插件 关闭再开启 task-gantt，或重启 Obsidian）→ 任务视图「读取库」→ 切到「统计面板」Tab → 预期：顶部出现「📋 项目总览 / 📈 任务统计 / 👥 人员统计」子 Tab 栏，默认显示项目总览（4 个指标卡片+项目任务统计+按文件夹统计+风险提醒）；点击「📈 任务统计」/「👥 人员统计」可切换对应图表组，激活 Tab 蓝色高亮；各图表卡片统一圆角、间距、阴影，图例颜色与图形一致

# TaskNexus v1.0.6
日期：2026-09-13

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v1.0.8
日期：2026-09-13

- 功能数：10 → 10
- 无功能变化（仅代码/修复）

# TaskNexus v1.0.6
日期：2026-09-12

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v1.0.6
日期：2026-09-12

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v0.0.0-freev
日期：2026-09-03

- 功能数：5 → 10
- 新增：激活管理(activation)、快捷键(shortcuts)、首页仪表盘(dashboard)、想法收集(inspiration)、规划视图(plan)

# TaskNexus v0.0.0-test
日期：2026-09-03

- 功能数：5 → 10
- 新增：激活管理(activation)、快捷键(shortcuts)、首页仪表盘(dashboard)、想法收集(inspiration)、规划视图(plan)

# TaskNexus v1.0.6
日期：2026-09-01

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

- `src/stats.ts`：统计面板按业务类型拆分为 3 个子 Tab（📋 项目总览 / 📈 任务统计 / 👥 人员统计），现有图表全部保留、仅重新分组：总览=指标卡片+项目任务统计+按文件夹统计+风险提醒；任务=近7天/近1月完成趋势+优先级分布+按月统计+月度趋势；人员=任务负载+完成率排行+未指派提醒
- `src/view.ts`：补全子 Tab 切换交互（点击 Tab 按钮切换对应面板，active 态高亮），不新增任何统计指标
- `src/styles.css`：统一卡片层级（圆角8px/内边距16px/轻微阴影/标题14px加粗带色条）、指标卡片数值20px、图例统一12px；统一语义色——已完成=绿、进行中·新建=蓝、待处理=灰蓝、逾期·风险=红、预警=橙、次要=灰；修复「新建」柱与图例颜色不一致（原橙色柱改蓝色）；人员负载超人均=橙色预警、有逾期=红色风险分色显示；优先级「低」由绿色改灰色（避免与已完成绿色语义冲突）；补全缺失的组件样式（子Tab栏、卡片容器、风险提醒卡片/芯片、完成率排行、空态、近1月曲线图）
- `manifest.json` / `versions.json` / `CHANGELOG.md`：版本升至 6.13.39

## 验证

- 构建通过：`node esbuild.config.mjs production`；`tsc --noEmit` 无本次改动新增类型错误（仓库既有错误与本次无关）
- UI 截图验证：`tools/stats-panel-ui/` 真实 styles.css 渲染统计面板，三个 Tab 逐一切换截图（DOM 断言：Tab 点击切换生效、非激活面板隐藏、卡片容器/图例/标题样式生效、无横向溢出、无重叠溢出），OCR 结构检查识别到子 Tab 文案与各图表标题
- 验证位置：测试库 `ob_plan/test_vault`（dev v6.13.39，main.js/styles.css/manifest.json shasum 与工作区构建产物一致）
- 用户验证步骤：打开测试库 → 重载插件（设置→第三方插件 关闭再开启 task-gantt，或重启 Obsidian）→ 任务视图「读取库」→ 切到「统计面板」Tab → 预期：顶部出现「📋 项目总览 / 📈 任务统计 / 👥 人员统计」子 Tab 栏，默认显示项目总览（4 个指标卡片+项目任务统计+按文件夹统计+风险提醒）；点击「📈 任务统计」/「👥 人员统计」可切换对应图表组，激活 Tab 蓝色高亮；各图表卡片统一圆角、间距、阴影，图例颜色与图形一致

# TaskNexus v0.0.0-test
日期：2026-08-30

- 功能数：5 → 7
- 新增：激活管理(activation)、快捷键(shortcuts)

# TaskNexus v0.0.0-test
日期：2026-08-30

- 功能数：5 → 7
- 新增：激活管理(activation)、快捷键(shortcuts)

# TaskNexus v0.0.0-test
日期：2026-08-30

- 功能数：5 → 7
- 新增：激活管理(activation)、快捷键(shortcuts)

# TaskNexus v1.0.6
日期：2026-08-30

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v1.0.6
日期：2026-08-24

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v1.0.6
日期：2026-08-24

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v1.0.6
日期：2026-08-24

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v1.0.6
日期：2026-08-24

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v1.0.9
日期：2026-08-21

> 新增（CT-20260819-10）：统计面板过渡版——图表按业务类型装进子 Tab，统一视觉/间距/颜色/图例/卡片层级。

## 变更

- `src/stats.ts`：统计面板按业务类型拆分为 3 个子 Tab（📋 项目总览 / 📈 任务统计 / 👥 人员统计），现有图表全部保留、仅重新分组：总览=指标卡片+项目任务统计+按文件夹统计+风险提醒；任务=近7天/近1月完成趋势+优先级分布+按月统计+月度趋势；人员=任务负载+完成率排行+未指派提醒
- `src/view.ts`：补全子 Tab 切换交互（点击 Tab 按钮切换对应面板，active 态高亮），不新增任何统计指标
- `src/styles.css`：统一卡片层级（圆角8px/内边距16px/轻微阴影/标题14px加粗带色条）、指标卡片数值20px、图例统一12px；统一语义色——已完成=绿、进行中·新建=蓝、待处理=灰蓝、逾期·风险=红、预警=橙、次要=灰；修复「新建」柱与图例颜色不一致（原橙色柱改蓝色）；人员负载超人均=橙色预警、有逾期=红色风险分色显示；优先级「低」由绿色改灰色（避免与已完成绿色语义冲突）；补全缺失的组件样式（子Tab栏、卡片容器、风险提醒卡片/芯片、完成率排行、空态、近1月曲线图）
- `manifest.json` / `versions.json` / `CHANGELOG.md`：版本升至 6.13.39

## 验证

- 构建通过：`node esbuild.config.mjs production`；`tsc --noEmit` 无本次改动新增类型错误（仓库既有错误与本次无关）
- UI 截图验证：`tools/stats-panel-ui/` 真实 styles.css 渲染统计面板，三个 Tab 逐一切换截图（DOM 断言：Tab 点击切换生效、非激活面板隐藏、卡片容器/图例/标题样式生效、无横向溢出、无重叠溢出），OCR 结构检查识别到子 Tab 文案与各图表标题
- 验证位置：测试库 `ob_plan/test_vault`（dev v6.13.39，main.js/styles.css/manifest.json shasum 与工作区构建产物一致）
- 用户验证步骤：打开测试库 → 重载插件（设置→第三方插件 关闭再开启 task-gantt，或重启 Obsidian）→ 任务视图「读取库」→ 切到「统计面板」Tab → 预期：顶部出现「📋 项目总览 / 📈 任务统计 / 👥 人员统计」子 Tab 栏，默认显示项目总览（4 个指标卡片+项目任务统计+按文件夹统计+风险提醒）；点击「📈 任务统计」/「👥 人员统计」可切换对应图表组，激活 Tab 蓝色高亮；各图表卡片统一圆角、间距、阴影，图例颜色与图形一致

# TaskNexus v1.0.8
日期：2026-08-19

- 功能数：4 → 5
- 新增：表格列展示配置(table-cols)、局部排产(local-sched)、变更记录(changes)
- 移除：项目视图(projects)、版本更新(changelog)

# TaskNexus v1.0.8
日期：2026-08-19

- 功能数：4 → 5
- 新增：表格列展示配置(table-cols)、局部排产(local-sched)、变更记录(changes)
- 移除：项目视图(projects)、版本更新(changelog)

# TaskNexus v1.0.8
日期：2026-08-19

- 功能数：4 → 5
- 新增：表格列展示配置(table-cols)、局部排产(local-sched)、变更记录(changes)
- 移除：项目视图(projects)、版本更新(changelog)

# TaskNexus v1.0.7
日期：2026-08-18

- 功能数：4 → 2
- 移除：项目视图(projects)、版本更新(changelog)

# TaskNexus v9.9.1
日期：2026-08-18

- 功能数：2 → 1
- 移除：quickadd

# TaskNexus v1.0.7
日期：2026-08-18

- 功能数：4 → 2
- 移除：项目视图(projects)、版本更新(changelog)

# TaskNexus v1.0.7
日期：2026-08-18

- 功能数：4 → 2
- 移除：项目视图(projects)、版本更新(changelog)

# TaskNexus v1.0.7
日期：2026-08-18

- 功能数：4 → 2
- 移除：项目视图(projects)、版本更新(changelog)

# TaskNexus v1.0.6
日期：2026-08-18

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# TaskNexus v1.0.6
日期：2026-08-18

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# Task Gantt v1.0.6
日期：2026-08-18

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# Task Gantt v1.0.7
日期：2026-08-18

- 功能数：3 → 3
- 无功能变化（仅代码/修复）

# Task Gantt v1.0.7
日期：2026-08-18

- 功能数：3 → 3
- 无功能变化（仅代码/修复）

# Task Gantt v1.0.6
日期：2026-08-17

> 紧急缺陷修复：项目任务表格勾选子任务「已完成」后刷新页面勾选被自动取消。根因：勾选只写任务行、依赖异步 syncLinkedNote 补写关联笔记 frontmatter（且解析可能因 metadataCache 滞后失败而静默跳过），笔记 status 停留在「未完成」；下次扫描 loadLinkedNoteProps/lineFromFrontmatter 以笔记为权威把行首勾选强制改回 [ ]（✅ 完成日期残留）。

## 修复

### 任务表格 · 勾选完成持久化（根本修复）

- `src/renderer.ts`：`toggleTaskCheck` / `toggleTaskCancel` 写行后立即 **await 同步写透关联笔记 frontmatter**（status / percentComplete / actualStart / actualEnd），行与笔记保持一致的原子语义，扫描不再回退勾选（renderAll 尾部 5 秒窗口重复同步为幂等空操作）
- `src/utils/linked-note.ts`：`resolveLinkedNoteFile` 在 metadataCache 未命中时回退「任务笔记同级平铺路径」磁盘检查，保证平铺布局的既有关联笔记能被确定性找到（此前解析失败导致笔记写透被静默跳过）
- `src/scanner.ts`：`lineFromFrontmatter` 用户勾选优先——行内已有 ✅ 完成日期（仅勾选动作会写入）时，笔记 status 滞后为「未完成」不再回退行首勾选，并把被误回退的 [ ] 自愈为 [x]；完成度按 100% 处理，笔记 percentComplete 滞后不再剥掉行内 🔄 100%

## 验证

- 用个人库真实数据回归：11 行带关联笔记的子任务中，旧逻辑 9 行被回退（BUG 复现）；新逻辑 0 行回退，历史损坏数据（[ ]+✅ 残留）9/9 自愈为 [x]
- esbuild 生产构建通过（node v22.14.0 darwin-arm64）


> 任务表格新增「批量修改负责人 / 提出人」；双击完成%单元格弹出的进度控件与今日任务面板一致（25/50/75/100 一键 + 自定义输入回车确认）。

## 变更

- `src/view.ts`：底部批量操作栏新增「👤 负责人」「🗣️ 提出人」按钮（勾选任务后可用），弹层支持已有负责人/来源快捷选择、自定义输入回车确认、清除标签；写盘按文件分组复用批量优先级模式（以文件实际行为基准，父任务聚合不影响写盘）
- `src/view.ts`：变更记录新增「批量改负责人」「批量改提出人」类型标签
- `src/renderer.ts`：`editPercent` 改为弹出进度快捷菜单；新增共用 `openPercentMenu`（25/50/75/100 一键 + 自定义 0-100 输入回车确认，Esc 关闭，视口钳制）
- `src/view.ts`：今日面板/任务看板的 `_openPercentMenu` 改为委托共用 `openPercentMenu`，三处控件行为完全一致

## 验证

- 新增 `test-batch-edit.ts` 单元回归：负责人换人/新增/自我换他人/清除、提出人替换/新增/清除、复选框前缀与缩进保留、改负责人不动提出人等 12 场景全部通过
- 新增 `tools/batch-edit-e2e`（真实 GanttView + 真实写盘 + Chrome headless）：批量按钮置灰逻辑、弹层列表/自定义输入/清除项、选择与自定义回车后 DOM 与文件行标签落盘、表格双击弹出 25/50/75/100 菜单、点 50%/自定义 66 回车落盘、今日面板 chip 菜单回归，23 项断言全部通过
- 既有回归：`test-sort` / `test-hierarchy` / `test-note-group` / `test-filter` / `test-reqcheck` / `test-codex-logic` / `test-deploy-check` 全部通过；esbuild 生产构建通过
- 截图：`/Users/simonni/.codex/visualizations/2026/08/17/ct20-*.png`，OCR 复核批量负责人弹层、表格与今日面板进度菜单文字一致

# Task Gantt v1.0.7
日期：2026-08-17

- 功能数：3 → 3
- 无功能变化（仅代码/修复）

# Task Gantt v1.0.6
日期：2026-08-17

- 功能数：3 → 3
- 无功能变化（仅代码/修复）

# Task Gantt v1.0.6
日期：2026-08-17

- 手动维护的更新记录：新增按人排产体验优化
- 修复筛选栏对齐

## v1.0.5（2026-08-17）
- 功能数：3 → 3
- 无功能变化（仅代码/修复）

## v1.0.5（2026-08-17）
- 功能数：3 → 3
- 无功能变化（仅代码/修复）

## v1.0.5（2026-08-17）
- 功能数：3 → 3
- 无功能变化（仅代码/修复）

## v1.0.6（2026-08-17）
- 功能数：3 → 3
- 无功能变化（仅代码/修复）

## v1.0.5（2026-08-17）
- 首次构建（功能数：3）

