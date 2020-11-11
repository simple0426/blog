---
title: jenkins之groovy和脚本式pipeline
tags:
categories:
---

# 总览

声明式pipeline以pipeline为根节点，形如：

```
pipeline {
}
```

脚本式pipeline以node为根节点，形如：

```
node {
}
```

# groovy

* 定义变量

  ```
  def x="abc"
  def y=1
  ```

* groovy行末尾分号不是必须的

* 方法调用

  ```
  1. 方法没有参数或默认参数时，方法之后的括号必须使用
  def sayHello(String name="humans"){
    print "hello ${name}"
  }
  sayHello()
  2. 参数形式调用方法，可以省略括号
  System.out.println "Helloworld"
  def createName(String givenName, String familyName){
    return givenName + " " + familyName
  }
  createName familyName = "Lee",givenName = "Bruce"
  ```

* 引号

  * 单引号包含字符串，双引号可以引用变量名
  * 三单引号和三双引号都支持换行，但是三双引号支持变量引用，但三单引号不支持

