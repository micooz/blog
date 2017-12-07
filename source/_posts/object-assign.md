---
title: Object.assign
date: 2016-02-17 00:00:00
tags: tech
---

    const _new = Object.assign(target, {...});

会改变 `target` 本身，引发问题。

> The Object.assign() method is used to copy the values of all enumerable own properties from one or more source objects **to** a target object. It will return the target object.

有三个解决办法：

**逐个拷贝**

    const _new = {k1: target.k1, k2: target.k2, ...};

**ES6**

    const _new = {...target, ...{...}};

**（最佳）把第一个参数设成空对象**

    const _new = Object.assign({}, target, {...});
