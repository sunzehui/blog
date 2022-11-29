---
title: 使用storybook抽离公共组件
date: 2022-11-30 07:09:04
tags:
- vue
- storybook
- coding
---

项目里一些常用的组件在复用到新项目里，总会遇到各种依赖四处流放的情况，导致我复制代码后，首先解决依赖找不到问题。寻求一种方式可以将组件的粒度缩小，不但包括项目结构方面减少依赖，还要代码灵活性上，让组件可随意配置，storybook看起来不错。

<!--more-->

## 组件介绍

今天刚上手storybook，试着抽一下项目里最简单的组件：`Title`



![image-20221130071616288](使用storybook抽离公共组件/image-20221130071616288.png)

Title 用来展示手机端的标题栏，默认红色背景，白色字体，自带返回图标，点击文字或图标会返回到上一页。

这是我描述的‘组件故事’，在controls里可以对组件的props修改，并且还能收到组件的事件。

![image-20221130071822774](使用storybook抽离公共组件/image-20221130071822774.png)





### 故事编写

storybook是嵌入到已存在的vue项目的，直接在项目根目录运行

```bash
npx sb init
```

storybook会在src里生成stories目录

![image-20221130072136442](使用storybook抽离公共组件/image-20221130072136442.png)

此时跑起来storybook，使用

```bash
yarn storybook
```

在stories里新建`Title.stories.js`，将项目里的`Title.vue`挪到stories里。

这是我项目里的 Title 组件

```vue
<script lang="ts" setup>
    import router from '@/router'

    const props = defineProps<{
        noBack?: boolean
    }>()
    const goback = () => {
		if(props.noBack) return;
        router.back()
    }
</script>
```

首先明显问题`router`这个东西在storybook里跑不了，也就是我挪到新项目里要解决的依赖，若是将Title这个组件挪到新项目里，我还要配置一堆router，想想都头疼。

所以改造一下，换成组件内触发back事件，让调用者去决定back时干啥。

```vue
<script lang="ts" setup>
	const emit = defineEmits(['back'])
    const props = defineProps<{
        noBack?: boolean
    }>()
    const goback = () => {
		if(props.noBack) 
        	return;
        emit('back')
    }
</script>
```

另外，组件的样式是写死的，为了让组件更灵活，我选择使用css v-bind 将props传入css里面。

```vue
<script lang="ts" setup>
import { computed, watch } from 'vue'

const props = defineProps<{
  noBack?: boolean
  bgColor?: string
  fontColor?: string
  fontSize?: string
  fontWeight?: string
  sidePadding?: string
}>()
const emit = defineEmits(['back'])

const bgColor = computed(() => props.bgColor || '#dc3333')
const fontColor = computed(() => props.fontColor || '#ffffff')
const fontSize = computed(() => props.fontSize || '4.26vw')// 32px
const sidePadding = computed(() => props.sidePadding || '3.2vw')// 24px
const fontWeight = computed(() => props.fontWeight || '500')
const goBack = () => {
  if (!props.noBack)
    return
  emit('back')
}
</script>

<template>
  <div class="title">
    <div v-if="!props.noBack" class="icon-wrap" @click="goBack">
      <img src="./assets/back.png" alt="">
    </div>
    <span>
      <slot />
    </span>
  </div>
</template>

<style lang="scss" scoped>
.title {
  z-index: 99999;
  background-color: v-bind(bgColor);
  padding: 0 v-bind(sidePadding);
  display: flex;
  align-items: center;
  position: sticky;
  top: 0;
  left: 0;
  right: 0;
  height: 94px;
}

.title .icon-wrap {
  height: 32px;
  width: 32px;
  cursor: pointer;
  img {
    height: 100%;
  }
}

.title span {
  font-size: v-bind(fontSize);
  font-weight: v-bind(fontWeight);
  color: v-bind(fontColor);
  line-height: 1;
  display: block;
  cursor: pointer;
}
</style>
```

组件写好了，storybook里直接引入后添加点默认配置就完成了

```javascript
import Title from './Title.vue'

export default {
  title: 'Title',
  component: Title,
  name: 'Title',
  // More on argTypes: https://storybook.js.org/docs/vue/api/argtypes
  argTypes: {
    bgColor: {
      name: 'bgColor',
      type: { name: 'string', required: false },
      defaultValue: '#dc3333',
      description: '背景颜色',
      table: {
        type: { summary: 'string' },
        defaultValue: { summary: '#dc3333' },
      },
      control: {
        type: 'color',
      },
    },
    fontColor: {
      name: 'fontColor',
      type: { name: 'string', required: false },
      defaultValue: '#fff',
      description: '字体颜色',
      table: {
        type: { summary: 'string' },
        defaultValue: { summary: '#fff' },
      },
      control: {
        type: 'color',
      },
    },
  },
}

// More on component templates: https://storybook.js.org/docs/vue/writing-stories/introduction#using-args
const Template = (args, { argTypes }) => ({
  props: Object.keys(argTypes),
  components: { Title },
    // $props 是所有传入的变量
  template: '<Title v-bind="$props">{{label}}</Title>',
})

export const TitleCmp = Template.bind({})
// More on args: https://storybook.js.org/docs/vue/writing-stories/args
TitleCmp.args = {
  noBack: false,
  label: '文章详情',
}
```

对，就是这么简单。

