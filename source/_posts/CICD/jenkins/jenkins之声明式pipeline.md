---
title: jenkins之声明式pipeline
date: 2020-02-10 15:12:41
tags:
    - jenkins
    - pipeline
categories: ['CICD']
---

# 基本结构-stages
```
pipeline {
   agent any    //指定构建环境【例如什么类型(物理机、虚拟机、docker)的哪个节点】

   stages {     //构建步骤，至少包含一个stage
      stage('build') {
        parallel{   //指定多个stage并行构建
            stage('build-1'){ 
                steps{    //在一个stage构建中的详细步骤，有且只有一个steps
                    echo 'build stage 1'
                }        
            }
            stage('build-2'){
                steps{
                    echo 'build stage 2'
                }        
            }
        }
      }
      stage('test') {
         steps {
            echo 'test stage'
         }
      }
   }
}
```

# steps内部指令

* sh：执行shell命令

* echo：执行echo命令

* script：执行groovy语法

* bat、powershel：执行windows批处理

* error：主动报错并终止pipeline执行

* timeout：代码块执行超时，默认单位min

  * time：超时时间
  * unit：时间单位，NANOSECONDS 、MICROSECONDS、MILLISECONDS、SECONDS、MINUTES(默认)、HOURS、DAYS

* waitUntil：不断重复waitUntil块内代码直到条件为true。waitUntil步骤最好与timeout步骤共同使用避免死循环

  ```
  timeout(time: 10,unit: 'SECONDS'){
    waitUntil{
      sleep(11)
      sh 'ok'
    }
  }
  ```

* retry：重复执行代码块

  ```
  retry(20){
    script{
      sh script 'curl http://example', returnStatus: true
    }
  }
  ```

* sleep：休眠时间。默认seconds，其他同timeout

  ```
  sleep(120)
  sleep(time: 2,unit: "MINUTES")
  ```

# 其他重要指令

* environment：用于设置环境变量，可以定义在pipeline或stage部分
* tools：定义可以使用的工具，可以定义在pipeline或stage部分
* input：定义在stage部分会暂停pipeline提示用户输入
* parallel：并行执行多个stage
* triggers：定义执行pipeline的触发器
* cron：用法同crontab，一般用于trigger【范例：分/时/日/月/周】

# pipeline设置-options

* buildDiscarder：保存的历史构建数量

  ```
  options { buildDiscarder(logRotator(numToKeepStr: '1')) }
  ```

* checkoutToSubdirectory：从版本库拉取代码到子目录中【默认：工作目录的根目录】

  ```
  options { checkoutToSubdirectory('foo') }
  ```

* disableConcurrentBuilds：禁止pipeline并行执行，防止出现抢占资源或调度冲突

  ```
  options { disableConcurrentBuilds() }
  ```

* newContainerPerStage：使用docker构建时，每个stage都使用新的容器

  ```
  options { newContainerPerStage() }
  ```

* retry：发生失败时执行重试的次数（包含第一次失败）；可以在pipeline或stage块

  ```
  options { retry(3) }
  ```

* timeout：执行超时时间，超过此时间pipeline被终止；可以设置在pipeline或stage块

  单位：HOURS、MINUTES、SECONDS

  实践：一般设置10分钟即可

  ```
  options { timeout(time: 1, unit: 'HOURS') }
  ```

# [参数化构建-parameters](https://jenkins.io/doc/book/pipeline/syntax/#parameters)

|   参数类型   |          参数说明          |
|--------------|----------------------------|
| string       | 字符串                     |
| text         | 文本，容纳多于字符串的信息 |
| booleanParam | 布尔类型                   |
| choice       | 下拉框多选                 |
| file         | 添加构建过程需要的文件     |
| password     | 密码类型                           |

```
pipeline{
    agent any

    parameters{
        choice(
            description: '你需要选择哪个模块进行构建？',
            name: 'modulename',
            choices: ['Module1', 'Module2', 'Module3']
            )
        string(
            description: '你需要在哪台机器上进行部署？',
            name: 'deploy_hostname',
            defaultValue: 'host131',
            )
        text(
            name: 'release_note',
            defaultValue: 'Release Note 信息如下所示:\n \
            bug-Fixed: \n \
            Feature-Added: ',
            description: 'Release Note的详细信息是什么？'
            )
        booleanParam(
            name: 'test_skip_flag',
            defaultValue: true,
            description: '你需要在部署之前执行自动化测试么？'
            )
        password(
            name: 'deploy_password',
            defaultValue: 'hejingqi',
            description: '部署机器连接时需要用到的密码信息是多少？'
            )
        file(
            name: 'deploy_property_file',
            description: "你需要输入的部署环境的设定文件是什么？"
            )
    }
    stages{
        stage('Build'){
            steps{
                echo "Build stage:选中的构建Module为：${params.modulename}..."
            }
        }
        stage('Test'){
            steps{
                echo "Test stage:是否执行自动化测试：${params.test_skip_flag}"
            }
        }
        stage('Deploy'){
            steps{
                echo "Deploy stage:部署机器的名称：${params.deploy_hostname}..."
                echo "Deploy stage:部署连接的密码：${params.deploy_password}..."
                echo "Deploy stage:Release Note的新的：${params.release_note}..."

            }
        }
    }
}
```
# [条件-when](https://jenkins.io/doc/book/pipeline/syntax/#when)【pre处理】
|    条件类型   |                                         使用说明                                        |                                              备注                                             |
|---------------|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| branch        | 指定分支构建时触发                                                                      | 只支持多分支pipeline                                                                          |
| buildingTag   | 构建tag时触发                                                                           | -                                                                                             |
| changelog     | SCM的变更日志包含指定内容时触发                                                         | 常与正则表达式结合使用                                                                        |
| changeset     | SCM的changeset包含指定文件时触发                                                        | 常与\*等表达式结合使用，缺省不区分大小写，可通过caseSensitive为true实现区分大小写             |
| changeRequest | 变更请求发生时触发（比如Github的Pull Request、Gitlab的Merge Request以及Gerrit的变更等） | 如未指定参数，每次请求都会触发；也可以指定【分支信息/标题/author/邮件地址】等信息作为参数触发 |
| environment   | 环境变量被设为某值时触发                                                                | -                                                                                             |
| equals        | 变量设为某值时触发                                                                      | -                                                                                             |
| expression    | 表达式为true时触发                                                                      | 返回的字符串必须转换为布尔类型才能操作                                                        |
| tag           | TAG_NAME变量满足匹配模式时被触发                                                        | -                                                                                             |
| not           | 当条件为false时触发                                                                     | -                                                                                             |
| allOf         | 当所有条件都为true才会被触发                                                            | -                                                                                             |
| anyOf         | 当所有嵌套条件至少一个为true时会被触发                                                  | -                                                                                             |
| triggeredBy   | 当前构建为指定方式时触发【触发条件：SCMTrigger/TimerTrigger/UpstreamCause】             | -                                                                                             |

```
pipeline{
    agent any

    environment{ //设置环境变量
        ENVIRONMENT_TEST_FLAG = 'NO'
    }
    stages{
        stage('Init'){
            steps{
                script{ //可以写grovy脚本，此处只定义变量
                    BUILD_EXPRESSION = true
                    DEPLOY_USER = 'liumiaocn'
                }
            }
        }
        stage('Build'){
            when{
                expression {BUILD_EXPRESSION} //表达式为真则执行
            }
            steps{
                sh 'echo Build stage ...'
            }
        }
        stage('Test'){
                when{
                    environment name: 'ENVIRONMENT_TEST_FLAG', //环境变量是指定值则执行
                    value: 'YES'
                }
                steps{
                    sh 'echo Test stage ...'
                }
        }
        stage('Deploy'){
            when{
                equals expected: 'liumiaocn', //变量是指定值则执行
                actual: DEPLOY_USER
            }
            steps{
                sh 'echo Deploy stage ...'
            }
        }
    }       
}
```
# post处理
>根据pipeline或stage执行结果进行后续处理

## 内置条件
* always：无论pipeline或stage执行结果如何，都会执行下面的操作
* changed：只有pipeline或stage执行结果与之前相比，状态发生变化，才会执行下边的操作
* fixed：前一次执行结果为不稳定或失败状态，本次执行成功，才会执行下边的操作
* regression：本次执行结果为不稳定、失败或中止状态，上次为成功状态
* aborted：本次执行操作被手动中止，UI显示为灰色
* failure：本次执行为failed状态，UI显示为红色
* success：本次执行成功，UI显示为绿色
* unstable：由于测试失败或代码规范性检查失败产生的状态，UI显示为黄色
* unsuccessful：执行结果不是success
* cleanup：在其他post条件后，无论执行结果如何都会执行

## 范例
```
pipeline{
    agent any
    stages{
        stage('Example'){
            steps{
                echo 'Hello World'
            }
        }     
    }
    post{
        always{
            echo 'I will always say Hello World'
        }
    }
}
```
