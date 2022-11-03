---
title: 在krpano中实现指南针
date: 2022-11-03 14:04:32
tags:
- krpano
---

指南针在全景世界中是很有必要的！

<!--more-->

处理好的全景图以水平坐标0度为正北方向，因此可以通过当前视野中心点水平位置去转动指南针，即全局变量`view.hlookat`。

因为素材是反着的，需要`180 - view.hlookat`回正。所以代码：

```xml
<!-- compass with rotating pointer -->
<layer name="compass1" url="compass_bg.png" align="lefttop" x="10" y="10" children="false" scalechildren="true" destscale="1.0" onclick="switch(destscale,1.0,0.5);tween(scale,get(destscale));">
    <layer name="compass1_plate"   url="compass_plate.png"   align="center"  zorder="1" />
    <layer name="compass1_pointer" url="compass_pointer.png" align="center"  zorder="2" />
    <layer name="compass1_ring"    url="compass_ring.png"    align="lefttop" zorder="3" />
</layer>

<!-- view changing eeent -->
<events name="compass_events" onviewchange="compass_update_rotate();" />

<!-- update the rotation values -->
<action name="compass_update_rotate">
    set(global_heading, 180);
    calc(plugin[compass1_pointer].rotate, view.hlookat - global_heading);
</action>
```

效果：

<iframe src="http://krpano360.com/wp-content/uploads/119pr8/viewer/krpano.html?xml=examples/compass/compass.xml&base=http://krpano360.com/wp-content/uploads/119pr8/viewer/krpano.html?xml=examples/compass/&"></iframe>