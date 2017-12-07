---
title: webpack-dev-server 最佳实践
date: 2016-05-10 00:00:00
tags: tech
---

## 简述

[webpack-dev-server](https://github.com/webpack/webpack-dev-server) 是用express和websocket实现的一套在开发环境下前端自动更新的工具。

webpack-dev-server提供CLI接口，读取传入的webpack.config.js配置文件，根据webpack配置，建立一个静态服务器，供前端加载静态资源，其中有一个关键附加脚本是 `webpack-dev-server.js`，位于PATH根路径，即 `/webpack-dev-server.js`，其中存放着websocket客户端。

## 一般使用方法

可以通过下面的命令运行webpack-dev-server：

    $ node node_modules/.bin/webpack-dev-server --config webpack/dev.config.js --inline --profile --colors --watch --display-error-details --display-cached

参数说明参考：[这里](http://webpack.github.io/docs/webpack-dev-server.html#webpack-dev-server-cli)

执行后会自动运行webpack进行打包等一系列操作。

在webpack配置文件中只需添加一个 `devServer` 配置项即可定义webpack-dev-server的行为：

```js
devServer: {
  port: 3000,
  host: 'localhost',
  historyApiFallback: true,
  quiet: false,
  watchOptions: {
    aggregateTimeout: 300,
    poll: 1000
  }
},
```

在这个例子中，webpack-dev-server会在本地**3000**端口上启动一个静态服务器，服务器serve的目录是webpack的必选配置 `output.path`，这是一个绝对路径。

## 一些问题？

请考虑下面这个问题：

我有一个网站项目，分模块，每个模块是一个node项目，且每个模块可以**独立存在**（启动，调试，运行），它们有些用到了webpack-dev-server。

再次强调每个模块相互独立，它们之间的耦合方式只有一种：**请求代理**。

现在假设模块A作为API服务器，监听3000端口；模块B作为应用服务器，要提供资源给浏览器，于是用webpack-dev-server在端口3001的 `/` 上建立了静态服务器。模块B还要从模块A存取数据，那么必定存在从3001跨域请求到3000的问题，消除这个问题有多种解决办法：

1. 在A上设置 `Access-Control-Allow-Origin` 为B的域。
2. 在A、B上层建立代理服务器，屏蔽端口限制。

不深入讨论上面的方法，现在假设我们**采用方法二**解决了跨域请求问题，然后我们再考虑一下接下来的一个问题：

假设存在模块C，和B十分类似，也属于应用服务器；如果B和C存在同名资源，比如 `main.js`，访问该资源就会引发冲突，因为两个模块都在 `/` 上建立了静态服务器，而这又符合每个模块可以**独立存在**的先决条件：

	// B
	http://localhost/B/index.html
    http://localhost/main.js
    // C
    http://localhost/C/index.html
    http://localhost/main.js // 哪个 main.js ?

解决办法看似很明显：

	// B
	http://localhost/B/index.html
    http://localhost/B/main.js
    // C
    http://localhost/C/index.html
    http://localhost/C/main.js // everyone is happy

但这又破坏了每个模块的独立性，我希望单独启动C时，C总能从 `/` 上获取资源，而不是 `/C/...` 这么冗余。

## 最佳实践

问题就出在 `webpack-dev-server`，它适合作为**静态资源服务器**，而不是**开发服务器**。因此，我们的开发环境除了需要 `webpack-dev-server`，还需要专门的**开发服务器**。

    // => Module B
    // dev server
    http://localhost/B/index.html
    // webpack-dev-server for B
    http://localhost:3001/...
    
    // => Module C
    // dev server
    http://localhost/C/index.html
    // webpack-dev-server for C
    http://localhost:3003/...

每个模块从对应的 `webpack-dev-server` 获取资源，解决了冲突又保留了每个模块的独立性。
