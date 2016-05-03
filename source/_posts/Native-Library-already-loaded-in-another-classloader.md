---
title: Native Library already loaded in another classloader
date: 2016-05-03 15:16:38
tags: [java]
---

应用服务器上多个webapp都使用了 ImageMagick来处理图片。
运行是发生如下异常：
```
java.lang.UnsatisfiedLinkError: Native Library /usr/lib/libJMagick.so already loaded in another classloader
     java.lang.ClassLoader.loadLibrary0(ClassLoader.java:1772)
     java.lang.ClassLoader.loadLibrary(ClassLoader.java:1732)
     java.lang.Runtime.loadLibrary0(Runtime.java:823)
```
ps：ImageMagick是以JNI本地库的形式调用。

<!--more-->

## 现象
如果服务器上多个webapp都需要使用相同的JNI本地库，当第二个webapp加载JNI本地库时，就会出现Native Library /usr/lib/libJMagick.so already loaded in another classloader 的异常。

## 原因
Jvm为了在jni本地库中确保基于classloader的命名空间独立，因而不允许一个jni本地库被两个不同的classloader加载，而应用服务器resin/tomcat等中的webapp的classloader是独立的，也就是每个webapp都用一个专属的classloader，这样就出现了两个classloader加载同一个jni本地库的情况。

## 解决
在应用服务器resin/tomcat中，虽然不同的webapp使用的不同classloader，但是webapp classloader的父classloader的相同的，这样根据双亲委派模型只要让父classloader加载jni本地库就可以避免被多个classloader加载。

把各个webapp的pom.xml中ImageMagick的依赖修改为`<scope>provided</scope>`,然后把ImageMagick的jar包放到应用服务器的classpath中，即应用服务器的lib目录。
