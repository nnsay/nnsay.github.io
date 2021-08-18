---
title: 阿里云迁移中吃过的亏(持续更新)
date: 2021-08-17 11:50:57
tags:
  - Aliyun
  - DevOps
  - CloudNative
---

### 0. 更新时间
- 2021-8-17: ASK/SLS/ECI/KMS/OSS

### 1. 背景
7月中旬完成了AWS到Alicloud的迁移, 迁移前后通过压测和生产环境的持续巡检发现了一些问题, 列举出来希望后来人避免跳坑.

### 2. 吃亏事件

#### 2.1 ASK(Aliyun Severless Kubernetes) PV/PVC 莫名其妙的Lose
- 问题描述: 使用Terraform在ASK中创建的PV/PVC状态过一段PVC状态就从Bound变成了Lose, 这个会导致所有引用PVC的Container处于一直Pending状态无法进行正常业务处理
- 原因: 查看k8s的audit日志, 发现PVC的状态有被更新成Lost, 查看更新的操作的来源其实来自Terraform, 如图:
![update pvc status](4290636159088.png)
![operator](1335619716611.png)
- 解决方案: 在Terraform中忽略PV/PVC的所有变更, 即Terraform只负责创建和删除, 不进行任何多余的维护, 维护的工作应该交给Kubernetes

#### 2.2 日志服务 SLS
- 问题描述: 在Container中配置了`aliyun_logs_*`等环境变量但是最后看不日志项目
- 原因: 日志项目名称阿里云全局唯一, 我用的日志项目名称太简单了, 已经存在了
- 解决方案: 日志项目名称增加公司名前缀
-
#### 2.3 弹性容器实例 ECI API
- 问题描述: Pod启动出错
- 原因: 触发了ECI API 节流阈值
![4053715841454](4053715841454.png)
![5744833505703](5744833505703.png)
- 解决方案: 工单申请增加ECI API的阈值

#### 2.4 密钥管理服务 KMS
- 问题描述: Pod启动失败, 代码进程报错退出
- 原因: 代码逻辑触发了KMS API 节流
- 解决方案: 
    - 提交工单申请更大的 KMS API 阈值
    - 代码修改捕获Throttling错误, 随机时长delay重试

#### 2.5 对象存储 OSS
- 问题描述: OSS结合CDN使用后上传失败
- 原因:  CDN上传最大支持300M, OSS直传没有这个限制, 之所以一定要用CDN两个原因: a. 自定义域名; b. 下载频繁
- 解决方案: 无法解决, 前端上传时超过300M提醒用户

#### 2.6 弹性容器实例 ECI CPU数量
- 问题描述: 压力测试逐步增加并发, 但是没有新的Pod都处于ProviderFailure状态
- 原因:  ECI CPU不够
![eci cpu](1501583789858.png)
- 解决方案: l提交工单增加ECI CPU阈值
