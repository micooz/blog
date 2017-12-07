---
title: 利用索引提升js的执行效率
date: 2016-03-17 00:00:00
tags: tech
---

**问题引入：**

前段时间，有一个任务是需要**频繁**在**大量的数据**集合中**快速定位**并修改某个元素某个字段的值。

数据结构是**数组**，元素的结构可能相当复杂且**乱序**。

**问题分析：**

假定这个数据集如下：

    // array dataset
    [{
      name: 'name1',
      body: {
        metadata: {
          header: {
            id: 1 // unique
          }
        }
      },
      ...
    }]

实际上就是一个**查找算法**问题，假设要从1000条数据中查找id为1的元素，最SB做法是直接遍历整个数据集：

```javascript
for(let ele of dataset) {
  if (ele.body.metadata.header.id === 1) {
    return ele;
  }
}
```

最坏的情况是O(n)，当然也可以使用其他常见的查找算法减少遍历次数，但如果要**频繁**查找，同步操作会导致页面直接卡死。

如果有一张**哈希表**就帮大忙了，不妨先想想下面这个问题：

> 在数据库里，为什么给一个字段加个索引就可以极大提升查询效率（通常情况）？

**解决方案：**

首先理解索引的含义，在js中，数组是线性结构，它的下标可以当成一种索引，通过下标访问元素时间复杂度为O(1)：

```javascript
const db = [1, 2, 3, 4, 5, ...];
const ele = db[2]; // very quick
```

对于一个Object，同样的：

```javascript
const obj = {
  col1: 1,
  col2: 2,
  ...
};
const col2 = obj['col2']; // very quick
const col2 = obj.col2; // very quick
```

再看看最开始的那个问题，如果我们可以：

```javascript
const id = 1;
const ele = dataset[id]; // very quick
```

实现这个效果实际上就要**建立索引**，此时的 `dataset` 显然已经不能是最原始的数组了。当id不是数字的时候，`dataset` 也不能是数组，
那么Object就理所当然地充当js里的HashMap了（ES6中已经有标准的[Map](http://es6.ruanyifeng.com/#docs/set-map#Map)实现）。

编写一个通用的索引创建函数，这个函数可以为一个数组，通过传入的回调函数的返回值创建一个包含所有数据引用的索引对象（Object）：

```javascript
const index = (arr, fn) => {
  let indexes = {};
  for (let it of arr) {
    const key = fn(it);
    if (!indexes[key]) {
      indexes[key] = {};
    }
    indexes[key] = it;
  }
  return indexes;
}
```

函数只需要**遍历一次数据集**来建立索引。

用法：

```javascript
const our_index = _index(dataset, ele => ele.body.metadata.header.id);
/*
{
  "1": {
    name: 'name1',
    body: {
      metadata: {
        header: {
          id: 1 // unique
        }
      }
    }
  },
  "2": {...},
  ...
}
*/
```

有了这个索引 `our_index`，就可以愉快的以**O(1)**的复杂度来访问任意元素，取出的元素是引用，于是也可以直接对原存储空间的数据进行操作：

```javascript
    let ele = our_index[1];
    // operation on ele
    ele.name = '_' + ele.name;
```

**小结**

原生JavaScript不支持Map数据结构，因此可以通过对象来实现；关键在于如何根据需要建立索引，建立索引的字段必须满足**唯一性**。
