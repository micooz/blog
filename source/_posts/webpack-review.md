---
title: webpack复习
date: 2016-03-22 00:00:00
tags: tech
---

使用`ProvidePlugin`暴露对象到全局：

```javascript
plugins: [
  new webpack.ProvidePlugin({
    $: 'jquery',
    jQuery: 'jquery',
    'window.jQuery': 'jquery'
  })
]
```

自定义require返回值：

```javascript
// webpack.config.js

externals: {
  'data': 'data data...'
}

// use in code
var data = require('data'); // 'data data...'
```

**开启Hot Module Replacement(HMR)**

方法一：

    $ webpack --hot --inline

* --hot: 添加HotModuleReplacementPlugin
* --inline: 在生成的js中添加websocket客户端

方法二：

```javascript
// webpack.config.js

entry: {
  'webpack/hot/dev-server', // 仅仅是为window添加一个listener
  'webpack-dev-server/client?http://localhost:8000' // websocket客户端
}

plugins: [
  new webpack.HotModuleReplacementPlugin()
]

```
