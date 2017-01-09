---
title: 【译】MobX文档
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

## 2.4 Autorun

`mobx.autorun` 可以用在那些你想创建一个响应函数，本身永远不会有观察者。这通常是当你需要从响应式桥到命令式代码时，例如对于日志记录(logging)，持久性(persistence)或UI更新代码。使用`mobx.autorun`，所提供的函数将在其某个依赖关系发生更改时就会触发。相比之下，`computed(function)`创建的函数只重新评估它是否有观察者本身，否则它的值被认为是不相关的。作为经验法则：如果您有一个应该自动运行但不会产生新值的函数，请使用`autorun`。其他情况使用`computed`。Autorun(自动运行)涉及启动效应，而不是产生新的价值。 如果字符串作为第一个参数传递给autorun，它将被用作调试名称。

```javascript
var numbers = observable([1,2,3]);
var sum = computed(() => numbers.reduce((a, b) => a + b, 0));

var disposer = autorun(() => console.log(sum.get()));
// prints '6'
numbers.push(4);
// prints '10'

disposer();
numbers.push(5);
// won't print anything, nor is `sum` re-evaluated
```

## 2.5 @observer

`observer`方法或者说装饰器可以用来使ReactJS的组件变成响应式的组件。它将组件的render函数包装在`mobx.autorun`中，以确保在组件渲染期间使用的任何数据在更改时强制重新渲染。它通过单独的`mobx-react`包提供。

```javascript
import {observer} from "mobx-react";

var timerData = observable({
    secondsPassed: 0
});

setInterval(() => {
    timerData.secondsPassed++;
}, 1000);

@observer class Timer extends React.Component {
    render() {
        return (<span>Seconds passed: { this.props.timerData.secondsPassed } </span> )
    }
});

React.render(<Timer timerData={timerData} />, document.body);
```
提示：当`observer`需要与其他装饰器(decorators)或高阶组件(higher-order-components)组合时，请确保`observer`是最内层（首次应用的）装饰器; 否则它可能什么都不做。

注意，使用`@observer`作为装饰器是可选的，`observer(class Timer ...{})`实现完全相同。

**了解：解除引用组件中的值**

MobX可以做很多，但它`不能使原始值可观察（虽然它可以将它们包装在一个对象中看到框的可观察值）。 所以不是可观察的值，而是对象的属性。 这意味着`@observer实际上反应了你取消引用一个值的事实。 所以在我们上面的例子中，如果Timer组件初始化如下：

```javascript
React.render(<Timer timerData={timerData.secondsPassed} />, document.body)
```
在这个片段中，只将当前值secondsPassed传递给Timer，它是不可变值0（所有原语在JS中都是不可变的）。这个数字在将来不会再改变，所以`Timer`永远不会更新。这是将来会改变的属性secondsPassed，所以我们需要在组件中访问它。 或者换句话说：值需要通过引用传递而不是值。

**ES5 support**

In ES5 environments, observer components can be simple declared using observer(React.createClass({ .... See also the [syntax guide](https://mobx.js.org/best/syntax.html)

**Stateless function components（无状态函数组件）**

上述定时器小部件也可以使用通过观察者传递的无状态函数组件编写：
```javascript
import {observer} from "mobx-react";

const Timer = observer(({ timerData }) =>
    <span>Seconds passed: { timerData.secondsPassed } </span>
);
```

**Observable local component state**

就像普通类一样，你可以使用`@observable`装饰器在组件上引入`@observable`。这意味着您可以在组件中具有本地状态，而不需要使用`React`的冗余和强制的`setState`机制来管理它，但是功能强大。反应状态将由`render`接收，但不会显式调用其他React生命周期方法，如`componentShouldUpdate`或`componentWillUpdate`。如果你需要那些，只需使用正常的基于React状态的API。

上面的例子也可以写为：
```javascript
import {observer} from "mobx-react"
import {observable} from "mobx"

@observer class Timer extends React.Component {
    @observable secondsPassed = 0

    componentWillMount() {
        setInterval(() => {
            this.secondsPassed++
        }, 1000)
    }

    render() {
        return (<span>Seconds passed: { this.secondsPassed } </span> )
    }
})

React.render(<Timer />, document.body)
```
有关使用observable本地组件状态的更多优点，请参见[3 reasons why I stopped using setState](https://medium.com/@mweststrate/3-reasons-why-i-stopped-using-react-setstate-ab73fc67a42e)。

**Connect `observer` to stores**

`mobx-react`包还提供了`Provider`组件，可以用于使用React的上下文机制传递存储。 要连接到这些stores，请将stores名称数组传递给observer，这将使stores像props一样使用。使用装饰器(`@observer("store")`) class ... 或者使用方法`observer(["store"], React.createClass({...}))`支持以上使用。
例子：
```javascript
const colors = observable({
   foreground: '#000',
   background: '#fff'
});

const App = () =>
  <Provider colors={colors}>
     <app stuff... />
  </Provider>;

const Button = observer(["colors"], ({ colors, label, onClick }) =>
  <button style={{
      color: colors.foreground,
      backgroundColor: colors.background
    }}
    onClick={onClick}
  >{label}<button>
);

// later..
colors.foreground = 'blue';
// all buttons updated
```

See for more information the [mobx-react docs](https://github.com/mobxjs/mobx-react#provider-experimental).

**什么时候该使用`observer`**

简单的经验法则是：渲染可观察数据的所有组件。 如果不想将组件标记为观察器，例如为了减少通用组件包的依赖性，请确保只传递纯数据。
使用@observer，不需要为了渲染的目的将'智能'组件与'哑'组件区分开。 它仍然是一个很好的分离，在哪里处理事件，请求等。所有组件变得负责更新时，他们自己的依赖关系改变。 它的开销是可以忽略的，它确保每当你开始使用可观察的数据，组件将响应它。 有关更多详细信息，请参阅此[线程](https://www.reddit.com/r/reactjs/comments/4vnxg5/free_eggheadio_course_learn_mobx_react_in_30/d61oh0l)。

**`observer` and `PureRenderMixin`**

`observer`也防止了当组件的道具仅仅浅地改变时的重新渲染，这使得传递到组件中的数据是反应性的，这是很有意义的。 此行为类似于[React PureRender mixin](https://facebook.github.io/react/docs/pure-render-mixin.html)，但状态更改始终处理。 如果组件提供自己的shouldComponentUpdate，那么它优先。 看到解释这个[github问题](https://github.com/mobxjs/mobx/issues/101)。

**`componentWillReact` (生命周期钩子)**

React组件通常在新堆栈上呈现，因此通常很难找出导致组件重新渲染的原因。 当使用mobx-react时，你可以定义一个新的生命周期钩子，`componentWillReact`（双关意图），当一个组件被调度重新渲染时，它将被触发，因为它观察到的数据已经改变。 这使得它很容易跟踪渲染回到导致渲染的操作。

```javascript
import {observer} from "mobx-react";

@observer class TodoView extends React.Component {
    componentWillReact() {
        console.log("I will re-render, since the todo has changed!");
    }

    render() {
        return <div>this.props.todo.title</div>;
    }
}
```
* `componentWillReact` 不带参数
* `componentWillReact` 在初始渲染之前不会触发（使用`componentWillMount`代替）
* `componentWillReact` 在接收到新的props或者在`setState`方法执行之后不会触发（使用`componentWillUpdate`代替）

**优化组件**

请参阅[相关部分](https://mobx.js.org/best/react-performance.html)。

**MobX-React-DevTools**

结合使用@observer，您可以使用MobX-React-DevTools，它显示组件何时被重新渲染，您可以检查组件的数据依赖关系。 请参阅[DevTools](https://mobx.js.org/best/devtools.html)部分。


**observer components 的特性**

* Observer仅订阅在上次渲染期间主动使用的数据结构。 这意味着您不能低于订阅或超量订阅。 您甚至可以在渲染中使用仅在稍后时间可用的数据。 这是异步加载数据的理想选择。
* 您不需要声明组件将使用什么数据。 相反，依赖性在运行时确定并以非常细粒度的方式跟踪。
* 通常反应性组件没有或很少有状态，因为在与其他组件共享的对象中封装（查看）状态通常更方便。 但你仍然可以自由使用状态。
* `@observer`以与`PureRenderMixin`相同的方式实现`shouldComponentUpdate`，这样children不会有没必要的重新渲染。
* Reactive components sideways load data; parent components won't re-render unnecessarily even when child components will.
* `@observer` 不依赖 `React` 的上下文系统.

## 2.6 action

使用
* `action(fn)`
* `action(name, fn)`
* `@action classMethod`
* `@action(name) classMethod`
* `@action boundClassMethod = (args) => { body }`
* `@action(name) boundClassMethod = (args) => { body }`

任何应用都有Actions。Actions是修改状态的地方。使用MobX，您可以通过标记它们在您的代码中显式地显示您的actions。Actions帮助你更好的组织代码。建议将它们用于修改observables或具有副作用的任何函数。action还提供了与devtools组合使用的有用的调试信息。Using the `@action` decorator with [ES 5.1 setters](http://www.ecma-international.org/ecma-262/5.1/#sec-11.1.5) (i.e. @action set propertyName) is not supported, however setters of [computed properties are automatically actions](https://github.com/mobxjs/mobx/blob/gh-pages/docs/refguide/computed-decorator.md#setters-for-computed-values).

注意：当启用严格模式(strict)时，使用操作是必需的，请参阅[useStrict](https://github.com/mobxjs/mobx/blob/gh-pages/docs/refguide/api.md#usestrict)。

有关`action`的详细介绍，请参阅[MobX 2.2 release notes](https://medium.com/p/45cdc73c7c8d/)。

Two example actions from the contact-list project:
```javascript
 @action    createRandomContact() {
        this.pendingRequestCount++;
        superagent
            .get('https://randomuser.me/api/')
            .set('Accept', 'application/json')
            .end(action("createRandomContact-callback", (error, results) => {
                if (error)
                    console.error(error);
                else {
                    const data = JSON.parse(results.text).results[0];
                    const contact = new Contact(this, data.dob, data.name, data.login.username, data.picture)
                    contact.addTag('random-user');
                    this.contacts.push(contact);
                    this.pendingRequestCount--;
                }
            }));
    }
```

**async actions and runInAction.**

action只影响当前运行的函数，而不影响当前函数调度（但未调用）的函数！ 这意味着如果你有一个`setTimeout`，promise.`then`或`async`构造，并且在该回调中一些更多的状态被改变，那些回调也应该包装在action中！ 这在上面用“createRandomContact-callback”动作来演示。
如果你使用`async / await`，这是一个有点棘手，因为你不能只是包装异步函数体在行动。 在这种情况下，`runInAction`可以派上用场，在你打算更新状态的地方包装。 （但不要在这些块中等待调用）。
例子：
```javascript
@action /*optional*/ updateDocument = async () => {
    const data = await fetchDataFromUrl();
    /* required in strict mode to be allowed to update state: */
    runInAction("update state after fetching data", () => {
        this.data.replace(data);
        this.isSaving = true;
    })
}
```
The usage of `runInAction` is: `runInAction(name?, fn, scope?)`.

If you use babel, this plugin could help you to handle your async actions: [mobx-deep-action](https://github.com/mobxjs/babel-plugin-mobx-deep-action).

