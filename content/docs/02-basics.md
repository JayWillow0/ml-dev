---
title: "前端 + 后端 + 数据库速通"
weight: 2
description: "零基础必备知识精炼，含安装避坑指南"
---

在正式开发之前，我**花了 2 周的时间补前端和 Django 的基础知识**。以下是精炼后的核心要点。

## 前端三件套：你需要理解到什么程度？

| 技术 | 你需要会的 | 不需要会的 |
|------|-----------|-----------|
| **HTML** | 标签结构（`<div>`, `<form>`, `<table>`）、表单提交 | CSS 动画、复杂布局 |
| **CSS** | Bootstrap 栅格系统（`col-md-6`）、按钮样式 | 手写响应式、Sass |
| **JavaScript** | jQuery 的 `$.ajax` 发请求、DOM 操作（`$('#id')`） | 原生 JS 闭包、原型链 |

> [!TIP]
> **关键认知**：你不需要"学会前端"，你需要的是 **能读懂 Bootstrap 模板并改文字**。真实项目中，我 95% 的 HTML 是 AI 写的，我只负责检查。

## Django 核心概念速通

Django 的开发流程可以浓缩为一句话：**定义 Model → 写 View → 配 URL → 填 Template**。

| 概念 | 类比 | 核心要点 |
|------|------|---------|
| **Models** | 数据库表的定义 | 用 Python 类定义字段，`makemigrations` + `migrate` 两步建表 |
| **Views** | 后端业务逻辑 | 接收请求、操作数据库、返回响应（HTML 或 JSON） |
| **URLs** | 路由表 | 把 URL 路径映射到 View 函数 |
| **Templates** | HTML 模板 | `{% extends 'layout.html' %}` 继承基模板，`{% block content %}` 填内容 |

## 我踩的坑

> [!WARNING]
> **App 注册**：创建 APP 后必须在 `settings.py` 的 `INSTALLED_APPS` 中注册，否则 `makemigrations` 不会生成迁移文件。

> [!WARNING]
> **CSRF Token**：Django 默认开启 CSRF 保护。AJAX POST 请求不带 token 会返回 403。解决方案是在 `layout.html` 中全局设置 `$.ajaxSetup({headers: {'X-CSRFToken': csrftoken}})`。

> [!WARNING]
> **静态文件 vs 媒体文件**：`static/` 放项目自身的 CSS/JS/图片，`media/` 放用户上传的文件。`media` 需要在 `urls.py` 中配置 `serve` 才能访问。

## MySQL 安装避坑

- MySQL 5.7 免安装版需要手动创建 `my.ini` 配置文件
- 初始化命令必须用 **管理员权限** 运行 cmd：`mysqld.exe --initialize-insecure`
- 设置密码：`set password = password("你的密码");`

> [!CAUTION]
> **千万不要用字符串拼接 SQL**，必须用参数化查询防注入。Python 推荐用 `mysqlclient`（`pip install mysqlclient`）。

## 关键依赖安装清单

```bash
pip install django==3.2.9
pip install mysqlclient
pip install celery
pip install sqlalchemy       # Celery SQLite broker 驱动
pip install scikit-learn
pip install lazypredict
pip install optuna
pip install numpy pandas
pip install matplotlib
pip install Pillow           # 验证码生成
```
