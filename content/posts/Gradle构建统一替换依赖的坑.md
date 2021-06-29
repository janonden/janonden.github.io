---
title: "Gradle构建统一替换依赖的坑"
date: 2018-10-31T11:13:51+08:00
categories: [Coding]
tags: [Gradle]
---

统一规定httpclient版本，采用构建时改变相关的依赖。
采用使用Gradle init.gradle方式，在执行Gradle构建开始时加进特殊替换处理的gradle脚本，脚本主要使用resolutionStrategy（依赖冲突解决策略），在init.gradle中配置resolutionStrategy逻辑

```
project.configurations.all {
    resolutionStrategy.eachDependency { DependencyResolveDetails details ->
        // 查找details中需要替换的目标
        // 使用details.useTarget
    }
}
```



## 坑1 - 构建服务器上有多个不同版本的Gradle

版本最低Gradle1.10，最高Gradle3.3

**方案**

1. 统一Gradle版本
2. 基于最低Gradle版本（1.10）相关接口编写脚本（目前采用这个方案）

## 坑2 - war工程中的providedCompile

providedCompile只用于构建，打包是不会放到WEB-INF/lib目录下。由于还存在不少war工程，而且有些是没人维护，构建输出存在不确定性；只有一小部分工程才会使用到httpclient，所以就采取把httpclient放到WEB-INF/lib目录。

**方案**

- 在afterEvaluate（在配置阶段要结束，Project评估完时执行的点）时，判断工程中是否有providedCompile，如果有，则把旧的httpclient添加exclude逻辑，重新添加新httpclient依赖到compile中
- 不能把resolutionStrategy去掉，因为不知道哪些依赖会依赖httpclient

## 坑3 - 与某些工程的gradle冲突

有些工程中的gradle文件会使用configurations.runtime等方法来获取依赖的jar包并放到输出的jar中，由于gradle的执行顺序，在afterEvaluate逻辑执行之前就已经把依赖和处理完毕，导致了两个问题：

1. 特殊逻辑执行失败。
2. jar中的classpath信息跟lib目录中不一致

**方案**

- 增加插入点，在beforeResolve（解析依赖包前，会有多次调用）时加多个检查，初始化resolutionStrategy逻辑以及坑2中afterEvaluate的逻辑，保证在工程gradle加载时需要获取依赖或者加载后，才执行特殊逻辑
- 好像可以把afterEvaluate的点去掉，不过目前不冲突，先暂时留着

## 坑4 - 与spring-boot-gradle-plugin插件冲突

由于是先加载init.gradle，然后再加载工程的build.gradle，所以httpclient的版本会被spring-boot-gradle-plugin插件强制替换版本，所以导致特殊替换逻辑不生效或者结果有问题（MD，跟了半天，加了日志也看到特殊逻辑替换成功，但是输出来的jar包就不是特殊逻辑中指定的那个）

**方案**

- 跟坑3类似，放在beforeResolve时加多个检查，初始化resolutionStrategy逻辑以及坑2中afterEvaluate的逻辑
- 检查DependencyResolveDetails时，不止检查requested（原有gradle文件中的依赖），而且要检查target（resolutionStrategy处理的结果）

## 坑5 - 某些第三方包只能用新版。。。

某些第三方包只能用新版（4.5.3或以上）暂时没啥好办法，只能先把工程加入白名单，后续再基于新版提供个吧

## 坑 - 预留。。。
