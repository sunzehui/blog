---
title: 谈原生JS封装组件
date: 2023-03-11 18:12:02
tags:
- 胡扯
- web
---

在没有组件化框架加持下，如何保证代码的健壮性、封闭性、可拓展性、低耦合性？

脱离框架所带来的便捷，自己编写代码则要更加注意各种可能存在的问题，很考验开发人员水平。

<!--more-->

## 一些常见问题

常见的，写好html后为需要脚本交互的元素绑定事件，而这个绑定事件所使用的选择器是全局的，是从全局往下去找你所选择的元素的，通常我们会忘掉某个选择器写的过于“模糊”，以至于只需要绑定A按钮点击事件，然而却同时绑定了A和B按钮的点击事件，这是不符合需求的，并且是脱离框架后很容易出现的问题。

又或是，元素还未加载，便在onload中手动添加了事件，而这会导致添加失败或是找不到元素的情况，只能在某个元素会出现的那一刻再去手动为其添加事件。相较于Vue中直接在template中使用@绑定事件，并指定当前文件下的函数来说，实在是过于原始。

再者，切换到某个页面后执行一些添加事件函数，元素上绑定好了事件后，再次切换到本页面又重新执行了添加事件函数，则导致事件在元素上被重复添加了两次。通常这种问题可以添加一个是否加载过的状态，如果加载过则不添加。或者是说将添加事件这一操作完全与切换分开，仅仅是该组件加载时执行，再次切换时并不会触发首次加载事件。



## 讨论组件封装的最佳实践

总觉得项目中的组件有些是没必要抽的，有些是需要完全贴合场景去设计的，而不是尽可能通用，毕竟时间不允许，我的意思是顺手就够了。

### 滑块组件

![image-20230311185259337](C:\Users\50157\AppData\Roaming\Typora\typora-user-images\image-20230311185259337.png)

如你所见，滑块组件包括拖动条（slider）、导航按钮（bar）、标签（label）、输入框（input）、指示器（tooltip）

slider直接套的layui-slider，改改样式就能用。其余就没啥好说的，正常编写就行了。

```javascript
/**
 * @description 按照编辑器需求定制的slider
 * @param {string} id editor-target el's id
 * @param { SliderOptions } cfg 
 */
export function useSlider(id, cfg = {}){
  let sliderInstance = null
  let sliderVal = 0 
  layui.use('slider', function(){
    let $ = layui.$
    ,slider = layui.slider;
    // 滑块
    sliderInstance = slider.render({
      elem: '#'+id,
      min:0,
      max:100,
      value: sliderVal,
      ...cfg,
      change(e){
        const $targetInput = $(cfg.inputElScelector || '.slider-input')
        const inputVal = cfg.format ? cfg.format(e): e
        $targetInput.val(inputVal)
        // 额外新增的不需要输入框只做展示的数据
        if(cfg.isText)
          $targetInput.text(inputVal)
        sliderVal = e
        cfg.change && cfg.change(e)
      }
    });
  })
  const step = cfg.step || 1
  // 绑定左右按钮事件
  if(cfg.navigator){
    const $increaseBtn = $(cfg.navigator.increaseBtn || '#'+id+' .editor-slider--right-bar')
    const $decreaseBtn = $(cfg.navigator.decreaseBtn || '#'+id+' .editor-slider--left-bar') 
    $increaseBtn.on('click',function(){
      const nextVal = sliderVal += step 
      sliderInstance.setValue(nextVal); //动态给滑块赋值
    })
    $decreaseBtn.on('click',function(){
      const nextVal = sliderVal -= step
      sliderInstance.setValue(nextVal); //动态给滑块赋值
    })
  }
  

  return {
    getValue(){
      return sliderVal
    }
  }
}
/*
	Usage:
	 const slider = useSlider('editor-slider--target',{
        step: 1,
        min: 0,
        max: 20,
        format:(value)=>{
          return value / 10 + '倍'
        },
        setTips: function(value){ //自定义提示文本
          return value / 10 + '倍';
        },
        inputElScelector:'.animation-setting .slider-input',
        navigator: {
          decreaseBtn: '.animation-setting .editor-slider--left-bar',
          increaseBtn: '.animation-setting .editor-slider--right-bar',
        }
      })
*/
```

仅仅是抛砖引玉，讲一下我自己的思路

layui的slider很好用，生成slider主要是围绕它，设置默认的参数，随后将用户自定义的参数覆盖默认。

当用户拖拽时，修改视图中右上角的输入框，直接改input的value就行了。（但是后面有不需要输入框，只展示数据的label，而label需要使用.text()，这里图省事直接加了标志去判断了）

将拖拽后数据改变事件暴露出去，调用传来的参数change函数。

format和setTips都是可以对当前数据增加人性化输出，而不是呆板的数字。

导航按钮则是通过参数传递选择器字符串过来，依据用户设置的step进行加减操作。



遗憾：

初始传入的slider选择器过于死板了，只能传slider的id，后面修改也不太好改了（涉及很多地方），如果让我优化一下的话，我会传入slider所在的container，仅仅是container不需要传这个slider本体，因为后面同样用到的navigator和input，修改后这里navigator就可以直接从container里面去寻找bar了，而不是在全局去寻找，这样反而要多考虑选择器作用域，增加心智负担。

layui-slider的step不能传入小数，在这种有小数点的情况下输入数字是无效的，仍是按照step=1去调节数值。所以只能扩大10^n倍去解决这个问题。我这里能操作的很少。

## 按钮分组管理组件

老实说这个组件我并没有设计得很好，一些问题仍然存在。

### 介绍需求

![gif](C:\Users\50157\Desktop\gif.gif)

点击创建按钮后，会在下方列表添加一个按钮。

点击创建分组后，会在下方列表添加一个分组，分组下面可以继续添加按钮。

而整个（按钮和分组）总共可以添加4个，分组中的子按钮也只能添加4个。在添加第一个按钮/分组后，首屏的Tips将会消失，同时更新右上角数量信息。

对于分组中的子按钮图标显示，在中间的按钮显示倒着的T，在最下面的按钮显示L。最后一个按钮在不满4个时显示“+创建子按钮”。



乍一看很复杂，其实只需要将按钮创建的逻辑理通就很简单了。



### 思路大概这样：

rxjs可以创建响应式对象，将列表的数据做成响应式，做好监听，每当数据改变时重新渲染页面，从而实现简单的MVVM模式，这里的EditorBtnListCtl便是ViewModel，用于处理view和model的转换，使得我们仅仅关心model，添加按钮和删除按钮只考虑数据变动即可。

#### 定义数据结构？

对于按钮：仅仅包括名字和图标，并附带id方便寻找后删除。

```javascript
{
	id: 1,
	type: 'btn',
	name: '按钮1',
	icon: null
}
```

对于按钮组：按钮组同上，额外多一个children，包含多个按钮。

```javascript
{
	type: 'group',
	name: '分组A',
	id: '1',
	icon: null,
	children: [{
		type: 'btn',
    	id: 1,
		name: '分组A - 1',
	},{
		type: 'btn',
		id: 2,
		name: '分组A - 2',
	}]
}
```

#### 如何渲染？

大致可看这里，将按钮和按钮组分门别类渲染，其renderBtn、renderGroup无非是些拼接字符串而已了。

```javascript
// 将数据渲染到视图
renderView = (viewModel) => {
    const tmpl = viewModel.map(item => {
        if (item.type === 'btn') return this.renderBtn(item)
        if (item.type === 'group') return this.renderGroup(item)
    }).join('')

    this.$list.empty().append(tmpl)
    this.bindEventHandler()
}
```

#### 如何保证数据变化时，重新渲染视图？

使用rxjs.ReplaySubject，与Observable不同，subject可以实现多播，方便暴露外部

```javascript
// 构造函数中注册响应式对象
constructor() {
    this.viewModel = new ReplaySubject(this._viewModel);
    this.viewModel.subscribe((val) => {
        this.renderView(val)
    })
}
```

#### 如何保证删除按钮点击时，可以删除指定的按钮？

这里是我的delBtn实现，略微死板。。这里考虑了按钮和按钮组的id可能会重复。

```javascript
  
  // 入参组id和按钮id
  delBtn(groupId, btnId) {
    const newModel = this._viewModel
    // 查找按钮是否在组里，
    const groupIdx = newModel.findIndex(item => item.id == groupId && item.type == 'group')
    if (groupIdx >= 0) {
      // 如果在，则从该组中找到按钮
      newModel[groupIdx].children = newModel[groupIdx].children.filter(item => item.id == btnId)
    } else {
      // 如果不在，那么该按钮一定作为一个独立个体在外部，直接查该按钮id
      newModel = newModel.filter(item => item.id == btnId)
    }
	// 更新视图
    if (this.viewModel)
      this.viewModel.next(newModel)
  }
```

实际上，曾考虑将id做的像是hashMap一样做成唯一的，直接一步查到，但想想可能会不方便获取id传到后端，算了。

#### 按钮的点击事件绑定和如何拿到组id和按钮id？

```javascript
$('.del-icon').on('click', evt => {
    const item = $(evt.target).parents('.del-icon')[0]
    const itemId = item.dataset.itemid
    const itemType = item.dataset.itemtype
    const groupId = item.dataset.groupid

    if (itemType === 'group')
        return this.delGroup(itemId)
    else if (itemType === 'btn')
        return this.delBtn(groupId, itemId);
})
```

非常硬核的写法，在创建时将组id和按钮id放到dataset中。

另外一种思路，创建按钮时添加删除按钮点击事件处理函数，但仅仅放到viewModel中存储，在创建按钮时很容易拿到组id和按钮id，此处创建的事件处理函数可以拿到局部变量，而在绑定事件中，直接将该按钮所携带的事件处理函数进行绑定即可。

例如另一个组件中写道：

```javascript
renderItem(item) {
    const icon = item.icon || '../assets/image/icon/color-ring.png'
    // 这里保存按钮的点击事件到viewModel上。
    item.handleDelBtnClick = () => {
      this.delItem(item.id)
    }

    return `
    <div class="btn btn-item rounded">
      <div class="btn-item--left">
        <div class="btn-icon">
          <img src="${icon}" alt="${item.name}">
        </div>
        <span class="text-sm">${item.phoneNum}</span>
        <span class="text-xs text-gray phone-friend-name">${item.name}</span>
      </div>
      <div class="del-icon"  data-itemid="${item.id}">
        <svg></svg>
      </div>
    </div>
    `
}

renderView(viewModel) {
    const tmpl = viewModel
    .map(friend => this.renderItem(friend))
    .join('')
    if (this.$list.length)
        this.$list.empty().append(tmpl)
    this.bindEventHandler()
}
bindEventHandler() {
    const listData = this._viewModel
    // 绑定点击事件
    listData.map(item => {
        this.$list.find(`.btn-item .del-icon[data-itemid="${item.id}"]`)
            .click(item.handleDelBtnClick)
    })
}
```













下面附上完整代码：


```javascript
const { ReplaySubject } = rxjs
/**
 * @description 处理渲染html，最终返回html字符串  供外部渲染
 * @param {Array} viewModel
 * */
export class EditorBtnListCtl {
  _viewModel = [
    {
      id: 1,
      type: 'btn',
      name: '按钮1',
      icon: null
    },
    {
      type: 'group',
      name: '分组A',
      id: '1',
      icon: null,
      children: [
        {
          type: 'btn',
          id: 1,
          name: '分组A - 1',
        },
        {
          type: 'btn',

          id: 2,
          name: '分组A - 2',
        },
        {
          type: 'btn',
          id: 3,
          name: '分组A - 2',
        }
      ]
    }
  ]

  constructor() {
    this.viewModel = new ReplaySubject(this._viewModel);
    this.viewModel.subscribe((val) => {
      this.renderView(val)
    })
  }
  // 仅仅处理数据
  id = 233
  createBtn = function (insideGroupId) {
    const newModel = this._viewModel
    const newBtn = {
      type: 'btn',
      name: '按钮1',
      id: this.id++,
      icon: null
    }
    if (!insideGroupId) {
      newModel.push(newBtn)
    } else {
      const findInsertGroupIdx = newModel.findIndex(item => String(item.id) == insideGroupId && item.type === 'group')
      console.log('l', newModel)
      if (findInsertGroupIdx < 0)
        throw '没找到组id'
      newModel[findInsertGroupIdx].children.push(newBtn)
    }
    if (this.viewModel)
      this.viewModel.next(newModel)
  }
  i = 2
  createGroup = function (_newGroup = {}) {
    const newModel = this._viewModel
    const newGroup = {
      type: 'group',
      name: '分组A',
      id: this.i++,
      icon: null,
      ..._newGroup,
      children: []
    }
    newModel.push(newGroup)
    if (this.viewModel)
      this.viewModel.next(newModel)
  }

  delBtn(groupId, btnId) {
    const newModel = this._viewModel
    const groupIdx = newModel.findIndex(item => item.id == groupId && item.type == 'group')
    if (groupIdx >= 0) {
      const btnIdx = newModel[groupIdx].children.findIndex(item => item.id == btnId)
      if (btnIdx >= 0) {
        newModel[groupIdx].children.splice(btnIdx, 1)
      }
    } else {
      const btnIdx = newModel.findIndex(item => item.id == btnId)
      if (btnIdx >= 0) {
        newModel.splice(btnIdx, 1)
      }
    }

    if (this.viewModel)
      this.viewModel.next(newModel)
  }
  delGroup(groupId) {
    const newModel = this._viewModel
    const groupIdx = newModel.findIndex(item => item.id == (groupId) && item.type == 'group')
    newModel.splice(groupIdx, 1)

    if (this.viewModel)
      this.viewModel.next(newModel)
  }

  renderLastBtn = function (group) {
    return `
      <div class="btn btn-item">
        <div class="btn-item--left">
          <div class="btn-icon">
            <img src="../assets/image/icon/last.png" alt="">
          </div>
          <div class="text-xs">
            <a href="#" class="link link-primary create-subbtn-link" data-groupid="${group.id}">
              +创建子按钮
            </a>
          </div>
        </div>
      </div>
    `
  }

  renderBtn = (btn) => {
    const icon = btn.icon || '../assets/image/icon/color-ring.png'
    const name = btn.name || '默认名称'
    const type = btn.type // mid | last | btn

    return `
      <div class="btn btn-item ${type === 'btn' ? 'list-item' : ''} rounded">
        <div class="btn-item--left">
          <div class="btn-icon">
            <img src="${icon}" alt="${name}">
          </div>
          <div class="text-xs">${name}</div>
        </div>
        <div class="del-icon" data-itemid="${btn.id}" data-itemtype="btn" data-groupid="${btn.groupId}">
          <svg></svg>
        </div>
      </div>
    `
  }

  renderGroup = (group) => {
    const groupName = group.name || '默认分组'
    const groupIcon = group.icon || '../assets/image/icon/color-ring.png'
    const child = group.children;

    return `
      <div class="btn-group btn-group-list list-item rounded">
          <div class="btn btn-item">
            <div class="btn-item--left">
              <div class="btn-icon">
                <img src="${groupIcon}" alt="${groupName}">
              </div>
              <div class="text-xs">${groupName}</div>
            </div>
            <div class="del-icon" data-itemid="${group.id}" data-itemid="${group.id}" data-itemtype='group'>
              <svg></svg>
            </div>
          </div>
          ${child.map((item, idx) =>
      idx < 3 ?
        this.renderBtn({
          ...item,
          type: 'mid-btn',
          icon: '../assets/image/icon/mid.png',
          groupId: group.id
        }) : this.renderBtn({
          ...item,
          type: 'last-btn',
          icon: '../assets/image/icon/last.png',
          groupId: group.id
        })
    ).join('')
      }
          ${child.length >= 4 ? '' : this.renderLastBtn(group)}
        </div>
    `
  }
  renderView = (viewModel) => {
    const tmpl = viewModel.map(item => {
      if (item.type === 'btn') return this.renderBtn(item)
      if (item.type === 'group') return this.renderGroup(item)
    }).join('')

    this.$list.empty().append(tmpl)
    this.bindEventHandler()
  }
  bindEventHandler() {
    $('.create-subbtn-link').on('click', evt => {
      const groupId = evt.target.dataset.groupid
      this.createBtn(groupId)
    })
    $('.del-icon').on('click', evt => {
      const item = $(evt.target).parents('.del-icon')[0]
      const itemId = item.dataset.itemid
      const itemType = item.dataset.itemtype
      const groupId = item.dataset.groupid

      if (itemType === 'group')
        return this.delGroup(itemId)
      else if (itemType === 'btn')
        return this.delBtn(groupId, itemId);
    })
  }
  mount = (selector) => {
    if (!this.$list) {
      this.$list = $(selector)
    }
    this.renderView(this._viewModel)

  }
  onChange(cb) {
    cb(this._viewModel)
    this.viewModel.subscribe(cb)
  }
}
```













