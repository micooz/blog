---
title: How to disable text selection in svg
date: 2016-07-26 00:00:00
tags: tech
---

In sass style:

```css
text {
  user-select: none;
  
  &::selection {
    background: none;
  }
}
```
