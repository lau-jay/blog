---
title: "在Airflow中使用BashOperator的环境变量问题"
date: 2025-05-28T23:12:49+08:00
draft: false
tags: ["Airflow", "NodeJS", "BashOperator", "环境变量"]
categories: ["技术解决方案"]
---

## 背景

我们有一个数据生成需求，官方仅提供了NodeJS SDK实现。整套T+1增量数据处理流程基于Airflow构建，需要在Airflow任务中将生成的文件上传至指定S3存储桶。技术背景：
- NodeJS实现完全基于官方SDK示例和AI辅助开发
- Airflow环境由其他团队搭建维护
- 缺乏系统的Airflow文档知识

## 技术方案选型

### BashOperator调用方案

在Airflow中调用NodeJS程序最直接的方案是使用`BashOperator`执行node命令。该方案存在三个关键技术要点：

1. **任务隔离性**
每个BashOperator任务会启动独立的Shell子进程（执行模式为`/bin/bash -c "your_command"`），与Airflow Worker进程完全隔离。

2. **环境变量隔离**
子进程默认不会继承父进程的环境变量，必须显式传递。

3. **工作目录差异**
默认工作目录为Airflow临时目录（如/tmp），而非DAG文件所在目录。

## 问题排查过程

### 初始现象
1. 首次部署时由于目录挂载配置错误，NodeJS脚本未正确打包到容器内，报`no module`错误
2. 修正目录后NodeJS可执行，但出现S3上传授权失败
3. 相同凭证的Python任务可正常操作S3

### 深度排查
1. 排除NodeJS脚本逻辑错误
2. 与运维团队核对AWS IAM权限策略，确认权限配置正确
3. 在NodeJS中添加环境信息输出，发现实际使用的ARN与预期不符

### 根本原因
BashOperator创建的Shell进程未继承Airflow Worker的环境变量，导致：
- AWS临时凭证未传递
- 实际使用默认凭证链中的随机凭证
- 与运维配置的授权策略不匹配

## 解决方案

显式传递环境变量上下文：

```python
BashOperator(
    task_id='nodejs_task',
    bash_command='node script.js',
    env={**os.environ, 'CUSTOM_VAR': 'value'},  # 继承并扩展环境变量
    cwd='/path/to/scripts'  # 显式设置工作目录
)
```
