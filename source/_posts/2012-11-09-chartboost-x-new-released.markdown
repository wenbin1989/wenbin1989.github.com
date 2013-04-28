---
layout: post
title: "Chartboost-x新鲜出炉: C++ wrapper of Chartboost for cocos2d-x"
date: 2012-11-09 19:54
comments: true
categories: [C/C++, cocos2d-x]
keywords: 交叉推广, Chartboost, cocos2d-x, 封装, Chartboost-x, C++, wrapper, github, iOS, Android
description: 游戏中加入Chartboost支持，以实现交叉推广。游戏使用cocos2d-x引擎，逻辑代码由C++编写。为了在C++代码内直接调用Chartboost相关接口，对Chartboost提供的iOS和Android SDK做了C++封装。运行良好，使用简单。
---

前几日团队的手游项目到了交付阶段，运营商要求在游戏中加入[Chartboost](http://chartboost.com/)支持，以实现交叉推广。

该平台的最大特点就是：全屏广告！让玩家欲罢不能，横竖都得点一下-\_-! 虽然对用户体验有一定影响，但据说效果很好，点击率超高，捷报频频。Backflip，Crowdstar, Disney, Gameloft, Get Set Games, GREE, Kabam, Kiloo, Playfirst, Pocket Gems, TinyCo等等大厂都在使用，实乃交叉推广之必备良药。

我们的游戏使用cocos2d-x引擎，逻辑代码由C++编写。为了在C++代码内直接调用Chartboost相关接口，本人对Chartboost提供的iOS和Android SDK做了一个简单的C++封装，方便使用。
最近闲来无事，对相关代码加以完善整理，取名为Chartboost-x，现正式放出，以造福大众！源代码与详细说明请见：

[Chartboost-x GitHub页](https://github.com/wenbin1989/Chartboost-x)

该封装基本覆盖了Chartboost SDK的所有接口，实现了包括Delegate等功能，运行良好，使用简单。如果您碰巧也使用cocos2d-x，碰巧也需要Chartboost，那一定要clone下来看看，或许能节省您不少宝贵时间。
