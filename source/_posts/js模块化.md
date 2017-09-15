---
title: js模块化
date: 2017-02-21 16:31:58
tags: javascript
---

js模块化：AMD、CMD、CommonJS、ES6 Module

<!--more-->

# AMD 规范

定义无依赖的模块

```javascript
define({
	
	add : function(x, y){
		return x + y;
	} 

})
```

定义有依赖的模块

```javascript
define(['jquery'], function($){
	
	function a(){}
	// ...
	return {
		a: a
	}

})
```

定义数据对象模块

```javascript
define({
	users: [],
	members: []
})
```

具名模块

```javascript
define("alpha", [ "require", "exports", "beta" ], function( require, exports, beta ){
  export.verb = function(){
      return beta.verb();
      // or:
      return require("beta").verb();
  }
});
```

包装模块

```javascript
define(function (){
	
})
```


required.js 2.0以后支持CMD写法，不过官方推荐依赖前置

# CMD 规范

CMD是SeaJS 在推广过程中对模块定义的规范化产出

* 对于依赖的模块AMD是提前执行，CMD是延迟执行。不过RequireJS从2.0开始，也改成可以延迟执行（根据写法不同，处理方式不通过）。
* CMD推崇依赖就近，AMD推崇依赖前置。

```javascript
define(function(require, exports, module){

    //依赖可以就近书写
    var a = require('./a');
    a.test();

    ...
    //软依赖
    if (status) {

        var b = requie('./b');
        b.test();
    }

})
```

# UMD

UMD是AMD和CommonJS的糅合

AMD模块以浏览器第一的原则发展，异步加载模块。
CommonJS模块以服务器第一原则发展，选择同步加载，它的模块无需包装(unwrapped modules)。
这迫使人们又想出另一个更通用的模式UMD （Universal Module Definition）。希望解决跨平台的解决方案。

```javascript
(function (window, factory) {
    if (typeof exports === 'object') {

        module.exports = factory();
    } else if (typeof define === 'function' && define.amd) {

        define(factory);
    } else {

        window.eventUtil = factory();
    }
})(this, function () {
    //module ...
});
```

# CommonJS 规范

Node应用由模块组成，采用CommonJS模块规范。[详情](http://javascript.ruanyifeng.com/nodejs/module.html)


# ES6 Module (ruanyf ES6入门 Module)

ES6 模块的设计思想，是尽量的静态化，使得编译时就能确定模块的依赖关系，以及输入和输出的变量。CommonJS 和 AMD 模块，都只能在运行时确定这些东西。比如，CommonJS 模块就是对象，输入时必须查找对象属性。

```javascript
// CommonJS模块
let { stat, exists, readFile } = require('fs');

// 等同于
let _fs = require('fs');
let stat = _fs.stat;
let exists = _fs.exists;
let readfile = _fs.readfile;
```

上面代码的实质是整体加载`fs`模块（即加载fs的所有方法），生成一个对象（`_fs`），然后再从这个对象上面读取3个方法。这种加载称为“运行时加载”，因为只有运行时才能得到这个对象，导致完全没办法在编译时做“静态优化”。

ES6 模块不是对象，而是通过`export`命令显式指定输出的代码，再通过`import`命令输入。

```javascript
// ES6模块
import { stat, exists, readFile } from 'fs';
```

上面代码的实质是从fs模块加载3个方法，其他方法不加载。这种加载称为“编译时加载”或者静态加载，即 ES6 可以在编译时就完成模块加载，效率要比 CommonJS 模块的加载方式高。当然，这也导致了没法引用 ES6 模块本身，因为它不是对象。
由于 ES6 模块是编译时加载，使得静态分析成为可能。有了它，就能进一步拓宽 JavaScript 的语法，比如引入宏（macro）和类型检验（type system）这些只能靠静态分析实现的功能。

除了静态加载带来的各种好处，ES6 模块还有以下好处。

* 不再需要UMD模块格式了，将来服务器和浏览器都会支持 ES6 模块格式。目前，通过各种工具库，其实已经做到了这一点。
* 将来浏览器的新 API 就能用模块格式提供，不再必须做成全局变量或者navigator对象的属性。
* 不再需要对象作为命名空间（比如Math对象），未来这些功能可以通过模块提供。


















