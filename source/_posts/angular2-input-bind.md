---
title: Angular2 @Input 绑定问题
date: 2016-02-19 00:00:00
tags: tech
---

    @Component({
      template: `<child [value]="bindValue"></child>`
    })
    class HomeComponent {
      bindValue:string = 'hello';
    }
    
    @Component({
      selector: 'child',
      template: `...`
    })
    class ChildComponent{
      @Input() value;
      
      ngOnInit(){
        // 这里可以取到value的值为'hello'
        // 当bindValue动态改变时，此函数不会再次调用，需要在ngOnChanges中手动更新
      }
      
      ngOnChanges(changes) {
        // this.value = changes['value'].currentValue;
        // ...
        // 但是要小心这个函数会被频繁调用，
        // 不要做复杂逻辑
      }
    }

# Array.prototype.map 的坑

map回调如果不给出返回值，则默认返回 `undefined`，[MDN文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map)中似乎并未提到这一点。

    var numbers = [1, 4, 9];
    var roots = numbers.map(n => {
      if (n === 4) {
        return -1;
      }
    });
    // [undefined, -1, undefined]
