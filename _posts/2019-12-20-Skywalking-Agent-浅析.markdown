---
layout: post
title: "SkyWalking Agent 浅析"
categories: DevOps
---
# SkyWalking Agent 浅析

## SkyWalking Agent 简介

SkyWalking是一个分布式APM（Application Performance Monitor）系统。

SkyWalking Agent是 SkyWalking Sniffer模块下，针对Java应用的无侵入性的应用探针。

SkyWalking Agent采用微内核架构，通过插件形式扩展。

## SkyWalking Agent 实现原理

SkyWalking Agent基于Java Instrumentation与字节码生成框架Byte Buddy。

### Java Instrumentation

Java Instrumentation基于JVMTI（JVM Tool Interface）。JVMTI是一套由Java虚拟机提供的，为JVM相关工具提供的本地API集合。Instrumentation最大的作用在于可以动态得更改类定义与其方法，运行时只能修改其方法体。JVM提供了javaagent参数，在运行main入口函数之前，执行基于Instrumentation的代理程序。

#### Java Instrumentation 详细流程

a. JVM启动，读取到javaagent参数，初始化其指定的Jar包，调用其的Agent_OnLoad函数。

b. 在Agent_OnLoad函数中，会获取JVM实例，调用RegisterEvent初始化注册JVMTI的事件回调函数，获取ClassFileLoadHook。

c. 然后在sun.instrument.InstrumentationImpl中调用loadClassAndCallPremain。将去初始化Premain-Class指定类的premain方法。

d. 执行Jar包中的premain函数，通过Instrumentation向JVM注册Agent的`ClassFileTransform`实例，这一步很`关键`。

e. Agent初始化完毕后，JVM调用main函数。JVM运行过程中在ClassLoader加载class文件之前，JVM`每次都会`（注意是每次，这就使得其具备了在运行时修改类方法体的能力）调用ClassFileLoadHook回调，该回调会调用ClassFileTransformer的transform函数，生成字节码。由于是在解析class之前，以二进制流的形式，对后续解析无影响。

以上就是Agent对代码无侵入的核心原理。

### Byte Buddy

Byte Buddy是JVM字节码操作工具，可以通过简单易用的API接口，封装JVM字节码底层细节，提供动态修改类结构、定义的一款开源工具。

Byte Buddy通过其自身生成字节码的能力，与Java Instrumentation的javaagent机制相结合，在应用的启动过程中按照自身需求，动态更改类定义，从而实现对应用进行无侵入探测的目标。

### SkyWalking Agent 实践

SkyWalking Agent是一个微内核架构模式。通过定义核心功能以及抽象核心接口，提供以插件形式集成服务的能力。

#### SkyWalking Agent 微内核

SkyWalking Agent核心功能目标是：a. 获取动态增强的目标类，b. 获取该类具体增强目标，c. 类动态增强。

为了实现上述目标，SkyWalking Agent启动时，会加载相应插件（默认是去工程的Resources文件夹下查找skywalking-plugin.def，在该文件中定义相关需要增强的类）。

SkyWalking Agent提供AbstractClassEnhancePluginDefine，ClassEnhancePluginDefine等基类，抽象，封装上述核心的插件启动，加载，运行逻辑。

采用Template设计模式，利用子类重载具体enhance，enhanceInstance，getConstructorsInterceptPoints，getInstanceMethodsInterceptPoints，enhanceClass，getStaticMethodsInterceptPoints等接口，达到上述a，b目标。具体的类增强目标，是通过获取到的InterceptPoints，利用Byte Buddy动态变更类定义。

通过InterceptPoint提供的拦截类，可以在方法执行前后对其进行拦截探测。

## SkyWalking Agent 扩展

如果我们需要对应用添加个性化的监控，通过自定义插件形式，继承上述ClassEnhancePluginDefine基类，重载对应关键接口，就可以通过SkyWalking Agent的内核，加载运行自己的自定义插件，完成对应用的无侵入探测。

InstanceMethodsInterceptPoint，StaticMethodsInterceptPoint两个接口，允许插件申明具体的增强点信息，提供插件自定义的拦截器类，交由Byte Buddy动态生成新的类，更改类默认的行为。