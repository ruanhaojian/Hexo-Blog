---
title: webpack打包小记
date: 2016-12-23 17:14:15
tags: webpack
categories:
  - 原创
---

> 记录下使用webpack遇到的问题与解决方案

<!--more-->

# 一、使用ES6模块import，打包出来项目体积偏大

假如我们使用ES6模块引用，以我使用mint-ui为例：

```javascript
import { Toast, MessageBox, InfiniteScroll } from 'mint-ui'
```

这时你只希望打包三个组件，可是webpack打包时候会把整个组件库都打包下来。所以，这个时候需要使用到一个babel插件`babel-plugin-component`，它可以帮助你使用require进行引用：

把 
```javascript
import { Button } from 'components'
```
转成
```javascript
var button = require('components/lib/button')
require('components/lib/button/style.css')
```
具体的资源引用是否正确，需要额外配置 `.babelrc or babel-loader` ， 具体文档点[这里](https://www.npmjs.com/package/babel-plugin-component)。

类似的lodash库也有许多模块，提供了分模块的插件 `babel-plugin-lodash`。

