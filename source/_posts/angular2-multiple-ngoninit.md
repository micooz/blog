---
title: Angular2 如何多次触发子组件的 ngOnInit
date: 2016-03-29 00:00:00
tags: tech
---

通常子组件加载后只会执行一次`ngOnInit`，不利于子组件的自我更新，但设法使子组件从Dom中移除后重建就可以多次触发`ngOnInit`。

```html
<person *ngIf="show"></person>
```

```javascript
class PersonComponent {
  ngOnInit() {
    // triggered if show is available
  }
}
```

像这种**带星号的指令**就是Angular2中一种模板语法糖，可以管控组件的生命周期。
