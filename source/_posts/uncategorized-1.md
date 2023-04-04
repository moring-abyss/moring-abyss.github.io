---
title: How to disable predictive scrolling
date: 2021-12-22 22:00:00
---

最近做一个全屏滚动的效果，类似 fullPage，之所以没用一个因为 fullPage 的开源协议是 GPL，另一个是需求上没有 fullPage 那么多兼容支持，就想着自己写一个。

想法挺好，大致的实现思路也简单：

1. body 或者外层套一个 div 设置高宽 100vw,100vh，并且禁止溢出
2. 每一屏一个容器 container，并且使用一个 wrapper 容器包裹起来
3. 初始化时设置 container 的高度为当前窗口高度，宽度为 100%
4. 监听 wheel 事件，每次发生滚动时，记录下当前滚动到第几屏，计算 y 值，设置 wrapper 的 transform 让页面上下滚动起来

> Demo 不考虑兼容性，实战的同学看下实现思路即可

大致的效果：<a href="https://moring-abyss.github.io/example/1.html" target="_blank">页面 Demo</a>，代码可以直接查看网页源代码

这里遇见个问题，无论是 wheel 还是 scroll 事件，滑轮轻轻的滚动一次都会造成 n 次的回调被执行,我们期望的结果是在滚动完成之前只执行一次，严格来说回调执行几次都不 care，最关键的发生滚动的步骤只能执行一次

最开始的思路挺多，因为可以预设 transition 动画的执行时间，有了这个时间有很多中解决方法，例如节流函数，或者 setTimeout or 监听 transitionend 事件

最后选择了 fullPage 一样的方式，设置滚动状态，通过 setTimeout 重置的方式

我以为我解决了这个问题，直到我用触控板触发滚动滑动....

> 正题来了 How to disable predictive scrolling

预测滚动，即当手指离开触控板后滚动仍会持续一段时间，查询国内外文章，大部分都指向 stackoverflow 的这个[问题](https://stackoverflow.com/questions/34831120/disable-predictive-scrolling-mousewheel-onscroll-event-fires-too-often-touc)

<b>简单概括一下就是：目前没有办法解决</b>

> 触控板不会触发 touch 事件，实验性质的 PointEvent 目前支持的也不好，单指和双指点按滑动的触发事件也解决不了这个问题

然后我发现 fullPage 没有这个 bug，于是我决定扌...额，学习一下大佬怎么解决的...

```javascript
if (canScroll) {
  if (isScrollingVertically && isAccelerating) {
    /* doing scroll */
  }
}
```

以上是关键代码,canScroll 的性质和我的思路一致，解决预测性滚动的关键点在于内部的 if 判断上。

isScrollingVertically:用来做是否垂直滚动的判断的：

```javascript
var horizontalDetection =
  typeof e.wheelDeltaX !== "undefined" || typeof e.deltaX !== "undefined";

var isScrollingVertically =
  Math.abs(e.wheelDeltaX) < Math.abs(e.wheelDelta) ||
  Math.abs(e.deltaX) < Math.abs(e.deltaY) ||
  !horizontalDetection;
```

isAccelerating 的计算逻辑：
```javascript
let scrollings = []
let prevTime = new Date().getTime()
function wheel(e) {
  if (scrollings.length > 149) {
    scrollings.shift();
  }

  var value = e.wheelDelta || -e.deltaY || -e.detail;
  var curTime = new Date().getTime();
  scrollings.push(Math.abs(value));
  // 上面这段逻辑很好理解，记录本次（包括本次）之前的150次的滑动距离

  var timeDiff = curTime - prevTime;
  prevTime = curTime;
  if (timeDiff > 200) {
    scrollings = [];
  } 
  // 这段逻辑是用来处理当本次事件执行与上次执行超过200ms时就可以认为这是第二次鼠标/触控板滑动触发
  // Q：200这个时间怎么定的，不太理解

  // getAverage函数是从scrollings中取出指定索引往后的所有元素的平均数
  // 那整个逻辑就好理解了，fullPage的作者通过记录前150次的滚动距离，并通过计算最后70次的平均滚动距离和最后10次的平均滚动距离，如果最后10次的不小于最后70次说明非预测性滚动
  var averageEnd = getAverage(scrollings, 10);
  var averageMiddle = getAverage(scrollings, 70);
  var isAccelerating = averageEnd >= averageMiddle;
  // 特别说明下，不明白的可以试验下，预测性滚动的最后几次滚动距离较小，这种判断方式目前看来没啥问题，贼强
}

function getAverage(elements, number) {
  var sum = 0;
  var lastElements = elements.slice(Math.max(elements.length - number, 1));
  for (var i = 0; i < lastElements.length; i++) {
    sum = sum + lastElements[i];
  }
  return Math.ceil(sum / number);
}
```
<br />
加了这些逻辑的<a href="https://moring-abyss.github.io/example/2.html">Demo</a>效果  
<br /><br />

> ps: resize事件中因需要重新设置子元素高度和transform，如果事先移除transition动画效果会造成页面大幅度的抖动