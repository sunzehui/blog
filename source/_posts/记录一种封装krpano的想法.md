---
title: 记录一种封装krpano的想法
tags:
  - krpano
  - vue
abbrlink: f217d560
date: 2023-01-30 06:09:13
---

最近在vue中使用krpano去实现展示热点，切换场景，一直在思考一种方便获取krpano实例，并且可以对切换场景的时机精准把控（因为场景没切过去热点添加不了），最近翻阅同类框架找到了一种似乎可行的方案，暂时记录一下，现在是设计阶段。

<!--more-->

## 场景简介

先看一下krpano的框架性质吧。

类似百度地图、threejs，会通过js绑定一个div，向其添加一些诸如canvas、div元素，展示场景。

同它们一样，可以添加热点，但是热点需要等地图加载完毕后添加。

对于krpano的组件库我希望：

- 顶层元素也就是如Map Canvas这些可以为下层提供实例，Map实例或是场景、相机等，在krpano这里只需要krpano的实例。

- 热点添加时可以为其指定分组，方便后续切换页面显示对应分组。

- 热点可以绑定点击事件，事件的处理函数必须可以访问到组件内的作用域。

- 热点可以自定义样式，支持预设，甚至使用krpano的语法完全自定义。
- 添加热点时可以等待某个场景加载完毕后添加，保证不存在添加热点在场景切换之前，导致添加热点无反应。

- 所有在scene上的热点可以通过自定义组件的方式写在页面组件上。





## 市场调研

最近看threejs发现，添加场景、添加摄像机、添加材质之类的操作，完全是通过js命令式的一步一步添加上去，这若是搬到vue里，看起来就和原生写法没什么区别了，对于vue这种声明式编程写法，可见即所得的template来说，简直是”返祖行为“！

好在有一个组件库（或者干脆叫轮子）能够声明式地添加物体和材质等等，不过是react写的。

[GitHub - pmndrs/react-three-fiber: 🇨🇭 A React renderer for Three.js](https://github.com/pmndrs/react-three-fiber)

这是示例代码

```tsx
import { createRoot } from 'react-dom/client'
import React, { useRef, useState } from 'react'
import { Canvas, useFrame } from '@react-three/fiber'

function Box(props) {
  // This reference gives us direct access to the THREE.Mesh object
  const ref = useRef()
  // Hold state for hovered and clicked events
  const [hovered, hover] = useState(false)
  const [clicked, click] = useState(false)
  // Subscribe this component to the render-loop, rotate the mesh every frame
  useFrame((state, delta) => (ref.current.rotation.x += delta))
  // Return the view, these are regular Threejs elements expressed in JSX
  return (
    <mesh
      {...props}
      ref={ref}
      scale={clicked ? 1.5 : 1}
      onClick={(event) => click(!clicked)}
      onPointerOver={(event) => hover(true)}
      onPointerOut={(event) => hover(false)}>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color={hovered ? 'hotpink' : 'orange'} />
    </mesh>
  )
}

createRoot(document.getElementById('root')).render(
  <Canvas>
    <ambientLight />
    <pointLight position={[10, 10, 10]} />
    <Box position={[-1.2, 0, 0]} />
    <Box position={[1.2, 0, 0]} />
  </Canvas>,
)
```

`Mesh`这个组件怎么来的我没细看，总之比直来直去的命令式代码看着舒服多了。

相同的场景同样出现在百度地图中。

其实想想自己在vue写的krpano，还停留在命令式编程中。虽说有了compositionApi，但同样是一团乱麻写在script标签里挨个添加热点，我想做成这种声明式，热点样式、数据、分组等等直接写在template中。

好在vue3有一份类似将命令式代码转成这种组件化的组件库参考

[yue1123/vue3-baidu-map-gl: 🎉百度地图 GL版 Vue3 组件库，baidu map gl components libary based on Vue3.0 (github.com)](https://github.com/yue1123/vue3-baidu-map-gl)



## 实现思路

相较于命令式编程，声明式编程可读性更高。但封装支持声明式编程的环境更复杂，需要良好的抽象能力，比如现在谈的这个东西。为了代码不至于乱成一锅粥，需要先设计一下模块划分。

### 通信部分

js操作krpano：krpano实例在js环境中是唯一可以操作krpano的通道，但需要在`embedpano`的`onready`回调中拿到`krpano实例`

krpano执行js：krpano执行js代码只能执行全局函数，所以要执行局部函数需要暴露出来，挂到window对象中。

上面说到`krpano`有加载完成回调函数，也就是`embedpano`中的`onready`，但这仅仅是**krpano整体加载完成**。

对于**加载场景完成回调**，第一次加载场景在xml里后续添加上的，也就是说，krpano默认不加载场景，需要在xml的onstart事件中手动加载第一个场景。

```xml
<action name="startup" autorun="onstart">
	if(startscene === null OR !scene[get(startscene)], copy(startscene,scene[0].name); );
	loadscene(get(startscene), null, MERGE);
	if(startactions !== null, startactions() );
</action>
```

所以说我要等待loadscene执行完毕后才能说加载场景完成，此时执行回调。

但是loadscene这个代码不是同步执行的，紧跟在后面的代码并不是加载完毕后执行，而是将loadscene放到异步任务队列了。

所以另想办法，试着在scene的onstart事件处理函数中执行js代码，效果还不错，确实是加载完毕后执行的。

```xml
<!-- 园区概览图 -->
<scene name="yqglt" onstart="js(handleSceneLoad())">
	<!-- ... -->
</scene>
```

这个`handleSceneLoad`是在`index.html`的全局函数。函数里发送事件通知页面，场景加载完成。

但这还不够，至少我认为。现在不知道是哪个场景加载完成，只知道有个场景加载完毕了，或许直接调用函数的时候添加上参数？我想要一种更通用的写法，毕竟所有组件都很依赖场景加载结束事件。还需要再考虑一下。





### 组件结构

参考`vue3-baidu-map-gl`写法，Map组件加载结束后触发init事件，Map子组件如Marker监听父组件init事件，当map组件init之后子组件Marker执行init后触发init事件

map组件向下提供(provide)：Map示例、作为下层组件的`父组件id`、设置Map示例的一些工具函数、获取Map组件配置的工具函数。

```typescript
provide('getMapInstance', () => map)
provide('parentUidGetter', uid)
provide('baseMapSetCenterAndZoom', (_center: { lng: number; lat: number }) => setCenterAndZoom(_center))
provide('baseMapSetDragging', (enableDragging: boolean) => setDragging(enableDragging))
provide('getBaseMapOptions', () => props)
```

示例写法是这样

```vue
<div>
    <Map
         :minZoom="3"
         :zoom='16'
         >
        <Marker
                :position="{ lat: 39.915185, lng: 116.400901 }"
                :icon="{
                       imageUrl: 'https://cdn.jsdelivr.net/gh/yue1123/vue3-baidu-map-gl@0.0.21/docs/public/logo.png',
                       imageSize: {
                       width: 100,
                       height: 100
                       },
                       }"
                enableDragging
                />
    </Map>
</div>
```

Map组件放`<slot/>`承载下层元素，`<div>`作为百度地图展示的容器。

结构如下

```vue
<template>
  <div
    class="baidu-map-container"
    ref="mapContainer"
  >
    <slot name="loading" v-bind:isLoading="!initd">
    </slot>
  </div>
  <slot></slot>
</template>
```



上面看完了百度地图的写法，我想krpano应该类似。

只是考虑切换页面要不要切换另一个`krpano`实例，毕竟写在一团不太好，分离出去又不方便调用Api。

对于切换场景，可以考虑热点外边加个`<Scene>`标签，但这要考虑krpano能不能动态操作Scene标签，毕竟一些资源路径都是得配置。

