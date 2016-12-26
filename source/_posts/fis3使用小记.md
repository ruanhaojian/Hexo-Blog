---
title: fis3使用小记
date: 2016-12-26 15:43:42
tags: fis3
categories:
    - 原创
---

> FIS3 是面向前端的工程构建工具。解决前端工程中性能优化、资源加载（异步、同步、按需、预加载、依赖管理、合并、内嵌）、模块化开发、自动化工具、开发规范、代码部署等问题。

<!--more-->

# 安装 （按照项目实际需要）
1.下载安装node
2.设置镜像 npm config set registry "http://registry.npm.taobao.org/"
3.安装fis3  npm install -g fis3
4.安装模块化组件 npm install -g fis3-hook-module
5.安装模块化依赖自动加载插件 npm install -g fis3-postpackager-loader
6.安装替换插件 npm install -g fis3-deploy-replace
7.安装相对路径插件 npm install -g fis3-hook-relative

# 模块化配置

```javascript
/*
****************基础配置****************
{
    mode: 模块化类型(AMD,CDM, CommandJs)
    baseUrl: 基础路径
    path: 配置别名或者路径
}
*/
fis.hook('module', {
    mode: 'commonJs',
    baseUrl: "./modules/",
    paths: {
        api: "common/api/",
        base: "common/base/",
        ...
	}
});
//fis-conf.js
// 启用插件
fis.hook('relative');


/****************模块化设置***************/

/*设置模块目录, 打包时自动包裹define*/
fis.match('/modules/**.js', {
    isMod: true
});

/*设置发布时不产出的文件*/
fis.match('**.{tmpl,txt,md}', {    
    release: false
});
// fis.match('test.html', {
//     release: false
// });
// fis.media("debug").match('test.html', {
//     release: true
// });

/*设置打包时自动处理模块化依赖关系*/
fis.match('::package', {
    // npm install [-g] fis3-postpackager-loader
    // 分析 __RESOURCE_MAP__ 结构，来解决资源加载问题
    postpackager: fis.plugin('loader', {
        resourceType: 'commonJs',
        useInlineMap: true // 资源映射表内嵌
    })
});

/*设置零散资源自动打包*/
fis.match('::packager', {
  postpackager: fis.plugin('loader', {
    allInOne: true
  })
});


/****************打包设置***************/

/*指定文件添加md5戳*/
fis.match('*.{js,css,png,jpg,gif}', {
  useHash: true
});

/*指定文件添加md5戳*/
// fis.match('/views/images/icon/*.*', {
//   useHash: false
// });

/*设置js压缩插件*/
fis.match('*.js', {
  // fis-optimizer-uglify-js 插件进行压缩，已内置
  optimizer: fis.plugin('uglify-js')
});

fis.media("debug").match('*.js', {
  // fis-optimizer-uglify-js 插件进行压缩，已内置
  optimizer: null
}); 


/*设置css压缩插件*/
fis.match('*.css', {
  // fis-optimizer-clean-css 插件进行压缩，已内置
  optimizer: fis.plugin('clean-css')
});



/*设置png图片压缩插件*/
fis.match('*.png', {
  // fis-optimizer-png-compressor 插件进行压缩，已内置
  optimizer: fis.plugin('png-compressor')
});



/****************合并设置***************/
//用 $1, $2, $3 来代表响应的捕获分组。其中 $0 代表的是 match 到的整个字符串。
// (**.js) 让 a 目录下面的 js 发布到 b 目录下面，保留原始文件名。

/*第三方组件合并处理*/
// fis.match('modules/libs/**.js', {
//   packTo: 'modules/libs/libs.js'
// });

/*系统资源合并处理*/
fis.match('/libs/**.js', {
  packTo: '/libs/mod.js'
});


/*公共组件合并处理*/
fis.match('modules/common/**.js', {
    packTo: 'modules/common/common.js'
});

/*样式合并处理*/
fis.match('views/css/**.css', {
    packTo: 'views/css/style.pack.css'
});

fis.match('modules/common/**.css', {
    packTo: 'views/css/common.pack.css'
});


/*图片输出处理*/
fis.match('views/**.{png,jpg,gif}', {
    release: '$0'
});

fis.match('modules/**/(*.{png,jpg,gif})', {
    release: '/views/images/$1'
});

/*html输出到根目录下*/
fis.match('views/(**.html)', {
    release: '$1'
});

/*首页输出到根目录下*/
// fis.match('views/pages/index.html', {
//     release: 'index.html'
// });

// 让所有文件，都使用相对路径。
fis.match('**', {
  relative: true
});
```

# 构建

执行以下命令预览项目
打包
`fis3 release`
打开服务器
`fis3 server open`
启动服务器(不需要预览可不执行)
`fis3 server start`

# 小结

这里fis3希望解决的是模块化开发以及资源打包压缩自动化的问题。比起webpack、gulp、grunt等配置使用起来要再简单一点，一般简单前端项目构建可参考使用。

