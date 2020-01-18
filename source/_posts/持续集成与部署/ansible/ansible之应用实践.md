---
title: ansible之应用实践
date: 2019-07-02 11:04:24
tags:
  - 异步
  - debug
  - 错误处理
categories: ['ansible']
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
* poll等于0，则任务不轮询结果
* ansible在执行任务的过程中，把任务交给节点后即刻切换到下一个节点，不等待任务的执行结果。

```
- name: test sleep
  command: /bin/sleep {{ 41 | random(35, 1)}}
  async: 100
  poll: 0
```

## 查询非等待模式结果
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

# 错误处理
## 忽略错误继续执行
- 默认行为：遇到错误会终止错误处以后task的执行
- 语法：ignore_errors：yes
- 定义位置：task

## 重置主机不可达错误
- 默认行为：当执行某任务时，主机不可达，会退出正在执行的任务进而退出整个play
- 设置后：重置主机不可达错误时，某个task的主机不可达不影响其他task的连接尝试
- 语法：ignore_unreachable: yes
- 定义位置：play

## 强制handler执行
* 默认行为：一个play中某个task的失败，会引起已经触发的handler不再执行
* 语法：force_handlers: True
* 定义位置：play、ansible.cfg或命令行（--force-handlers）
  
```
force_handlers: true
handlers:
  - name: touch file
    command: touch /tmp/file
tasks:
  - name: exe handlers
    command: touch /tmp/file1
    notify:
      - touch file
  - name: touch failed
    command: touch /home/muk/ceshi
```

## 任何错误都退出
* 默认情况下：在多个主机执行任务时，在一个主机上遇到错误只造成此主机剩余任务不执行，不影响其他主机任务的执行【多个主机间的错误互不干扰】
* 设置后：任何主机的任何错误都会造成整个play的退出，不再执行。
* 语法：any_errors_fatal: true
* 定义位置：play

## 重定义失败状态
* 定义位置：task
* 语法：failed_when
* 含义：只有满足自定义的条件才算是失败状态

```
- name: test fail when
  command: touch /home/ceshi/12
  register: result
  failed_when: "'No such' in result.stderr"
```

## 重定义变更状态
* 定义位置：task
* 语法：changed_when
* 含义只有满足自定义的条件才算是“变更”状态

## 使用block处理错误
