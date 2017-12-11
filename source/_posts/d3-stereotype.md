---
title: D3绘图套路
date: 2016-06-16 16:46:12
tags:
  - tech
  - d3
---

> 接触D3三天，发现有些套路可以反复运用，所以记录一下。

`D3`相比其他图表库，学习成本较高。但最为灵活，需要使用者精雕细琢图形的每个细节。D3处处体现了**函数式**的编程思维。

**NOTE:** 这里所用D3版本是：

```
""dependencies"": {
  ""d3"": ""^4.0.0-alpha.49""
}
```

下面以一个简单的横纵坐标图为例。

## ① 定义高宽

```js
var margin = {left: 70, top: 20, right: 20, bottom: 50};

var svgWidth = 800;
var svgHeight = 500;

var graphWidth = svgWidth - margin.left - margin.right;
var graphHeight = svgHeight - margin.top - margin.bottom;
```

这一步实际上比较重要，图形高宽参数会在后面绘图中常常用到。

## ② 创建svg

```js
var svg = d3.select('.container').append('svg')
    .attr('width', svgWidth)
    .attr('height', svgHeight);
```

这里可以直接在html中放一个`<svg>`，但为了可移植性，用脚本生成。

## ③ 创建绘制区域<g>

```js
var graph = svg.append('g')
    .attr('width', graphWidth)
    .attr('height', graphHeight)
    .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')');
```

`<g>`中将包含所有图形元素（坐标轴、标签、线条……），现在svg树是这样的：

```html
<svg>
  <g></g>
</svg>
```

## ④ 设置X、Y的图形坐标范围

```js
var x = d3.scaleTime()
    .range([0, graphWidth]);

var y = d3.scaleLinear()
    .range([graphHeight, 0]);
```

这个时候由于数据还没获取，只能先设置他们的**图形坐标**范围，注意这个**不是**数据的定义域。

这个两个`函数`通常有两个用途，以`x`为例：

```js
var px = x(data); // 根据数据值计算对应的x坐标值
var data = x.invert(px); // 根据x坐标值反算对应的数据值
```

## ⑤ 获取数据

一般情况下都是从远端取回特定格式的数据，这是个异步过程：

```js
d3.csv('data.csv', parser, function(err, data) {
  // data
})
```

D3很人性化的给你留了个格式化数据的地方`parser`。

```js
function parser(d) {
  return {
    date: d3.timeParse('%b %Y')(d.date),
    value: +(d.price)
  };
}
```

D3取出数据的每一行，依次传入该函数，然后可以返回你需要的格式化数据，最后以数组形成出现在`data`变量中。

## ⑥ 设置X、Y的值域

```js
// value domain of x and y
x.domain(d3.extent(data, function (d) {
  return d.date;
}));

y.domain([0, d3.max(data, function (d) {
  return d.value;
})]);
```

现在可以给x和y设置值域了，值域类型可以很灵活，上面设置x的值域是一个时间范围。

## ⑦ 开始绘制

接下来就是构思你图形的各个部分了，坐标轴、标签、图形内容等等，也是逐步生成一个完整`svg`的过程。为了简便起见，这里不会贴出冗余代码。

绘制X轴：

```js
// X、Y轴可以利用D3的axisXXXXXX函数简单创建，
// 它会把坐标轴的每个数据标签、包括刻度线都为你生成好
graph.append('g')
    .attr('class', '.x-axis')
    .attr('transform', 'translate(-1,' + graphHeight + ')')
    .call(d3.axisBottom(x));
```

绘制折线：

```js
// 创建一个线段生成器，会根据绑定数据，通过x和y访问器计算每个点的坐标位置
var line = d3.line()
    .x(function (d) {
      return x(d.date)
    })
    .y(function (d) {
      return y(d.value)
    });

// 折线图可以用svg中的<path>并设置其d属性来绘制
// line函数需要一个点集来生成一个字符串，这个字符串可以直接填充<path>的d属性
// 为了使线段可见，还需要设置其stroke和stroke-width样式。
var path = graph.append('path')
    .attr('d', line(data));
```

至此，就可以看到一个基本图形了。
