---
title: "异步预测：AI 引导搭建 Celery 任务"
weight: 7
description: "从同步阻塞到 Celery 异步，Windows SQLite broker 方案"
---

## 为什么需要异步？

最初训练是 **同步阻塞** 的——用户点"开始训练"后，浏览器一直转圈等后端跑完整个 pipeline（可能几分钟到几十分钟）。这期间用户什么也做不了。

AI 检查了我的代码，指出：

> 前端有 2s 轮询机制（为异步设计的），但后端是同步的。用户看到的就是"训练中"转圈直到后端跑完才一次性更新为"已完成"。

## 我的提示词——改造为 Celery 异步

> 是的，改造为 Celery 异步。我的电脑是 Windows 系统。

![我和AI讨论异步架构的对话片段](celery-discussion.png)

### AI 的关键设计决策

Windows 不支持 Redis 的默认 prefork 模式。AI 选择了 **SQLite 作为 Celery broker**（零依赖，开箱即用）：

```python
# settings.py
CELERY_BROKER_URL = 'sqla+sqlite:///celery_broker.db'
CELERY_RESULT_BACKEND = 'db+sqlite:///celery_result.db'
CELERY_WORKER_POOL = 'solo'  # Windows 单进程
```

启动 Worker 的命令：

```bash
celery -A mysite worker --pool=solo --loglevel=info
```

## 流程变化

```
之前: 用户点训练 → 浏览器转圈等后端跑完 → 一次性返回结果
现在: 用户点训练 → 立即返回 task_id → 前端每 2s 轮询状态
      → Celery worker 在后台跑 pipeline → 完成后 status 变为 completed
```

## 训练失败后的状态回退问题

训练失败后，我发现无法用同一个数据集重新训练，必须先删除失败的任务。

**我的提示词**：

> 为什么在报错之后，需要先删除报错的数据任务，才能执行下一个预测任务？

**AI 的定位**：训练失败时 `MLTask.status = 'failed'`，但 **DataSet 状态没有回退**，卡在中间状态。

修复很简单——在 Celery 任务的 except 块中加一行 `dataset.status = 'preprocessed'`，让数据集回退到可训练状态。

## ⚠️ 容易忽略的陷阱：修改代码后必须重启 Celery Worker

我在修改了 `tasks.py` 后，新训练的任务还是走的旧逻辑。AI 解释：

> Celery Worker 启动时加载代码到内存。修改 Python 文件后，Django 的 `runserver` 会自动热加载，但 **Celery Worker 不会**。必须手动 Ctrl+C 停掉 Worker 再重启。

> [!WARNING]
> 这不是 Bug，是 Celery 的设计——但如果你不知道这一点，会浪费很多时间在"为什么改了代码没生效"上。


## 提示词工程心得：给上下文 > 给命令

| ❌ 差的提示词 | ✅ 好的提示词 |
|---|---|
| "帮我加 Celery" | "改造为 Celery 异步。我的电脑是 Windows 系统" |
| AI 可能用 Redis 方案（Windows 不兼容） | AI 直接选 SQLite broker，一步到位 |
