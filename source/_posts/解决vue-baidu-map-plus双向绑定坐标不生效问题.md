---
title: 解决vue-baidu-map-plus双向绑定坐标不生效问题
date: 2022-11-01 07:05:21
tags:
- vue
---

基本功能能跑，偶尔多加点小需求这个模块就出问题了，总不能推倒重来吧，试试修复一下。

<!--more-->

vue-baidu-map-plus是从vue-baidu-map上fork出来修改的，这个是用的webgl版本的，比vue-baidu-map原版应该要新一点。所以我直接用的vue-baidu-map-plus。没成想还有些每修改完全的位置，出了点bug。



之前刚装上的时候有个globalthis的问题，我这里vue2.7用的setup拿不到this（或者什么问题的不清楚），导致拿不到Map实例，这个就不说了，在index.html里声明个globalthis的全局变量就好了

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">

<head>
   <script>
    var global = window
  </script>
</head>

<body>
  <div id="app"></div>
  <script type="module" src="/src/main.js"></script>
</body>

</html>
```



## 问题

其实我写着写着出问题后去翻文档，发现文档的示例都有问题，直接讲示例的问题吧，一样的。

先看原版正常的vue-baidu-map的双向绑定

文档链接：[Vue Baidu Map (dafrok.github.io)](https://dafrok.github.io/vue-baidu-map/#/zh/map/baidu-map)

![动画](解决vue-baidu-map-plus双向绑定坐标不生效问题/动画-1667258385930.gif)



再看有bug的vue-baidu-map-plus双向绑定

文档链接：[Vue Baidu Map Plus/GL (ronliruonan.github.io)](https://ronliruonan.github.io/vue-baidu-mapgl/#/zh/map/bmap-gl)

![动画](解决vue-baidu-map-plus双向绑定坐标不生效问题/动画.gif)



问题很明显，后者的双向绑定并没有实现！也就是拖动地图没有实时修改center属性。

猜想是拖动事件没有监听到，导致没有动态修改center。



## 解决

找到`node_modules\vue-baidu-map-plus\components\map\Map.vue`

```javascript
export default {
  name: 'bmap-gl-map',
  methods: {
    init (BMapGL) {
      if (this.map) { return }
      let $el = this.$refs.view
      for (let $node of this.$slots.default || []) {
        if ($node.componentOptions && $node.componentOptions.tag === 'bmap-gl-view') {
          this.hasBmView = true
          $el = $node.elm
        }
      }
      const { minZoom, maxZoom, setMapOptions, zoom, getCenterPoint } = this
      const map = new BMapGL.Map($el, { enableHighResolution: this.highResolution, enableMapClick: this.mapClick, minZoom, maxZoom })
      this.map = map
      setMapOptions()
      // 此处为map绑定事件
      bindEvents.call(this, map)
      map.reset()
      map.centerAndZoom(getCenterPoint(), zoom)
      this.$emit('ready', { BMapGL, map })
    },
  },
}
```

在`bindEvents.call(this, map)`中

```javascript
// node_modules\vue-baidu-map-plus\components\base\bindEvent.js
import events from './events.js'

export default function (instance, eventList) {
  // 关键：this.$options.name == 'bmap-gl-map'
  const ev = eventList || events[this.$options.name]
  
  ev && ev.forEach(event => {
    const hasOn = event.slice(0, 2) === 'on'
    const eventName = hasOn ? event.slice(2) : event
    const listener = this.$listeners[eventName]
    listener && instance.addEventListener(event, listener.fns)
  })
}
```

查看`events.js`里面的事件。

```javascript
export default {
  'bmap-gl': [
    'click',
    'dblclick',
    'rightclick',
    'rightdblclick',
    'maptypechange',
    'mousemove',
    'mouseover',
    'mouseout',
    'movestart',
    'moving',
    'moveend',
    'zoomstart',
    'zoomend',
    'addoverlay',
    'addcontrol',
    'removecontrol',
    'removeoverlay',
    'clearoverlays',
    'dragstart',
    'dragging',
    'dragend',
    'addtilelayer',
    'removetilelayer',
    'load',
    'resize',
    'hotspotclick',
    'hotspotover',
    'hotspotout',
    'tilesloaded',
    'touchstart',
    'touchmove',
    'touchend',
    'longpress'
  ],
  'bmap-gl-geolocation': [
    'locationSuccess',
    'locationError'
  ],
  // ...
}
```

发现并没有`bmap-gl-map`，猜测是作者错写成`bmap-gl`，改成`bmap-gl-map`即可。

回到代码，按照文档写法测试，发现仍有问题

```javascript
const syncCenterAndZoom = (e) => {
  console.log("sync");
  console.log(mapRef.value.map);
  const { lng, lat } = e.target.getCenter();
  console.log(lng, lat);
  nowPosition.value = {
    lng,
    lat,
  };
  nowZoom.value = e.target.getZoom();
};
```

```html
 <baidu-map
      :scroll-wheel-zoom="true"
      :zoom="nowZoom"
      :center="nowPosition"
      class="bm-view"
      @moving="syncCenterAndZoom"
      @moveend="syncCenterAndZoom"
      @zoomend="syncCenterAndZoom"
    >
</baidu-map>
```

![动画](解决vue-baidu-map-plus双向绑定坐标不生效问题/动画-1667259341362.gif)

如图，死循环，重复触发moving事件，最终发现是`e.target.getCenter`导致。

换成从map绑定ref后拿到内部map对象实现，写法

```javascript
// 此处ref应绑定<baidu-map ref="mapRef"/>
const mapRef = ref(null);
const syncCenterAndZoom = (e) => {
    console.log("sync");
    console.log(mapRef.value.map);
	// 删掉
    // const { lng, lat } = e.target.getCenter();
    // 替换为
    const { lng, lat } = mapRef.value.map.centerPoint;
    console.log(lng, lat);
    nowPosition.value = {
        lng,
        lat,
    };
    nowZoom.value = e.target.getZoom();
};
```



## 效果

![动画](解决vue-baidu-map-plus双向绑定坐标不生效问题/动画-1667259739964.gif)

然后我又在想，相较于直接用百度地图的api和用人家封装的这模块，用这个模块的意义在哪里？

不过map-point写起来挺顺手的，百度地图提供的应该是插入dom的写法吧。用这个模块更清晰一点

```html
  <baidu-map
      :scroll-wheel-zoom="true"
      :zoom="nowZoom"
      :center="nowPosition"
      class="bm-view"
      @moving="syncCenterAndZoom"
      @moveend="syncCenterAndZoom"
      @zoomend="syncCenterAndZoom"
      ref="mapRef"
    >
      <MapPoint
        v-if="item.pos"
        v-for="item in mapDataVO"
        :id="item.id"
        :text="item.title"
        :position="item.pos"
        :key="item.id"
        @pointclick="handlePointclick(item)"
      >
      </MapPoint>
    </baidu-map>
```

