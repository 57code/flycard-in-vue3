## 备战2021，仿探探拖拽卡片效果Vue3实现

大帅刚做了一版类似`探探`的`飞卡`效果组件，十分炫酷！

只可惜不是vue3版本，下面带大家看看如何正确搬运到vue3中。

`绝对抄袭，如有不同，纯属巧合`😁


![Dec-18-2020 11-07-03](https://gitee.com/57code/picgo/raw/master/Dec-18-2020%2011-07-03.gif)



### 飞卡原理

核心点有三：`卡片堆叠布局`、`拖动卡片`和`飞卡`

布局主要利用`z-index`和`absolute`定位；

拖动主要利用几个touch事件：`touchstart`,`touchmove`,`touchcancel`,`touchend`；

飞卡主要利用`勾股定理`😁

详情参见[原文](https://juejin.cn/post/6906143905922678797)，不再赘述。



### 组件化

这里抽取组件是核心，先看看`FlyCard`组件template中的结构：

```html
<div>
  <div>
    <div class="card"
         @touchstart="touchStart"
         @touchmove="touchMove"
         @touchcancel="touchCancel"
         @touchend="touchCancel">
      <slot name="firstCard"></slot>
    </div>
    <div class="card">
      <slot name="secondCard"></slot>
    </div>
    <div class="card">
      <slot name="thirdCard"></slot>
    </div>
    <div class="card">
    </div>
  </div>
</div>
```

> 注意这里省略了所有样式，替换所有`view`为`div`，每张卡片预留了`具名插槽`方便外界传入内容进来。
>
> 只有卡片1需要监听事件，最后预留一张空卡等待“上位”😁



那么，使用`FlyCard`组件时，需要使用`v-slot`指令分发内容，来看看`demo-tantan.vue`

```html
<fly-card>
  <template #firstCard>
    <div v-if="cards[0]" class="tantanCard">
      <img :src="cards[0].img" mode="aspectFill" />
    </div>
  </template>
  <!--省略其他几个template-->
</fly-card>  
```

> 这里的`<template #firstCard>`分发内容进去，完整写法应该是`<template v-slot:firstCard>`

> 注意这里使用`vite`，图片`src`是动态设置的，需要做特殊处理，否则不能正常显示：
>
> ```js
> import img1 from "../assets/1.jpg";
> ```
>
> ```js
> cards: [{img: img1}]
> ```



### 逻辑代码拆分

目前FlyCard接近`400`行，不太容易维护了，我们可以用`Composition API`拆分它们。

观察一下不难发现，拖动逻辑只有`卡片1`需要，所以这一部分的数据和逻辑控制是独立的，完全可拆分出来。

<img src="https://gitee.com/57code/picgo/raw/master/image-20201218090053511.png" alt="image-20201218090053511" style="zoom:25%;" />

<img src="https://gitee.com/57code/picgo/raw/master/image-20201218090302103.png" alt="image-20201218090302103" style="zoom:30%;" />



因此创建`use/touch.js`，抽取这部分逻辑代码，思路是：

- 抽取useTouch函数，接收卡片属性和回调函数等
- 响应数据就是上面的left，top这些
- 控制它们的逻辑是touchStart这些
- 组织在一起并导出供外界使用，日后还能复用在其他项目



抽取`useTouch`，接口如下：

```js
function useTouch(props, {
  onDragStart,
  onDragMove,
  onDragStop,
  onThrowStart,
  onThrowDone,
  onThrowFail,
}) {}
```

> 传入卡片属性后面计算逻辑要用到，还要留出事件回调，这样外界可以做一些额外事情：



响应式数据创建

```js
const cardOneState = reactive({
  left: 0,
  top: 0,
  startLeft: 0,
  startTop: 0,
  isDrag: false,
  isThrow: false,
  needBack: false,
  isAnimating: false,
})
```



控制逻辑：替换大量`this.xxx`，类似下面这样：

```js
function touchStart(e) {
  if (cardOneState.isAnimating) return;
  cardOneState.isDrag = true;
  cardOneState.needBack = false;
  cardOneState.isThrow = false;
  // ......
}
```

> 这里有一个例外是`getDistance`方法，这是一个工具方法，外界不需要它，完全可以放到utils中去。



下面是飞卡逻辑和卡片回弹逻辑，它们需要处理另外几张卡的状态

```js
const otherCardsState = reactive({
  left2: 0,
  top2: 0,
  width2: 0,
  height2: 0,
  // ...
});

function resetAllCardDown() {/*...*/}
function resetAllCard() {/*...*/}
function makeCardThrow() {/*...*/}
function makeCardBack() {/*...*/}
```



生命周期钩子处理

```js
import { onMounted } from "vue";

function useTouch() {
  // ...
  onMounted(() => {
    resetAllCard()
  })
}
```



最后导出接口：

```js
return {
  ...toRefs(cardOneState),
  ...toRefs(otherCardsState),
  touchStart,
  touchMove,
  touchCancel,
};
```



重构完成，useTouch()长这样

<img src="https://gitee.com/57code/picgo/raw/master/image-20201218105201015.png" alt="image-20201218105201015" style="zoom:30%;" />

<img src="https://gitee.com/57code/picgo/raw/master/image-20201218105251740.png" alt="image-20201218105251740" style="zoom:30%; " />



### 组件内使用

下面在`FlyCard`里面使用`useTouch`，额外暴露一下`emits`选项，组件`输入输出`更明确。

```js
import useTouch from "../use/touch";

export default {
  props: {},
  emits: [
    "onDragStart",
    "onDragMove",
    "onDragStop",
    "onThrowFail",
    "onThrowStart",
    "onThrowDone",
  ],
  setup(props, { emit }) {
    const touchState = useTouch(props, {
      onDragStart: () => emit("onDragStart"),
      onDragMove: (obj) => emit("onDragMove", obj),
      onDragStop: (obj) => emit("onDragStop", obj),
      onThrowFail: () => emit("onThrowFail"),
      onThrowStart: () => emit("onThrowStart"),
      onThrowDone: () => emit("onThrowDone"),
    });
    return { ...touchState };
  },
};
```

> 可以看到FlyCard组件简洁多了，组件由接近400行缩减至200行



### 敲黑板

重构完成了，这是我们使用vue3 composition api的一次小实践，好处显而易见：

- 我们的组件更简洁、易维护了
- 我们的业务逻辑可复用了
- 我们的代码完全消除了`this`，更有利于支持ts
- 重构过程我们加强了对业务的理解，这些代码都不是我写的，但是我很快就搞清楚了组件真正需要的接口有哪些，哪些方法只是touch内部需要并不需要暴露出去的。



### 思考

大家观察其他卡片的操作代码，不难发现，它们很有规律，应该很容易进一步抽象成更加通用、可复用的逻辑，比如我能不能动态指定卡片的数量，而不是像现在这样写死，这样大大限制了它的通用性。这个留给大家实现，可以给[我的项目](https://github.com/57code/flycart-in-vue3)提pr。

<img src="https://gitee.com/57code/picgo/raw/master/image-20201218112352669.png" alt="image-20201218112352669" style="zoom:30%;" />



### 代码仓库

https://github.com/57code/flycart-in-vue3



### 关注杨村长

关于该案例就说到这里，希望抛砖引玉，引出更多好的内容出现。

我近期的文章（感谢掘友的鼓励与支持🌹🌹🌹）：

- [🔥又是一夜，这篇Composition-API实操还觉得短吗](https://juejin.im/post/6892017198450081800) 198👍
- [🔥拿下vue3你要做好这些准备](https://juejin.im/post/6866373381424414734) 62👍
- [🔥闪电五连鞭：Composition API原理深度剖析](https://juejin.im/post/6894993303486332941) 49👍
- [🔥我的非凡2020 | 掘金年度征文](https://juejin.cn/post/6904058925482672141) 35👍

我的视频教程（感谢掘友的鼓励与支持🌹🌹🌹）：

- [【全网首发】Vue3.0光速上手「持续更新中」](https://www.bilibili.com/video/BV1Wh411X7Xp) 523👍
- [【面霸养成】天天造轮子 (每天一更，建议收藏)](https://www.bilibili.com/video/BV13v411C7VC) 69👍
- [【源码解读】村长vue3源码剖析](https://www.bilibili.com/video/BV1iT4y137yj) 60👍
- [【快乐1024】给程序员同胞在线发老婆！](https://www.bilibili.com/video/BV16z4y1o7Wg)31👍

