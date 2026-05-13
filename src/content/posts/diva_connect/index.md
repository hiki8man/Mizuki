---
title: 【待更新】歌姬计划的多押连接线是如何实现的
published: 2026-05-13
pinned: false
description: 看似很简单，实际上并不简单的一个问题
tags: [歌姬计划]
category: 研究
#licenseName: "Unlicensed"
author: hiki8man
#sourceLink: "https://github.com/emn178/markdown"
draft: true
date: 2026-05-13
#image: "./cover.png"
pubDate: 2026-05-13
permalink: "diva_kiseki"
---

在歌姬计划Arcade中，Sega引入了多押Note的概念，玩家需要按下多个对应的按键触发判定得分。也许是考虑到辅助玩家读谱，Sega在游戏中将多押Note图像变得更亮并且用连接线连了起来  
而这也将成为后续尝试复现原作逻辑的同人游戏开发者的噩梦……

## 只是将点连起来成为一个多边形有这么难吗？ 
也许你会觉得，我们只需要把多个点**按顺序连接起来就可以了**
对于两个点，我们只需要将其直接连接起来就行
```python
if len(multi_note) == 2:
    draw_connect_line(multi_note[0], multi_note[1])
```
三个点也不复杂，只需要按顺序进行连接就可以确保每个点都能两两相连形成三角形  
```python
if len(multi_note) == 3:
    draw_connect_line(multi_note[0], multi_note[1])
    draw_connect_line(multi_note[1], multi_note[2])
    draw_connect_line(multi_note[2], multi_note[0])
```

## 四点构造多边形问题
对于给定的

## 那么Diva是如何实现的呢？  
很遗憾，**我们目前并不知道Diva究竟使用了什么办法**  
在测试下Diva官作有可能产生自交的多边形，我个人比较怀疑Diva的算法本身仍然是基于矩形去设计，目前仍在实验中    

## 其他可行的算法

### ProjectDxxx：上下凸包法
这个……
然而细心的读者可能已经发现了，凸包算法是为了在一堆点钟寻找能够包起来的多边形，而如果为一个凹多边形，凸包算法将会舍弃掉凹下去的点形成一个三角形，多出来的点会没有任何连接孤零零的显示在那。  
因此KHC选择了一种更简单粗暴的办法：如果存在被舍弃的点，那么这个点将会**直接与其他点相连。**  
这样的做法很简单暴力，但考虑到四个点构成的多边形大概率是凸多边形，这种少数情况不进行严格处理也在情理之中

### ComfyStudio：极角排序法  
这个算法也被应用于其他同人项目中，

### 单调链全点链接
