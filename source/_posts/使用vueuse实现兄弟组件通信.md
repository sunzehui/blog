---
title: 使用vueuse实现页面间通信
date: 2022-12-09 21:33:55
tags:
- vue
- coding
---

兄弟组件之间通信一直是令人头疼的问题，他们俩彼此之间关系不大，只能共享出一块空间，彼此生产消费......

<!--more-->

如图，这是一个详情页，该页面由两个板块构成，一个是基本的信息页面组件，在views目录中；另一个是提取出来所有的详情页的footer，配合router-view显示页面作整体框架的组件,，在layouts目录中。

![image-20221209215600715](使用vueuse实现兄弟组件通信/image-20221209215600715.png)

他们的代码层级上是这样

```bash
<Layouts>
	<Page />  # <RouterView />
	<Footer />
</Layouts>
```

基本信息的获取是在`<Page />`中拿到，这稀松平常，但现在问题出在`<Footer />`中，可以看到截图上有收藏、分享、加入团队等操作，这些都要拿到基本信息才能继续。

我的思路很简单，搞一个事件总线传递消息，`<Layouts />`做监听，当“收藏”按钮点击时，`Layouts`将点击事件`emit`到`page`中处理，收藏成功后一般要刷新数据，此时将刷新指令emit到`<Page />`中刷新数据；

并且搞一个共享store传递数据，在`Page`基本信息获取完成后将信息存到`store`中，`Layouts`从store中显示收藏按钮状态和其他两个按钮必要信息。

```typescript
// views/detail.vue
const setPageStatus = (type: number) => {
  if (type === -1) {
    pageStore.setState('TEAM', 'UNJOIN')
    return
  }
  else if (type === 0) {
    pageStore.setState('TEAM', 'UNCHECK')
    return
  }
  else if (type === 1) {
    pageStore.setState('TEAM', 'JOINED')
    return
  }
  else if (type === 2) {
    pageStore.setState('TEAM', 'REJECT')
    return
  }
  return pageStore.setState('TEAM', 'UNKNOW')
}
const syncData = (data) => {
  // 设置按钮状态（待申请、已申请、已拒绝）
  setPageStatus(Number(data.user_team))
  pageBus.reset()
  // 监听按钮点击并做相应处理
  useTeamSubmitBtn(props.id)
  usePageStarBtn(props.id, 'team')
  // 向store传递数据
  pageStore.setPageData(data)
}
const authStore = useAuthStore();
const loadData = async () => {
  const id = props.id
  const uid = authStore.user.id
  // 从服务器获取数据
  const data = await ApiGetTeam(id, uid)
  if (!data)
    return
  detail.value = data
  // 将数据传递到store
  syncData(data)
  pageBus.on((evt) => {
    const evtName = evt.name
    if (evtName === 'refresh')
      loadData()
  })
}
```

```typescript
// composables/useTeamSubmitBtn.ts
export const useTeamSubmitBtn = (teamId: string | number) => {
  const handleSubmit = async (type) => {
    const doJoin = async () => {
      return await ApiApplyTeam(teamId)
    }
    const doCancel = async () => {
      return await ApiCancelTeam(teamId)
    }
    let res = null
    if (type === 'join')
      res = await doJoin()
    else if (type === 'cancel')
      res = await doCancel()

    if (res && res.code === 200) {
      Toast.success(res.data)
      // 请求完成后传递refresh，Page页面有监听
      pageBus.emit({ name: 'refresh' })
    }
  }
  
  pageBus.on((e: any) => {
    if (e.name !== 'submit-click')
      return
    // 根据页面状态判断是加入还是退出
    const pageStore = usePageStore()
    const style = pageStore.nowState.style
    let type = null
    if (style === 'btn-place__UNCHECK')
      type = 'join'
    else if (style === 'btn-place__JOINED')
      type = 'cancel'

    if (e.name === 'submit-click')
      handleSubmit(type)
  })
}
```

收藏按钮的处理也差不多，判断是收藏还是取消收藏。

```typescript
// layouts/detail.vue
const { nowState, pageData, pageType, starStatus } = toRefs(usePageStore())

/**
 * @description: 点击后发送事件给详情页面组件，
 * 详情页面组件接收到事件后，根据事件类型进行不同的操作
 * @param evt
 */
const handleSubmit = () => {
  const isBan = [
    'btn-my-project__DONE',
    'btn-my-btn-project__UNCHECK',
  ].includes(nowState.value.style)
  // 灰色按钮下不需要提交
  if (isBan)
    return
  if (!useAuthStore().isLogin) {
    Toast.fail('请先登录')
    router.push('/login')
    return
  }
  pageBus.emit({ name: 'submit-click', data: null })
}
const handleStar = () => {
  const authStore = useAuthStore()
  if (!authStore.isLogin) {
    Toast.fail('请先登录')
    router.push('/login')
    return
  }
  pageBus.emit({ name: 'star-click', data: null })
}
const teamPosterRef = ref(null)
const projectPosterRef = ref(null)
// 分享海报根据不同页面做了不同处理，因为字段名不一样
const showShare = () => {
  const type = pageType
  if (type.value === 'PROJECT' || type.value === 'MYPROJECT') { projectPosterRef.value.show() }
  else if (type.value === 'TEAM') {
    const poster = teamPosterRef.value
    poster?.show()
  }
}
```

在pagestore里导出eventbus

```typescript
import { defineStore } from 'pinia'
import { computed, ref, unref } from 'vue'
import { useEventBus } from '@vueuse/core'
import { get } from 'lodash-es'

export const pageBus = useEventBus('page-event')

enum TeamStateType {
  UNJOIN = '申请加入',
  UNCHECK = '待审核',
  JOINED = '我要退出',
  REJECT = '已拒绝',
}
enum MyProjectStateType {
  UNCHECK = '待录用',
  JOINED = '我要退出',
  DONE = '已结束',
}
enum ProjectStateType {
  CANJOIN = '报名参加',
  JOINDONE = '报名已结束，进行中',
  DONE = '活动已结束',
}

const stateTypeCombine = {
  ...TeamStateType,
  ...MyProjectStateType,
  ...ProjectStateType,
  UNKNOW: 'unknow',
}
type stateType = keyof typeof stateTypeCombine

type pageEnum = 'PROJECT' | 'MYPROJECT' | 'TEAM' | 'UNKNOW'
const stateMap = {
  TEAM: {
    UNCHECK: {
      text: TeamStateType.UNCHECK,
      style: 'btn-place__UNCHECK',
    },
    UNJOIN: {
      text: TeamStateType.UNJOIN,
      style: 'btn-place__UNCHECK',
    },
    JOINED: {
      text: TeamStateType.JOINED,
      style: 'btn-place__JOINED',
    },
    REJECT: {
      text: TeamStateType.REJECT,
      style: 'btn-place__REJECT',
    },
  },
  MYPROJECT: {
    UNCHECK: {
      text: MyProjectStateType.UNCHECK,
      style: 'btn-my-project__UNCHECK',
    },
    JOINED: {
      text: MyProjectStateType.JOINED,
      style: 'btn-my-project__JOINED',
    },
    DONE: {
      text: MyProjectStateType.DONE,
      style: 'btn-my-project__DONE',
    },
  },
  PROJECT: {
    CANJOIN: {
      text: ProjectStateType.CANJOIN,
      style: 'btn-project__CANJOIN',
    },
    JOINDONE: {
      text: ProjectStateType.JOINDONE,
      style: 'btn-project__JOINDONE',
    },
    DONE: {
      text: ProjectStateType.DONE,
      style: 'btn-project__DONE',
    },
  },
  UNKNOW: {
    text: '未知',
    style: '',
  },
}

export const usePageStore = defineStore('pageStore', () => {
  const state = ref<stateType>('UNKNOW')
  const pageType = ref<pageEnum>('UNKNOW')
  function setState(type: pageEnum, newState: stateType, rest?: any) {
    pageType.value = type
    state.value = newState
  }

  const nowState = computed(() => {
    return get(stateMap, [`${unref(pageType)}`, `${unref(state)}`], {
      text: stateMap.UNKNOW.text,
      style: stateMap.UNKNOW.style,
    })
  })

  const pageData = ref(null)
  const setPageData = (data) => {
    pageData.value = data
  }

  const starStatus = computed(() => pageData.value?.user_collection > 0)
  return {
    state,
    setPageData,
    pageData,
    nowState,
    starStatus,
    pageType,
    setState,
  }
})

```

