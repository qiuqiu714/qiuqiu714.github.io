---
title: MatchTarget使用方法
date: 2019-05-24 11:20:00
categories:
- Unity
tags: Unity, C#
---

# 一. MatchTarget作用
MatchTraget是Animator中的一个方法，可以自动匹配Avatar的某一部位(手、脚、根节点等)到目标点，是的动画效果更合理。


# 二. MatchTarget参数和使用方法
```
public void MatchTarget(Vector3 matchPosition, Quaternion matchRotation, AvatarTarget targetBodyPart, MatchTargetWeightMask weightMask, float startNormalizedTime, float targetNormalizedTime = 1);
```
matchPosition        需要匹配到的目标点的位置坐标  
matchRotation        需要匹配到的目标点的位置旋转  
targetBodyPart       需要匹配的骨架的部位  
weightMask           匹配坐标的权重 new MatchTargetWeightMask(new Vector3(1,0,1), 0) 这里表示x,z坐标权重为1，z和旋转坐标权重为0  
startNormalizedTime  起始的单位时间  
targetNormalizedTime 结束的单位时间  

# 三. 使用中遇到的问题
## (1) 匹配开始和结束的时间问题
这次用到MatchTarget方法是为了使人物翻墙时手能够按在墙的上部，刚开是使用整段动画进行匹配，发现无论怎么调整效果都不理想，后来通过查找资料发现不能使用整段动画进行匹配，targetNormalizedTime的时间点应该是人物翻墙时手撑墙动作的那几帧，这样MatchTarget能够将手很好的匹配到墙上。
## (2) 如果动画在转换状态时MatchTarget不会生效
刚开始时，在连续翻墙时Untiy总会报警告  
    ```
    Calling Animator.MatchTarget while in transition does not have any effect.  
    UnityEngine.Animator:MatchTarget(Vector3, Quaternion, AvatarTarget, MatchTargetWeightMask, Single,Single)  
    PlayerMovement:processVault() (at Assets/Scripts/PlayerMovement.cs:58)  
    PlayerMovement:Update() (at Assets/Scripts/PlayerMovement.cs:31)  
    ```  
意思是在动画转化状态时,MatchTarget不会生效，因此在判定条件中加了一句  
    ```
    anim.IsInTransition(0) == false
    ```
