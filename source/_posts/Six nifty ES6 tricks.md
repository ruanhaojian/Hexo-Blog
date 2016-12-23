---
title: 【译】Six nifty ES6 tricks
date: 2016-12-23 15:40:59
tags: ES6
---

> 六个ES6特性技巧 

<!--more-->

# 1、通过参数默认值强制参数

ES6参数默认值是在你实际使用时候来求值的，所以需要你实际强制提供一个默认值。

```javascript
	/**
	 * Called if a parameter is missing and
	 * the default value is evaluated.
	 */
	function mandatory() {
	    throw new Error('Missing parameter');
	}
	function foo(mustBeProvided = mandatory()) {
	    return mustBeProvided;
	}
```

默认值方法 mandatory() 只会在没有设置参数调用时执行，所以输出结果是

```
> foo()
Error: Missing parameter
> foo(123)
123
```

# 2、通过 for-of 遍历一个数组的下标和元素

我们通过 forEach() 去迭代遍历一个数组中的元素：

```javascript
    var arr = ['a', 'b', 'c'];
    arr.forEach(function (elem, index) {
        console.log('index = '+index+', elem = '+elem);
    });
    // Output:
    // index = 0, elem = a
    // index = 1, elem = b
    // index = 2, elem = c
```
ES6中的 for-of 是支持 ES6 迭代(提供 可迭代对象 和 迭代器)和解构功能，如果你把解构与新的阵列的方法entries()组合使用：

```javascript
const arr = ['a', 'b', 'c'];
for (const [index, elem] of arr.entries()) {
    console.log(`index = ${index}, elem = ${elem}`);
}
```
`arr.entries()` 返回一个对应的指标元素，解构[index, elem]提供我们直接访问每对解构值。

# 3、简单的模板提供字面量模板

ES6字面量模板比起文本模板，更像是一个字符串字面量模板，但是你可以使用它们返回一个模板：

```javascript
 const tmpl = addrs => `
        <table>
        ${addrs.map(addr => `
            <tr><td>${addr.first}</td></tr>
            <tr><td>${addr.last}</td></tr>
        `).join('')}
        </table>
    `;
```

使用模板

```javascript
const data = [
    { first: '<Jane>', last: 'Bond' },
    { first: 'Lars', last: '<Croft>' },
];
console.log(tmpl(data));
// Output:
// <table>
//
//     <tr><td><Jane></td></tr>
//     <tr><td>Bond</td></tr>
//
//     <tr><td>Lars</td></tr>
//     <tr><td><Croft></td></tr>
//
// </table>
```









