---
title: "从不会 Django 到跑通第一个页面"
weight: 4
description: "ML 模块如何融入 Django，以及 CSRF 等经典 Bug 排查"
---

## 技术盲区

ML 模块测试通过后（91/91），我需要把它融入 Django。问题是我的 Django 仅粗略地学习了两周。核心困惑：

- Django 的 View 怎么和前端通信？
- 前端怎么拿到训练结果并渲染图表？
- 用户上传的文件怎么处理？

## 我的提示词——把 ML 和 Django 连接起来

> 接下来开发第二部分。前端使用纯 Django Template（HTML + jQuery/Bootstrap），要方便预留 Celery 异步处理节点。
>
> 请将 ml_utils.py 融入 Django 架构，前端界面要求：
> 1. 数据管理：上传 CSV → 后台存入 Media → 前端展示前 5 行预览
> 2. 特征配置：多选框选特征和目标，选择填充策略
> 3. 模型看板：ECharts 渲染多模型对比柱状图、热力图
> 4. 在线预测：上传新数据 → 加载模型 → 预测 → 下载 CSV
>
> **注意**：View 返回 JsonResponse，用 jQuery AJAX 异步获取并渲染。MySQL 仅保存相对路径。

> [!NOTE]
> **为什么强调"JsonResponse + AJAX"？** 传统做法是 View 直接 `render(request, 'xxx.html', context)` 把数据塞进模板。但这不适合图表交互——ECharts 需要在前端 JS 中动态获取数据。所以我让 AI 用 **API 分离模式**：页面 View 只渲染空 HTML 壳子，数据通过 `/api/` 接口返回 JSON，前端 jQuery 获取后渲染。

## AI 给出的关键架构决策

1. **ECharts 配置存 MySQL**：训练完成后 `Visualizer` 输出的 ECharts JSON 存入 `MLTask.charts_json`（Django 的 JSONField），前端直接 `setOption()` 渲染
2. **相对路径策略**：DB 只存 `media/upload_data/admin/xxx.csv`，View 层拼接 `BASE_DIR` 得绝对路径
3. **顺序流程约束**：通过 `DataSet.status`（`uploaded → preprocessed → completed`）强制约束

这三个决策 **我全部直接采用**，事后证明非常正确。

## 跑通第一个页面遇到的 Bug

### Bug：CSRF Token 缺失导致上传失败

错误日志：
```
Forbidden (CSRF token missing or incorrect.): /api/datasets/upload/
"POST /api/datasets/upload/ HTTP/1.1" 403 2519
```

**我的提示词**：

> 登录成功后跳转的是启动训练看板，但用户此时还未上传文件。另外，数据管理点击上传没反应（拖拽可以），点击上传后台报错 "Forbidden (CSRF token missing or incorrect.)"

**AI 的排查步骤**：

{{% details "Bug 1：登录跳转错误" %}}
登录后跳转 `/dashboard/`，但用户还没上传数据。改为跳转 `/data/`（数据管理页）。
{{% /details %}}

{{% details "Bug 2：点击上传无响应" %}}
`<input type="file">` 在 `#upload-zone` 内部导致点击事件冒泡冲突。移到外部即可。
{{% /details %}}

{{% details "Bug 3：CSRF 403" %}}
在 `layout.html` 中全局 `$.ajaxSetup` 自动附带 CSRF token，所有 AJAX POST 都会携带。
{{% /details %}}

![csrf修复对比](csrf-compare.png)

## 提示词工程心得：同时给错误日志 + 操作步骤

| ❌ 差的提示词 | ✅ 好的提示词 |
|---|---|
| "上传功能不工作" | "点击上传没反应（但拖拽可以），后台报错 Forbidden CSRF token missing。登录后跳转的页面也不对" |
| AI 需要猜问题在哪 | AI 直接定位三个不同的 Bug |
