# QuickMVC

## 源码地址

[quickmvc](https://github.com/liuyueyi/quick-mvc)

## quick-mvc
> 自己动手实现的一个mvc框架，just for study

## Background

### 1. Why do this?

做了两年的java-web开发，平时对于spring-mvc多是处于使用阶段，很多东西只知道可以这么做，但是不清楚内部的逻辑和原理

其次就是遇到一些问题，可能不知道怎么下手去调试（特别是某些bean没有被加，或者加载了两边；切面死活切不到等）

再其次就是对spring-mvc中的一些设计还是蛮感兴趣的，想去看一下人家是怎么玩的，又为什么这么玩

再再其次，恰好在github上看到不少相关的东西，重复造轮子起始也没有什么问题，只要在造轮子的过程中能掌握造轮子的技术就是值得的

### 2. What to do?

大致上参考spring-mvc，然后就是github上找到的两个工程

- [jw](https://github.com/menyouping/jw)
- [Dump](https://github.com/yuanguangxin/Dump)

期望做的事情

- bean的扫描，自动加载
- ioc
- aop
- servlet-dispatcher （path -> controller映射）
- 事务
- jpa
- cache 封装
- 消息事件
- 过滤器： filter
- 拦截器： interceptor
- Servlet
- 统一异常处理
- 视图

## doing

### 1. 指定path路径扫描bean

已完成，支持扫描jar包

### 2. IoC

已完成，支持自动依赖注入，目前所有bean都是单例

### 3. AOP

完成基本功能


