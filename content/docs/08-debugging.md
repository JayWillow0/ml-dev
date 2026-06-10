---
title: "调试血泪史"
weight: 8
description: "4 个真实 Bug 的完整排查过程：从错误日志到 AI 定位修复"
---

## Bug 1：PNG 图片保存到了项目根目录

**现象**：训练完成后，项目根目录下出现一堆 `.png` 文件。

**我的提示词**：

> 模型预测保存的 PNG 图片保存到了项目的根目录下了

**AI 的修复**：`Visualizer` 的 `output_dir` 在 View 中没有显式设置，默认是 `'.'`（当前工作目录）。改为 `output_dir='media/charts/<task_id>/'`。

**我的后续决策**：我发现自己根本不需要 PNG——ECharts 的 JSON 配置已经够用了。于是我问：

> 我不需要生成 PNG 文件，应该注释哪里的代码？

> [!TIP]
> AI 的回答很巧妙：不需要逐个注释所有 `plot_*` 方法，只需让 `_save_png` 方法跳过保存即可。**一处改动控制所有 PNG 输出**。

---

## Bug 2：无监督学习查看图表弹出的是空看板

**现象**：无监督任务训练成功后，点击"查看图表"跳转到模型看板，但没有任何图表展示。

**我的提示词**：

> 训练好的结果在点击"任务追踪"的"详情"→"查看图表"时返回的是"启动训练"看板，并没有出现图表页面

**AI 的排查**：问题不在图表生成，而在页面跳转。Dashboard 只显示训练启动表单，不加载已有任务的图表。需要让 Dashboard 支持 `?task_id=xxx` 参数，自动加载图表。

**修复方案**：
- View 从 URL 读取 `task_id` 参数传入模板
- Dashboard JS 初始化时检测参数，非空则自动调用 `loadResults()` 渲染图表
- 任务列表的"查看图表"链接改为 `/dashboard/?task_id=<id>`

---

## Bug 3：降维为"No"时前端无图表

**现象**：聚类任务不选降维，后端运行正常（Celery 日志显示 succeeded），但前端不显示任何图表，连肘部图也没有。浏览器控制台报 500 错误。

**根因链条**（两层 Bug 叠加）：

{{% details "后端：NoneType AttributeError" %}}
`generate_unsupervised_charts` 中 `dim_reduction` 为 `None` 时，`None.get('PCA')` 抛出 `AttributeError`。

```bash
AttributeError: 'NoneType' object has no attribute 'get'
in generate_unsupervised_charts
```

**修复**：`r.get('dim_reduction', {}) or {}`（用 `or {}` 兜底 `None`）
{{% /details %}}

{{% details "前端：容器硬编码显示" %}}
`chartsToShow` 硬编码了全部 5 个无监督图表容器。不选降维时后端不生成散点图/碎石图，容器被 show 但内容为空。

**修复**：不提前 show 所有 wrap，改为只有 `loadChart` 成功获取到数据时才显示对应容器。
{{% /details %}}

---

## Bug 4：ECharts 图表数据标记显示 `{c:.2f}`、`{c:.4f}`

**现象**：训练结果的柱状图、热力图上，数据标签显示的是原始字符串 `{c:.2f}` 而不是格式化后的数字。

**根因**：Python 风格的格式串 `{c:.2f}` 不是 ECharts 的合法模板语法。ECharts 只支持裸 `{c}`（无格式修饰符）。

**修复策略**（AI 的方案很优雅）：不在 formatter 里做格式化，而是 **在构造数据时预先 round 到目标精度**：

| 图表方法 | 修复前 | 修复后 |
|---------|--------|--------|
| 相关性热力图 | `{c:.2f}` | data 预先 `round(v, 2)`，formatter → `{c}` |
| 特征重要性 | `{c:.4f}` | data 预先 `round(v, 4)`，formatter → `{c}` |
| 肘部曲线 | `{c:.0f}` | data 预先 `int(v)`，formatter → `{c}` |
| 碎石图 | `{c:.1f}%` | data 预先 `round(r*100, 1)`，formatter → `{c}%` |

> [!NOTE]
> 8 处 formatter Bug 统一用"数据预 round + 裸 formatter"策略修复，一劳永逸。


![修复后的ECharts图表对比](formatter-bugfix.png)

---

## 提示词工程心得：粘贴完整错误日志

| ❌ 差的提示词 | ✅ 好的提示词 |
|---|---|
| "降维不选的时候报错了" | "选择降维为 No 后，程序运行正常但前端无图表。浏览器报 500。后端日志：`AttributeError: 'NoneType' object has no attribute 'get' in generate_unsupervised_charts`" |
| AI 需要从头猜 | AI 直接定位到 `dim_reduction` 为 None 的问题 |
