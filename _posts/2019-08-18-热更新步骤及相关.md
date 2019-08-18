---
title: Android-Unity联合开发(一)导出aar包
date: 2019-06-15 9:50:00
categories:
- Java, Android, Unity
tags: Android, Java, Unity
---

# 一. 新建安卓工程并导入相应的SDK
UnitySDK目录：
```
E:\unity3d\Editor\Data\PlaybackEngines\AndroidPlayer\Variations\mono\Release\Classes
```
如果有额外的包一同拷贝至\Libs文件夹下，额外的资源拷贝至main\jniLibs(没有此文件夹自行创建)
然后右键Add to modual 记得检查是否都传入

# 二. 创建new Activity
delete layout下main activity文件，删除MainActivity中对应的语句。

# 三. 修改AndroidMainFest文件
修改label 
添加<meta-data android:name="unityplayer.UnityActivity" android:value="true"/> 
修改继承自activity extends UnityPlayerActivity 
有时候需要添加权限 

# 四. 导出aar包
导出的文件一共两个一个是aar包另一个是AndroidManiFest文件 
完成代码编写后 build make moduals 
在build\output\aar文件夹下找到对应的aar包 

# 五. 改装aar包 
覆盖libs的classes包 
删除Manifest的label 

# 六. 修改manifest文件
拷贝出 build\manifests\fill\debug\AndroidManiFest 
修改包名为工程名 无大写 

# 七. 在Unity中调用
将文件拷至unity工程目录Plugins-Android 
```
AndroidJavaClass jc = new AndroidJavaClass("com.unity3d.player.UnityPlayer");//拿到类
AndroidJavaObject jo = jc.GetStatic<AndroidJavaObject>("currentActivity");//拿到Activity
jo.Call("ShowToast","Send from Unity");//调用相应的类
```
# 八. 备注
以后还是要去看看Java和Android相关
