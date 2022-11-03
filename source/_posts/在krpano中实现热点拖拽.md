---
title: 在krpano中实现热点拖拽
date: 2022-11-03 17:26:10
tags:
- krpano
---

上次热点可以做成动画之后，还想让它可以移动，有移动事件，改改坐标的事而已！

<!--more-->

## 分析

krpano最近刚接触，不太熟悉，那先看一下JavaScript是怎么处理的吧。

![img](在krpano中实现热点拖拽/20211026110637647.gif)

转自——[JavaScript实现鼠标拖拽效果_javascript技巧_脚本之家 (jb51.net)](https://www.jb51.net/article/226366.htm)

一个盒子可以拖动，关键在于它是随着鼠标位置拖的，没有位置闪动什么的。

![img](在krpano中实现热点拖拽/2616181-20220512113139629-1726759823.png)

转自——[JavaScript | 事件案例之鼠标拖拽特效 - 一马当先G - 博客园 (cnblogs.com)](https://www.cnblogs.com/ymdx/p/16261816.html)

简单写的话，让div的坐标等于鼠标坐标就好了，但是会导致不跟手，因为鼠标坐标就是正常位置，而div的坐标是从左上角开始算的，也就是上图的offsetLeft、offsetTop。

所以最终移动时要改变的div的位置应该是：

$ x = clientX - unchangeX$

$ y = clientY - unchangeY$

上面表示(鼠标位置)减去(盒子边缘到鼠标位置的间距)，此时的盒子坐标是从左上角计算。

而盒子边缘到鼠标位置的间距：`unchangeX`，`unchangeY`，

在`movestart`时

$unchangeX = clientX - offsetLeft$

$unchangeY = clientY - offsetTop$



经过上面分析，krpano的写法也就确定了，基本一致。

## 实现

```xml
<!-- 脚本 -->
<action name="move">
	<!-- 球面坐标转屏幕坐标，相当于offsetLeft,Top -->
	spheretoscreen(
		ath, atv, hotspotcenterx, hotspotcentery, 
		calc(mouse.stagex LT stagewidth/2 ? 'l' : 'r'));
	<!-- 计算边到鼠标的间距 -->
	sub(drag_adjustx, mouse.stagex, hotspotcenterx);
	sub(drag_adjusty, mouse.stagey, hotspotcentery);
	<!-- 只有按下的时候执行，相当于 move 事件 -->
	asyncloop(pressed,
		<!-- 鼠标位置从左上角计算 -->
		sub(dx, mouse.stagex, drag_adjustx);
		sub(dy, mouse.stagey, drag_adjusty);
		<!-- 转换回球面坐标 -->
		screentosphere(dx, dy, ath, atv);	
	  );
</action>

<!-- 场景 -->
<scene name="scene_1" title="项目全景" onstart="trace(1)" havevrimage="true" lat="" lng="" heading="">
	<!-- ... -->
	<style 	name="hotspot_active"
	  		onloaded="do_crop_animation(89,109, 6)"
	  		ondown="move()"
			url="%SWFPATH%/index/1/active.png" edge="bottom" oy="0"/>
	<hotspot name="active" 
             title="cat" 
             style="hotspot_active" 
             atv="34.120" ath="18.393" scale="3" 
             visible="true" />
</scene>
```

按下的事件，绑定事件处理函数。

action里面的球面坐标atv、ath正是热点的属性，直接拿直接改。



## 效果

完美的拖动效果！

![动画](在krpano中实现热点拖拽/动画.gif)