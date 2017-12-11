---
title: 【记坑】关于d3.zoom
date: 2016-06-14 14:28:57
tags:
  - tech
  - d3
---

> D3.js的最新v4版提供了d3.zoom模块，用它可以给svg图形增加缩放/移动的特性，但通过非鼠标/触控的方式改变图形的位置和缩放比例后，d3.zoom的行为就变得不正常了。本文给出一个解决方法。

```js
// 1. 初始化一个zoom行为
const zoom = d3.zoom();

// 2. 设置zoom行为参数
zoom.scaleExtent([.5, 5]);
...

// 3. 添加三个事件监听器
zoom.on('start', () => {...});
zoom.on('zoom', zoom);
zoom.on('end', () => {...});

// 4. 应用行为
g.call(zoom);
```

我们一般会关注`zoom`事件：

```js
const zoom = () => {
  // d3自动计算的transform值会放在d3.event里
  container.attr('transform', d3.event.transform);
};
```

有趣的是，如果我们在其他地方主动设置：

```js
container.attr('transform', 'translate(10, 10)');
```

再用鼠标或者触控板调整图形时，zoom行为并不会从(10, 10)位置开始计算下一个`d3.event.transform`，zoom行为似乎`自己保存`了上一次的transform，而不关心我们设置到`container`上的transform。

结果就是我们用attr设置的transform其实是`临时的`。

如果查看应用了zoom行为的宿主元素（这里是`g`）的属性，会发现其Element上有一个`__zoom: Transform`，这个玩意儿每次走`zoom`方法的时候都会变化，这正是zoom自己保存的transform对象。

所以我们在主动设置transform后，只需要想办法更新这个__zoom就可以`骗过`zoom行为，让它下一次调用zoom()事，从我们设置的值开始计算。

查阅文档：

https://github.com/d3/d3-zoom/blob/master/README.md#zoom_transform

https://github.com/d3/d3-zoom/blob/master/README.md#transform_scale

可以这样设置：

```js
// 自定义transform
container.attr('transform', 'translate(10, 10)');

// 初始化一个空的Transform对象，这一点文档没说明怎么构造一个Transform对象
const transform = d3.zoomTransform(0)

// 填充{x, y}
    .translate(10, 10);

// 注意一定要设置在.call(zoom)的元素上，它才有'__zoom'
d3.zoom().transform(g, transform);
```

之后d3触发zoom回调之前会取g的__zoom计算下一次的transform，这个transform才是我们想要的。
