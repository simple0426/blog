---
title: jenkins之pipeline使用
date: 2020-02-10 15:12:41
tags:
    - jenkins
    - pipeline
categories: ['CICD']
---

# pipeline插件
* pipeline
* blue ocean(图形化/更直观的查看pipeline执行状态)

# 基础语法结构stages
```
pipeline {
   agent any    //指定构建环境 

   stages {     //构建步骤，至少包含一个stage
      stage('build') {
        parallel{   //指定多个stage并行构建
            stage('build-1'){ 
                steps{    //在一个stage构建中的详细步骤
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

# [参数化构建](https://jenkins.io/doc/book/pipeline/syntax/#parameters)
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
# [when处理](https://jenkins.io/doc/book/pipeline/syntax/#when)【pre处理】
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

# if语句
>grovvy编程，等同于pipeline下的when

```
pipeline{
    agent any

    environment{
        ENVIRONMENT_TEST_FLAG = 'NO'
    }
    stages{
        stage('Init'){
            steps{
                script{
                    BUILD_EXPRESSION = true
                    DEPLOY_USER = 'liumiaocn'
                }
            }
        }
        stage('Build'){
            steps{
                script{
                    if(BUILD_EXPRESSION){
                        sh 'echo Build stage...'
                    }
                }
            }
            // when{
            //     expression {BUILD_EXPRESSION}
            // }
            // steps{
            //     sh 'echo Build stage ...'
            // }
        }
        stage('Test'){
            steps{
                script{
                    if(ENVIRONMENT_TEST_FLAG == 'YES'){
                        sh 'echo Test stage...'
                    }
                }
            }
                // when{
                //     environment name: 'ENVIRONMENT_TEST_FLAG',
                //     value: 'YES'
                // }
                // steps{
                //     sh 'echo Test stage ...'
                // }
        }
        stage('Deploy'){
            steps{
                script{
                    if(DEPLOY_USER == 'liumiaocn'){
                        sh 'echo Deploy stage...'
                    }
                }
            }
            // when{
            //     equals expected: 'liumiaocn',
            //     actual: DEPLOY_USER
            // }
            // steps{
            //     sh 'echo Deploy stage ...'
            // }
        }
    }       
}
```
