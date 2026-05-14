---
title: 歌姬计划的多押连接线是如何实现的
description: 看似很简单，实际上并不简单的一个问题
author: hiki8man
category: 研究
tags: [歌姬计划]
#sourceLink: "https://github.com/emn178/markdown"
#image: "./cover.png"

published: 2026-05-13
updated: 2026-05-14T09:00:00

pinned: false
draft: false
permalink: "diva_kiseki"
#licenseName: "Unlicensed"
---

在歌姬计划Arcade中，Sega引入了多押Note的概念，玩家需要按下多个对应的按键触发判定得分。也许是考虑到辅助玩家读谱，Sega在游戏中将多押Note图像变得更亮并且用连接线连了起来  
而这也将成为后续尝试复现原作逻辑的同人游戏开发者的噩梦……

## 只是将点连起来成为一个多边形有这么难吗？ 
也许你会觉得，我们只需要把多个点**按顺序连接起来就可以了**
对于两个Note，我们只需要将其直接连接起来就行
```python
if len(multi_note) == 2:
    draw_connect_line(multi_note[0], multi_note[1])
```
三个Note也不复杂，只需要按顺序进行连接就可以确保每个点都能两两相连形成三角形  
```python
if len(multi_note) == 3:
    draw_connect_line(multi_note[0], multi_note[1])
    draw_connect_line(multi_note[1], multi_note[2])
    draw_connect_line(multi_note[2], multi_note[0])
```
正当你兴致勃勃的尝试实现四个Note的连线时，问题开始出现了：  
1. 四个点无法形成一个确定的多边形
2. 我们无法确定程序读取到的数据顺序一定是正确的
3. 随意的连接四个Note有极高的概率出现自交

因此我们需要对Note进行排序

## 那么正确的多押线应该如何实现呢？    
目前，我们**并不能确定Diva究竟使用了什么办法实现这个功能**，各个同人游戏给出的多押实现在特定情况下均会与官作产生差异，但我们仍然可以用其他算法方式去实现  
> [!NOTE] 
> 2025.05.14
> 目前看来Diva可能使用的是单调多边形，具体思路下面会补充

### ComfyStudio：极角排序法   
一种做法是先找出这些点的中点，**以中点为圆心转圈**向四周扫描，按扫描到的顺序对点进行排序后连接。这个方法也被应用于其他同人游戏项目中，效果很不错。  
代码思路如下:
1. 计算各个点的中点
2. 计算各个点到中点的夹角
3. 按夹角重新排序，如果夹角一致则使用距离
4. 按顺序绘制多押线并将结尾连起来

```python
@dataclass
class Vector:
    x:float
    y:float

def sort_notes(multi_note: list[Vector]) -> list[Vector]:
    centorid = Vector(0,0)
    for note in multi_note:
        centorid += note

    centorid /= len(multi_note)

    def sorted_func(note):
        # 返回夹角和距离平方值用于排序
        angle = math.atan2(note.y - centorid.y, note.x - centorid.x)
        distance_square = (note.x - centorid.x)**2 + (note.y - centorid.y)**2
        return (angle, distance_square)

    return sorted(multi_note, key=sorted_func)

def draw_line(multi_note: list[Vector]):
    # 绘制连接线逻辑
    pass


def draw_connect(multi_note: list[Vector]):
    # 按顺序连接
    for i in range(len(multi_note)-1):
        draw_line(multi_note[i], multi_note[i+1])
    # 首尾连接
    draw_line(multi_note[-1], multi_note[0])

def on_same_position(multi_note: list[Vector]) -> bool:
    check_point: None|Vector = None
    for note in multi_note:
        if Vector is None:
            check_point = note
            continue
        
        if check_point.x == note.x and check_point.y == note.y:
            return True
    
    return False

def multi_connect(multi_note: list[Vector]):
    # 多押连接线调用函数
    multi_count = len(multi_note)

    if multi_count < 2:
        # 错误情况，不执行操作
        pass

    elif multi_count < 4 or on_same_position(multi_note):
        # 能确定情况的直接按默认顺序连接
        draw_connect(multi_note)

    else:
        draw_connect(sort_notes(multi_note))
```


### ProjectDxxx：上下凸包法
这个……
[详细算法说明]
然而细心的读者可能已经发现了，凸包算法是为了在一堆点钟寻找能够包起来的多边形，而如果为一个凹多边形，凸包算法将会舍弃掉凹下去的点形成一个三角形，多出来的点会没有任何连接孤零零的显示在那。  
因此KHC选择了一种更简单粗暴的办法：如果存在被舍弃的点，那么这个点将会**直接与其他点相连。**  
这样的做法很简单暴力，但考虑到四个点构成的多边形大概率是凸多边形，这种少数情况不进行严格处理也在情理之中  

### 也许是Diva的做法？
简单思路：
1. 找出一个x最左侧的点，如果有多个则选择y最低点  
2. 找出一个x最右侧的点，如果有多个则选择y最高点
3. 连起来将剩余的点分为上下两个部分
4. 下侧从左往右绕，以x轴变大排序，如果x相同优先y更小的
5. 上侧从右往左绕，以x轴变小排序，如果x相同优先y更大的
6. 最后连起来

特殊的，我们先定义分割线左侧的点为A，右侧的点为C，对于Diva目前观测到的情况来说：
- 如果所有的点都在同一个位置，则按照DSC读取顺序链接成闭合图形
- 其他情况下游戏一定会按照A点 → 下侧点 → C点 → 上侧点 → A点的顺序连接，如果产生空则跳过
- 如果点在分割线上，则将其默认分给上侧

示例代码
```python
@dataclass
class Vectory:
    x:float
    y:float

    def __add__(self, other):
        if isinstance(other, Vectory):
            return Vectory(self.x + other.x, self.y + other.y)
        elif isinstance(other, (int, float)):
            return Vectory(self.x + other, self.y + other)

    def __sub__(self, other):
        if isinstance(other, Vectory):
            return Vectory(self.x - other.x, self.y - other.y)
        elif isinstance(other, (int, float)):
            return Vectory(self.x - other, self.y - other)

def cross_product(p0: Vectory, p1: Vectory, p2: Vectory) -> float:
    line1 = p1 - p0
    line2 = p2 - p1
    return line1.x * line2.y - line1.y * line2.x

def draw_connect(start_point: Vectory, end_point: Vectory):
    pass # 绘制连接线逻辑

def monotone_polygon() -> list[Vectory]:
    pass

def multi_connect(multi_note: list[Vectory]):
    if len(multi_note) == 2:
        draw_connect(multi_note[0], multi_note[1])
        draw_connect(multi_note[1], multi_note[0])
    
    elif len(multi_note) == 3:
        draw_connect(multi_note[0], multi_note[1])
        draw_connect(multi_note[1], multi_note[2])
        draw_connect(multi_note[2], multi_note[0])
    
    elif len(multi_note) > 3:
        pass

    else:
        pass # 传入了错误数据长度的数据

```
