---
title: Angular2 事件绑定注意
date: 2016-02-23 00:00:00
tags: tech
---

```html
<select [(ngModel)]="value" (change)="onSelect(value)">
  ...
</select>
```

```javascript
onSelect(value) {
  // value 还是原来的值，没来得及改变
}
```

**解决办法**

```javascript
onSelect(value) {
  setTimeout(() => {
    // value 绑定完成后的值
  });
}
```

# console.log 或者说Chrome DevTools的坑

考虑下面的代码：

```javascript
let obj = {a: []}, n = 100;
while(n--) {
  obj.a.push(n);
}
console.log(obj); // obj.a[50]: -100

obj.a[50] = -100;
console.log(obj); // obj.a[50]: -100
```

浏览器里可以发现两次输出的结果中 `a[50]` 都是 `-100`。这一点如果第一次遇到的话还真是匪夷所思。

这里我故意把 `a` 数组的元素弄得很多，使 `DevTools` 以 **折叠** 方式显示：

> Object {a: Array[100]}
> Object {a: Array[100]}

看似友好的显示方式，实际上里面有很大的问题。

当我们**展开第一个输出**时， `DevTools` 会 **及时** 读取变量值，由于这是个 **引用** 类型，实际上它读到的是 `obj` 的最终值，及 `a[50]` 是 `-100`。

如果数组a只有很少的元素，`DevTools` 不启用智能显示时就不会出现这个问题。

也就是说，`console.log` 到 `DevTools` 里的实际上**是引用而不是拷贝**，**展开**操作会及时读取变量值。

如果把上面例子的两个输出改成：

```javascript
console.log(JSON.stringify(obj));
```

结果将和预期的一致。

**因此**，在浏览器中调试 `js` 程序应该以 `调试器` 下断点为主，日志为辅。
