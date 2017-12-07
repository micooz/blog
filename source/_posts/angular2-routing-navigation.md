---
title: Angular2 - Routing & Navigation
date: 2016-02-15 00:00:00
tags: tech
---

`@routerCanActive` 在加载组件前执行，其回调函数有两种返回方式：

    @routerCanActive(function() {
      // return true; 同步
      // return Promise.resolve(true); 异步
    })
