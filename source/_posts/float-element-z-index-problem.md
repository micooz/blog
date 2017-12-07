---
title: 浮动元素z-index无效问题
date: 2016-02-18 00:00:00
tags: tech
---

    .view-container {
      float: right !important;
      z-index: 999; /* invalid! */
      
      /* position must be set */
      position: relative;
    }
