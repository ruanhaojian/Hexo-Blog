---
title: MobX文档
date: 2017-01-05 17:45:14
tags: mobx
---

> [mobxjs intro](http://mobxjs.github.io/mobx/intro/overview.html)  英文文档。

<!--more-->

# 1.MobX介绍

## 1.1 MobX要点

到目前为止，这一切听起来有点花哨，但使用MobX实现响应时应用只需要以下三步：

**1.定义你的观察者状态**

在任何您喜欢的数据结构中存储状态：objects、arrary、classes。循环数据结构或者关联数据，都没关系。只要确保你想要实现响应观察的状态改变的变量使用MobX来实现观察者。

```javascript
import {observable} from 'mobx';

var appState = observable({
    timer: 0
});
```

**2.创建一个视图来响应你的状态改变**

我们没有让`appState`观察任何东西；你现在可以创建一个任何在`appState`里面的数据改变而自动更新的视图。
通常来讲，任何方法都可以变成观察了数据的响应式视图，并且MobX可以应用在任何使用ES5的环境下。但是，这里有一个来自使用ES6的react组件的视图例子。

```javascript
import {observer} from 'mobx-react';

@observer
class TimerView extends React.Component {
    render() {
        return (<button onClick={this.onReset.bind(this)}>
                Seconds passed: {this.props.appState.timer}
            </button>);
    }

    onReset () {
        this.props.appState.resetTimer();
    }
};

React.render(<TimerView appState={appState} />, document.body);
```
（关于方法`resetTimer`的实现，我们下一步再说明）

**3.修改状态**

第三步就是修改状态。不同于其他的框架，MobX不会约束你怎么做。这里的最佳实践，但是要记住的是：MobX帮助你做一个简单的事情。
接下来的代码会在每秒修改你的数据，并且UI会在需要的时候自动修改。没有明确的关系中定义的控制器的功能，改变状态或视图，应该更新。对于MobX检测所有关系只需装饰你的状态和视图使用`observable`就够了。这里有两个改变状态的例子：

```javascript
appState.resetTimer = action(function reset() {
    appState.timer = 0;
});

setInterval(action(function tick() {
    appState.timer += 1;
}), 1000);
```

`action`的使用仅仅在我们使用严格模式（默认是关闭）的MobX，但是可以帮助你更好的组织应用的结构和表示函数修改状态的意图。

## 1.2 概念与原则

**概念**

---

mobx区分下列应用程序中的概念。你看到他们在以前的主旨，但让我们挖掘多一点细节。

1、State(状态)

State是驱动你应用的数据。通常有特定区域的状态如一个todo的items和一个当前选中的元素的视图状态。记住，状态就像持有值的电子表格单元格。

2、Derivations(推导)

任何可以从状态中得到的，没有任何进一步的相互作用的我们把它叫做Derivations(推导)。Derivations 存在很多情况：
* 用户界面（UI）
* Drived data(导出的数据)，如todos。
* Backend integrations(后端集成)，就像将更改发送到服务器。 

MobX区分两种Derivations:
* Computed values(计算值)。这个数据会被一个方法通过当前观察的状态进行计算输出。
* Reactions(响应)。响应会在状态修改的发生作用。这些都是必要的反应和反应编程之间的桥梁。或者更简单的说，他们最终需要实现I/O(输入/输出).

用户一开始使用MobX就企图经常使用reactions。这里有个黄金法则是：如果你想创建一个基于当前状态的值，那么请使用`computed`。
回到试算表的类比，公式是计算值的推导.。但你作为一个用户能够在屏幕上看到一个reaction(反应)需要重画的GUI部分。

3、Actions(动作)

一个Action是改变State的地方。用户事件，后端数据推送，预定的事件等等。一个动作就像在电子表格单元格中输入新值的一个用户一样。
Actions可以明确的定义在你的MobX项目来帮助你更清晰地组织代码结构。如果你使用的是严格模式，MobX将会强制要求你只能在actions里面修改你的State。

### 原则

mobx支持单向数据流，动作状态发生变化，从而更新所有受影响的看法。

Action -> State -> Views

所有Derivations都是根据状态修改来自动更新(automatically)和原子(atomically)的。因此，是不可能观察中间值的。
默认情况下，所有Derivations同步(synchronously)更新。这意味着，例如，Actions可以安全地检查一个计算值后，直接改变状态。
Computed values的是懒更新(updated lazily)的。任何不在激活使用的Computed values不会被更新，除非发生了它需要的副作用(I/O)操作。如果视图不再使用，将自动垃圾回收。
所有Computed values都应是纯的（pure），他们不应该改版状态State。

### Illustration(例证)

下面列出了上述概念和原则：

```javascript
import {observable, autorun} from 'mobx';

var todoStore = observable({
    /* some observable state */
    todos: [],

    /* a derived value */
    get completedCount() {
        return this.todos.filter(todo => todo.completed).length;
    }
});

/* a function that observes the state */
autorun(function() {
    console.log("Completed %d of %d items",
        todoStore.completedCount,
        todoStore.todos.length
    );
});

/* ..and some actions that modify the state */
todoStore.todos[0] = {
    title: "Take a walk",
    completed: false
};
// -> synchronously prints 'Completed 0 of 1 items'

todoStore.todos[0].completed = true;
// -> synchronously prints 'Completed 1 of 1 items'
```

![MobX](https://mobxjs.github.io/mobx/getting-started-assets/overview.png)

In the [10 minute introduction to MobX and React](https://mobxjs.github.io/mobx/getting-started.html) you can dive deeper into this example and build a user interface using React around it.

# 2. Api 概述

[MobX Api Reference](http://mobxjs.github.io/mobx/refguide/api.html)

## 2.1 observable

使用：
* `observable(value)`
* `@observable classProperty = value`

Observabl的值可以是JS primitives, references, plain objects, class instances, arrays and maps。应用下列转换规则，但可以使用修饰符进行微调。看下面。

1. 如果value被包裹在修改器`asMap`里面：将会返回一个新的[Observable Map](http://mobxjs.github.io/mobx/refguide/map.html)。Observable maps是非常有用的，如果你不想作出反应，只是修改一个特定的条目，但也增加或删除条目。
2. 如果value是一个数组(array)，将会返回一个新的[Observable Array](http://mobxjs.github.io/mobx/refguide/array.html)。
3. 如果value是一个没有原型(prototype)的对象，它里面所有属性都将会变成observable。见[Observable Object](http://mobxjs.github.io/mobx/refguide/object.html)。
4. 如果value是一个有原型(prototype)的对象，一个JavaScript原语或者function，将会返回一个[Boxed Observable](http://mobxjs.github.io/mobx/refguide/boxed.html)，MobX不会让一个有prototype的对象自动observable，因为这是它的构造函数的责任。在构造函数使用`extendObservable`或者在它的类里面使用`@obervable`定义。

这些规则看起来似乎很复杂，但你会注意到，在实践中他们非常直观的工作。 一些注意事项：

* 要创建动态键控对象，请使用`asMap`修饰符！ 只有对象上的最初存在的属性将被使得可观察，虽然可以使用`extendObservable`添加新的属性。
* 要使用@observable装饰器，请确保在转换器（babel或typescript）中启用装饰器。
* 默认情况下，使数据结构可观察是感染性的; 这意味着observable可以自动应用于数据结构包含的任何值，或者将来包含在数据结构中。 此行为可以通过使用修饰符更改。

一些例子：

```javascript
const map = observable(asMap({ key: "value"}));
map.set("key", "new value");

const list = observable([1, 2, 4]);
list[2] = 3;

const person = observable({
    firstName: "Clive Staples",
    lastName: "Lewis"
});
person.firstName = "C.S.";

const temperature = observable(20);
temperature.set(25);
```
## 2.2 @observable

可以在ES7或TypeScript类属性上使用的装饰器，以使它们可见。 @observable可以在实例字段和属性getters上使用。 这提供了对对象的那些部分变得可观察的细粒度控制。

```javascript
import {observable} from "mobx";

class OrderLine {
    @observable price:number = 0;
    @observable amount:number = 1;

    constructor(price) {
        this.price = price;
    }

    @computed get total() {
        return this.price * this.amount;
    }
}
```
如果你的环境不支持装饰器或字段初始化器，` @observable key = value `是` extendObservable(this, { key : value }) `的语法糖。
Enumerability(可枚举性)：属性装饰器与@observable是可枚举的，但定义在类原型而不是类实例。 换一种说法：
```javascript
const line = new OrderLine();
console.log("price" in line); // true
console.log(line.hasOwnProperty("price")); // false, the price _property_ is defined on the class, although the value will be stored per instance.
```
装饰器`@obervable`可以和修饰符一起使用，如`asStructure`：
`@observable position = asStructure({ x: 0, y: 0})`

**在你的转换器中启用装饰器**

当使用TypeScript或Babel等待ES标准中的定义时，默认情况下不支持装饰器。
* For typescript, enable the --experimentalDecorators compiler flag or set the compiler option experimentalDecorators to true in tsconfig.json (Recommended)
* For babel5, make sure --stage 0 is passed to the Babel CLI
* For babel6, see the example configuration as suggested in this [issue](https://github.com/mobxjs/mobx/issues/105)


## 2.3 @computed

-

可以在ES6或TypeScript派生类属性上使用的装饰器，以使它们变成可观察的。
`@computed`只能用在实力属性的`get`方法上面。

如果你有一个值可以从纯粹的方式从其他observables导出，使用`@computed`。

不要混淆了`@computed`和`autorun`，它们都是响应式调用的表达式。如果你想响应产生一个可以被其他观察者使用的新值，那么使用`@computed`；如果你不想产生一个新的值，而是调用一些命令式的代码，如日志记录，网络请求等，那么请使用`autorun`。

计算的属性可以在许多情况下被MobX优化，因为它们被假定为纯的。因此，当它们的输入参数没有修改，或者如果它们没有被一些其他计算值或自动运行观察到时，它们将不被调用。

```javascript
import {observable, computed} from "mobx";

class OrderLine {
    @observable price:number = 0;
    @observable amount:number = 1;

    constructor(price) {
        this.price = price;
    }

    @computed get total() {
        return this.price * this.amount;
    }
}
```
如果你的环境不支持装饰器或字段初始化器，` @gomputed get funcName() {} `是` extendObservable(this, { funcName : func }) `的语法糖。

` @computed `可以是参数化的。` @computed({asStructure: true}) `确保derivation(推导)的结果在结构上进行比较，而不是与其预览值。这确保计算的观察者不重新评估是否返回结构上等于原始结构的新结构。这在使用点，矢量或颜色结构时非常有用。 它的行为与可见值的`asStructure`修饰符相同。

@computed属性不可枚举。 它们也不能在继承链中被覆盖。

**使用`observable`或者`extendObservable`创建computed values(计算值)**

方法`obervable(object)`或者`extendObservable(target, properties)`也可以用来引入计算属性，作为使用装饰器的替代方法。对于这个ES5 getters可以使用，所以上面的例子也可以写成：
```javascript
var orderLine = observable({
    price: 0,
    amount: 1,
    get total() {
        return this.price * this.amount
    }
});
```
注意：在MobX 2.5.1中引入了对getter的支持。 MobX将自动将作为属性值传递的无参函数转换为observable / extendObservable为计算属性，但该形式将在下一个主版本中消失。

**Setters for computed values**

也可以为计算值定义setters。 请注意，这些setter不能用于直接更改计算属性的值，但它们可以用作derivation(派生)的“inverse”。 例如：

```javascript
const box = observable({
    length: 2,
    get squared() {
        return this.length * this.length;
    },
    set squared(value) {
        this.length = Math.sqrt(value);
    }
});
```
或者这样同样可行：

```javascript
class Foo {
    @observable length: 2,
    @computed get squared() {
        return this.length * this.length;
    },
    set squared(value) { //this is automatically an action, no annotation necessary
        this.length = Math.sqrt(value);
    }
}
```
Note: setters require MobX 2.5.1 or higher

**computed(expression)**

`computed`同样可以像方法一样调用。就像`observable(primitive value)`，它将创建一个独立的observable。对返回的对象使用`.get()`以获取计算的当前值，或者使用`.observe(callback)`观察其更改。这种形式的计算不是经常使用，但在某些情况下，你需要在它周围传递一个“盒装(boxed)”的计算值可能证明是有用的。
例子：
```javascript
import {observable, computed} from "mobx";
var name = observable("John");

var upperCaseName = computed(() =>
    name.get().toUpperCase()
);

var disposer = upperCaseName.observe(name => console.log(name));

name.set("Dave");
// prints: 'DAVE'
```



















