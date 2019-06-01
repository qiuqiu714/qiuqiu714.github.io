---
title: 四元数与欧拉角
date: 2019-06-01 9:20:00
categories:
- Unity
tags: Unity, C#
---

# 一. 概述
最近在做一个案例的时候，让游戏物体旋转时直接使用了transform.Angle()，结果物体只能朝着一个方向旋转。后来才想起来以前看API手册时，Angle似乎时没有方向的，因此旋转是用的都是Quaternation(四元数)，所以在此先做一个大概的总结，有些概念或者错误的地方以后再补充和改正。


# 二. 四元数
这块等以后看3D数学的时候再做补充（要学的东西太多了。）

# 三. 欧拉角
欧拉角在Unity里面是一个Vector3对象，但是！！通过transform.rotation获取到的值是一个四元数而不是欧拉角，因此直接赋一个欧拉角的值是通不过编译的。

# 四. 旋转两种方式
旋转的方式有两种，一种是直接对GameObject的欧拉角赋值，赋值方法如下  
```
cube.eularAngles = new Vector3()
```
第二种方式是将欧拉角转化为四元数赋值，转化和赋值方式如下  
```
cube.rotation = Quaternation.Eular(new Vector3())
```

