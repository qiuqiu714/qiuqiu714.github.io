---
title: crontab执行jar包版本不匹配问题
date: 2018-04-26 10:35:00
categories:
- java
tags: crontab, java
---

在linux环境上使用java -jar 命令运行jar包时遇到这样一个问题

正常运行没报错
但将命令写成shell脚本并且添加到crontab定时执行时报错，报错如下：

> Exception in thread "main" java.lang.UnsupportedClassVersionError:         
    > com/kingnetdc/streaming/Main : Unsupported major.minor version 52.0
        at java.lang.ClassLoader.defineClass1(Native Method)
        at java.lang.ClassLoader.defineClass(ClassLoader.java:648)
        at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
        at java.net.URLClassLoader.defineClass(URLClassLoader.java:272)
        at java.net.URLClassLoader.access$000(URLClassLoader.java:68)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:207)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:201)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:200)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:325)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:296)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:270)
        at sun.launcher.LauncherHelper.checkAndLoadMain(LauncherHelper.java:406)

一开始以为是本地jdk版本过高导致的，但是使用jdk1.7编译依然是同样的问题

报错原因其实很简单，是因为没有加载正确的配置导致，在shell脚本前加上
> source /etc/profile

让配置立即生效即可