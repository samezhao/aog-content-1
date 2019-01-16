---
title: "WebJobs 深度使用详解"
description: "WebJobs深度使用详解"
author: hylinux
resourceTags: 'App Service Web, WebJobs'
ms.service: app-service-web
wacn.topic: aog
ms.topic: article
ms.author: hongwei.guo
ms.date: 01/11/2019
wacn.date: 01/11/2019
---

# WebJobs 深度使用详解

该文章不着眼于 WebJobs 的基础，如果想了解如何在门户上创建和使用 WebJobs，请移步：[在 Azure 应用服务中使用 WebJobs 运行后台任务](https://docs.azure.cn/zh-cn/app-service/web-sites-create-web-jobs)。

## 什么是 WebJobs

WebJobs 是 Web Service 提供的一个特性，其主要的设计目的是为了满足客户运行后台任务的需要，当前 WebJobs 和 Web Service 是共用 Web Context 的。从技术上看是由 *w3wp.exe (scm)* 这个进程来引导的。但是根据 WebJobs 的类型不同，在触发上还是有区别的。当前有两种类型的 WebJobs，一种是连续运行的 WebJobs， 一种是触发运行的 WebJobs， 这两种 WebJobs 最大的区别在于：连续运行的 WebJobs 是可以分布在多个 Web Service Instance 上运行，同时由 WebJobs 的 host container 来自动做请求分片， 但是触发型的 WebJobs 只能在一个 Instance 上运行，并且定时的 WebJobs 由背后的 Azure Scheduler Service 来触发。 所以在 WebJobs 里首先明白两种 WebJobs 有什么区别非常重要。

## WebJobs 的配置项

### 应用程序配置项

1. **WEBJOBS_RESTART_TIME**：该项仅仅是为连续型的 WebJobs 设定的，单位是秒，设定该值意味着，当一个连续 WebJobs 停止之后（无论什么原因），经过多长时间系统会自动重启该 Web。
2. **WEBJOBS_IDLE_TIMEOUT**：该项仅仅支持触发型的 WebJobs，单位是秒，如果一个触发型的 WebJobs 什么事也不做，也就是 idle 状态，系统会终止该 WebJobs，这个时间就是设定一个阈值，idle 多长时间，系统把它终止（所谓的 idle 就是不占用 CPU，没有输出）。
3. **WEBJOBS_HISTORY_SIZE**：该项仅仅针对触发型的 WebJobs，默认情况系统会保存 50 次触发的日志记录，超过 50 次会被复写，如果需要保持更多的日志记录，适当的增加该值。
4. **WEBJOBS_STOPPED**：如果设置该值为 1，相当于停止使用 WebJobs 的功能，包括如果有正在运行的 WebJobs 也会被停止。
5. **WEBJOBS_DISABLE_SCHEDULE**：如果该值设置为 1，那么所有触发型的 WebJobs 不会被触发，但是可以手动运行。
6. **WEBJOBS_ROOT_PATH**：用于更改 WebJobs 默认的 root path。
7. **WEBJOBS_LOG_TRIGGERED_JOBS_TO_APP_LOGS**：该项也是针对于触发型的 WebJobs，不过要配合 Web Service 的诊断日志打开，设置为 true 之后，WebJobs 的日志会被重定向到 Application pipeline，也即 Web Service 的诊断日志，如果诊断日志被配置为存储到文件系统，那么 WebJobs 的日志也会被存储到文件系统，如果是存储到 storage，那么 WebJobs 的日志也会被存储到 storage，或者 Application insight tec。

### 环境变量

当一个 WebJobs 运行起来之后，会传入如下的系统变量到当前的进程中，根据您使用的语言不同，可以存取这些系统的环境变量：

* **WEBJOBS_PATH**：正在运行的 WebJobs 的路径。
* **WEBJOBS_NAME**：正在运行的 WebJobs 的名字。
* **WEBJOBS_TYPE**：正在运行的 WebJobs 的类型（连续型还是触发型）。
* **WEBJOBS_DATA_PATH**：当前 WebJobs 的元数据目录。
* **WEBJOBS_RUN_ID**：当前触发型的运行 ID。

## WebJobs 的日志记录

1. 连续型 WebJobs：

    标准输出和标准错误被重定向到 Applications Logs(Application pipeline)，根据您当前的应用诊断日志的设定，会自动重定向到 d:\home\Logsfile 或者初始化到 storage etc，并且在每一次调用的时候，开始的 100 行日志记录也会放置到 data/jobs/continuous/jobname。

2. 触发型 WebJobs：

    默认是将标准输出和标准错误保存在 data/jobs/triggered/jobName/runId，默认是 50 次运行记录，可以通过更改变量 WEBJOBS_HISTORY_SIZE 来调整这个大小，同时，如果设置了变量 WEBJOBS_LOG_TRIGGERED_JOBS_TO_APP_LOGS 为 true，也会将标准输出和标准错误重定向到 Application pipeline，也会和连续型的 WebJobs 一样。

## Settings.job 参考指南

Settings.job 是一个 json 文件，可以同您的应用一起打包到一个 zip 文档里然后发布到您的 WebJobs。

这里有一个实例：

```json
{
  "schedule": "0 * * * * *",
  "is_in_place": true
}
```

其中可以使用的元素列表如下：

* [is_in_place](https://github.com/projectkudu/kudu/wiki/WebJobs#webjob-working-directory)：设置为 true，允许 WebJobs 在上传的目录里运行，不需要拷贝到暂存的目录里。
* [stopping_wait_time](https://github.com/projectkudu/kudu/wiki/WebJobs#graceful-shutdown)：通常用于连续型 WebJobs 用于控制停止的动作。
* [is_singleton](https://github.com/projectkudu/kudu/wiki/WebJobs-API#set-a-continuous-job-as-singleton)：是否仅仅运行在一个实例上。
* [schedule](https://github.com/projectkudu/kudu/wiki/WebJobs#scheduling-a-triggered-webjob)：触发型 WebJobs 的运行时间设定。

## WebJobs SDK 开发 tips

WebJobs 的开发支持很多的语言，可以运行 jar，exe，bat 等，但是想要完整地使用 WebJobs 的全部功能， 必须使用 C# 通过 WebJobs SDK 来开发，如果想尝试其他的语言，只能使用另外一个服务：Functions，Functions 基于 WebJobs SDK，同时提供其他语言的 SDK，像 Java 等，但是 Function 和 WebJobs 的区别主要是：

1. WebJobs 是需要自己通过代码来创建 WebJobs Host，然后 WebJobs 的代码运行在 WebJobs Host 这个容器里。但是 Function 是由平台创建好 WebJobs Host，用户只能通过 host.json 来进行调整。
2. Functions 支持更多的触发器，但是 WebJobs SDK 支持少量的触发器，WebJobs 可以通过 WebJobs 扩展支持更多的触发器。

C# 的 WebJobs SDK 主要的理念就是触发器，input 绑定，output 绑定，可以参考[如何使用 Azure WebJobs SDK 进行事件驱动的后台处理](https://docs.azure.cn/zh-cn/app-service/webjobs-sdk-how-to)。

由于 WebJobs SDK 是运行在 WebJobs Host 这个容器里，这个容器对于 out going 的 connection 是有限制的，默认是 2 个，也就是不管你在 WebJobs 里触发多少个 out going connection，只有 2 个被送到 OS 端，其他的被队列起来了，要改变这个需要更改 System.Net.DefaultConnectionLimited 的值，推荐直接设置为 Int32.MaxValue。

详细的 WebJobs SDK 开发请参考文档：[如何使用 Azure WebJobs SDK 进行事件驱动的后台处理](https://docs.azure.cn/zh-cn/app-service/webjobs-sdk-how-to)。