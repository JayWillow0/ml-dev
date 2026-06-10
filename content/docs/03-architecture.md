---
title: "架构构思与第一次提示词设计"
weight: 3
description: "如何设计第一份提示词，让 AI 理解你的 ML 平台需求"
---

## 我的想法

我要做的平台叫 **ML计算平台**，核心功能是 7 步：

```
用户登录 → 上传数据 → 选择特征和标签 → 选择学习类型 → 训练+调参 → 保存模型 → 在线预测
```

我非常清楚机器学习做监督学习和非监督学习的工作流（尤其基于 scikit-learn 框架），但我的同事大多没有这方面的基础。因此方案需要从新手用户角度出发，尽可能避免用户做复杂的数据处理、特征工程、模型选择和模型调参工作。

## 第一次提示词——告诉 AI "我要什么"

我花了**大概 1 天的时间**写这段提示词（精简版）：

> 我想开发一个具备登录认证、数据永久存储的机器学习算法平台，功能包括：
> 1. 用户导入 CSV/XLSX 数据
> 2. 用户选择特征和标签
> 3. 监督学习（回归/分类）走相关性分析 + lazypredict + optuna 调参；非监督学习走聚类
> 4. 模型保存、新数据预测、结果导出
>
> 前端用 HTML + Bootstrap + JavaScript，后端 Django，数据存 MySQL。
>
> **请先只实现 ML 算法部分（不涉及前端和 Django），集成到 ml_utils.py 中，方便后续导入 Django。** 我的环境是 Python 3.11.7、Django 3.2.9、scikit-learn 1.9.0、optuna 4.9.0。

### 为什么这么写？

1. **先限定范围**——"先不做前端"避免 AI 一次输出几千行代码
2. **给出具体包名和版本**——避免 AI 用了不兼容的 API
3. **说清楚集成方式**——"集成到 ml_utils.py"让 AI 知道模块化要求

## AI 的建议：哪些我直接用了，哪些我调整了

{{< tabs >}}

{{% tab "✅ 直接采用的" %}}

- **8 个类的模块化设计**：`DataImporter`、`DataPreprocessor`、`FeatureSelector`、`SupervisedTrainer`、`UnsupervisedTrainer`、`ModelTuner`、`ModelManager`、`MLPipeline`
- **lazypredict 批量训练 30+ 模型** + Optuna 自动调参的组合方案
- **`MLPipeline` 作为编排器**统一调用其他类

91 项单元测试全部通过。

{{% /tab %}}

{{% tab "🔧 我调整的" %}}

AI 原本没有设计 `Visualizer` 类。我在看到 ML 测试通过后，追问了"可视化数据在哪里"，AI 建议抽取一个 `Visualizer` 类。我选择了"两种都要"（ECharts JSON + Matplotlib PNG），后来经过充分验证后不需要保留 PNG，注释掉了 `_save_png` 方法——**一处改动控制所有 PNG 输出**。

{{% /tab %}}

{{< /tabs >}}

## 提示词工程心得：分阶段推进

| ❌ 差的提示词 | ✅ 好的提示词 |
|---|---|
| "帮我做一个机器学习网站" | "先只实现 ML 核心模块，集成到 ml_utils.py，不涉及前端" |
| 范围太大，AI 要猜 | 范围锁定，AI 专注输出 |

## AI给出的Python类架构

```
MLError (基类异常)
├── MLDataError          # 数据相关错误
├── MLFeatureError       # 特征处理错误
├── MLTrainingError      # 训练错误
├── MLTuningError        # 调参错误
└── MLModelError         # 模型保存/加载错误

DataImporter             # 数据导入（CSV/XLSX，自动编码检测）
DataPreprocessor         # 预处理（缺失值填充、编码、缩放）
FeatureSelector          # 特征选择（F检验/互信息/Top-K）
SupervisedTrainer        # 监督学习（lazypredict 批量训练 ~30 个模型）
UnsupervisedTrainer      # 无监督学习：聚类 + 异常检测 + 降维
  ├── cluster()          #   聚类（KMeans/DBSCAN/Agglo./Spectral/GMM）
  └── reduce_dimensions()#   降维（PCA/t-SNE）
ModelTuner               # 超参数调优（Optuna）
ModelManager             # 模型持久化（joblib 保存/加载/预测/导出，仅监督学习）
MLPipeline               # 流水线编排
  ├── run_supervised()        # 监督学习全流程
  ├── run_unsupervised()      # 聚类分析全流程
  ├── run_anomaly()           # 异常检测全流程
  └── run_dim_reduction()     # 降维可视化全流程
Visualizer               # 可视化（ECharts JSON + Matplotlib PNG）
  ├── generate_supervised_charts()       # 监督学习图表
  ├── generate_unsupervised_charts()     # 聚类图表（肘部/轮廓系数对比/散点）
  └── generate_dim_reduction_charts()    # 降维图表（碎石图/散点）
```
