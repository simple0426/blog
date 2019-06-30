---
title: ansible之应用实践
tags:
categories:
---
# 异步模式
## 异步等待模式
* ansible在客户端异步执行，在任务执行过程中，无论ssh连接是否中断都不影响任务执行  
* async：任务执行超时时间，如无此关键词，则任务以同步模式执行【ansible默认模式】
* poll：任务轮询时间，大于0的正值，任务异步执行，但在任务出结果之前ansible在前台阻塞挂起

```
- name: test sleep
  command: /bin/sleep {{ 41 | random(35, 1)}}
  async: 100
  poll: 20
```
## 异步非等待模式
poll等于0，则任务不轮询结果；ansible在执行任务的过程中，把任务交给节点后即刻切换到下一个节点，不等待任务的执行结果。

```
- name: test sleep
  command: /bin/sleep {{ 41 | random(35, 1)}}
  async: 100
  poll: 0
```

## 稍后查询异步非等待模式结果
```
- name: test sleep
  command: /bin/sleep {{ 41 | random(35, 1)}}
  async: 100
  poll: 0
  register: sleep_status
- name: check status
  async_status:
    jid: "{{ sleep_status.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 10
  delay: 10
```

# debugger调试
+ 可以在任意具有name属性的区块设置，比如play、role、block、task
+ debugger值：
    - always：总是调用调试模块
    - never：从不调用
    - on_failed：仅在任务失败时调用调试模块
    - on_unreachable：当主机不可达时调用模块
    - on_skipped：任务跳过时调用模块
+ 可用命令：
    - p task：显示任务名称
    - p task.args：显示任务参数
    - p task_vars：显示任务变量
    - p host：显示任务操作主机
    - p result._result：显示任务执行结果
    - task_vars[key] = value：设置变量
    - task.args[key] = value：设置参数
    - r（edo）：重新运行任务
    - c（continue）：继续执行
    - q（uit） ：从debugger模式退出

# 信息加密：ansible-vault
# pull模式：ansible-pull
