---
title: react学习笔记
date: 2017-04-14 14:00:38
tags:
---

记录一些好的实践、性能优化。

<!--more-->

React特点：
1. “轻”；react是很轻量的框架 
2. 速度快，效率高；真正Dom其实是很巨大的，直接操作Dom很耗性能。React引入virtual dom概念，它不直接操作Dom而是将Dom结构存储在内存中，然后通过render中返回的内容做一个Dom Diff的比较，再将变化反映到Dom做局部更新。 
3. 组件化思想；利用React可以轻松构建所需要的组件，并且各组件之间可轻易的组合，使得代码重用及各模块之间的测试变得更加简单。 
4. 单向式响应数据流；React实现了单向式相应数据流，使得数据交互变得非常简易清晰，减少了许多重复代码，比传统的数据绑定简单。

# React知识点

**ReactJS Lifecycle Methods(Context)**

```javascritp

constructor(props, context)

componentWillReceiveProps(nextProps, nextContext)

shouldComponentUpdate(nextProps, nextState, nextContext)

componentWillUpdate(nextProps, nextState, nextContext)

componentDidUpdate(prevProps, prevState, PrevContext)

```

**Mounting**

```javascript

constructor()

componentWillMount()

render()

componentDidMount()

```

**Updating**

```javascript

componentWillReceiveProps(nextProps, nextContext)

shouldComponentUpdate(nextProps, nextState, nextContext)

componentWillUpdate(nextProps, nextState, nextContext)

render()

componentDidUpdate(prevProps, prevState, PrevContext)

```

**UnMount**

```javascript

componentWillUnmount()

```

**在生命周期中的哪一步你应该发起 AJAX 请求**

我们应当将AJAX 请求放到 componentDidMount 函数中执行，主要原因有下：

* React 下一代调和算法 Fiber 会通过开始或停止渲染的方式优化应用性能，其会影响到 componentWillMount 的触发次数。对于 componentWillMount 这个生命周期函数的调用次数会变得不确定，React 可能会多次频繁调用 componentWillMount。如果我们将 AJAX 请求放到 componentWillMount 函数中，那么显而易见其会被触发多次，自然也就不是好的选择。

* 如果我们将 AJAX 请求放置在生命周期的其他函数中，我们并不能保证请求仅在组件挂载完毕后才会要求响应。如果我们的数据请求在组件挂载之前就完成，并且调用了setState函数将数据添加到组件状态中，对于未挂载的组件则会报错。而在 componentDidMount 函数中进行 AJAX 请求则能有效避免这个问题。

**优化关键：shouldComponentUpdate**

shouldComponentUpdate 允许我们手动地判断是否要进行组件更新，根据组件的应用场景设置函数的合理返回值能够帮我们避免不必要的更新。

**调用 setState 之后发生了什么？**

在代码中调用setState函数之后，React 会将传入的参数对象与组件当前的状态合并，然后触发所谓的调和过程（Reconciliation）。经过调和过程，React 会以相对高效的方式根据新的状态构建 React 元素树并且着手重新渲染整个UI界面。在 React 得到元素树之后，React 会自动计算出新的树与老树的节点差异，然后根据差异对界面进行最小化重渲染。在差异计算算法中，React 能够相对精确地知道哪些位置发生了改变以及应该如何改变，这就保证了按需更新，而不是全部重新渲染。

**React 中 Element 与 Component 的区别是？**

简单而言，React Element 是描述屏幕上所见内容的数据结构，是对于 UI 的对象表述。典型的 React Element 就是利用 JSX 构建的声明式代码片然后被转化为createElement的调用组合。而 React Component 则是可以接收参数输入并且返回某个 React Element 的函数或者类。更多介绍可以参考[React Elements vs React Components](http://link.zhihu.com/?target=https%3A//tylermcginnis.com/react-elements-vs-react-components/)。

**在什么情况下你会优先选择使用 Class Component 而不是 Functional Component？**

在组件需要包含内部状态或者使用到生命周期函数的时候使用 Class Component ，否则使用函数式组件。
```javascript
export class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = {count: props.initialCount};
    this.tick = this.tick.bind(this);
  }
  tick() {
    this.setState({count: this.state.count + 1});
  }
  render() {
    return (
      <div onClick={this.tick}>
        Clicks: {this.state.count}
      </div>
    );
  }
}
Counter.propTypes = { initialCount: React.PropTypes.number };
Counter.defaultProps = { initialCount: 0 };
```

```javascript
const HelloMessage = (props) => <div>Hello, {props.name}</div>;
HelloMessage.propTypes = {
  name: React.PropTypes.string
}
HelloMessage.defaultProps = {
  name: 'John Doe'
}
ReactDOM.render(<HelloMessage name="Mădălina"/>, mountNode);
```

**React 中 refs 的作用**

Refs 是 React 提供给我们的安全访问 DOM 元素或者某个组件实例的句柄。我们可以为元素添加ref属性然后在回调函数中接受该元素在 DOM 树中的句柄，该值会作为回调函数的第一个参数返回：

```javascript
 render: function() {
    return (
      <TextInput
        ref={function(input) {
          if (input != null) {
            input.focus();
          }
        }} />
    );
  },

  // or using an es6 arrow function:

  render: function() {
    return <TextInput ref={(c) => this._input = c} />;
  },
  componentDidMount: function() {
    this._input.focus();
  },
```

**React 中 keys 的作用**

Keys 是 React 用于追踪哪些列表中元素被修改、被添加或者被移除的辅助标识。
```javascript
render () {
  return (
    <ul>
      {this.state.todoItems.map(({task, uid}) => {
        return <li key={uid}>{task}</li>
      })}
    </ul>
  )
}
```
在开发过程中，我们需要保证某个元素的 key 在其同级元素中具有唯一性。在 React Diff 算法中 React 会借助元素的 Key 值来判断该元素是新近创建的还是被移动而来的元素，从而减少不必要的元素重渲染。此外，React 还需要借助 Key 值来判断元素与本地状态的关联关系，因此我们绝不可忽视转换函数中 Key 的重要性。

**React 中this.props.children的用法**

回调渲染模式（Render Callback Pattern）。这种模式中，组件会接收某个函数作为其子组件，然后在渲染函数中以props.children进行调用：

```javascript
<Twitter username='tylermcginnis33'>
  {(user) => user === null
    ? <Loading />
    : <Badge info={user} />}
</Twitter>
```
```javascript
import React, { Component, PropTypes } from 'react'
import fetchUser from 'twitter'
class Twitter extends Component {
  state = {
    user: null,
  }
  static propTypes = {
    username: PropTypes.string.isRequired,
  }
  componentDidMount () {
    fetchUser(this.props.username)
      .then((user) => this.setState({user}))
  }
  render () {
    return this.props.children(this.state.user)
  }
}
```
这种模式的优势在于将父组件与子组件解耦和，父组件可以直接访问子组件的内部状态而不需要再通过Props传递，这样父组件能够更为方便地控制子组件展示的UI界面。譬如产品经理让我们将原本展示的Badge替换为Profile，我们可以轻易地修改下回调函数即可：
```javascript
<Twitter username='tylermcginnis33'>
  {(user) => user === null
    ? <Loading />
    : <Profile info={user} />}
</Twitter>
```

**Controlled Component 与 Uncontrolled Component 之间的区别**

React 的核心组成之一就是能够维持内部状态的自治组件，不过当我们引入原生的HTML表单元素时（input,select,textarea 等），我们是否应该将所有的数据托管到 React 组件中还是将其仍然保留在 DOM 元素中呢？这个问题的答案就是受控组件与非受控组件的定义分割。受控组件（Controlled Component）代指那些交由 React 控制并且所有的表单数据统一存放的组件。譬如下面这段代码中username变量值并没有存放到DOM元素中，而是存放在组件状态数据中。任何时候我们需要改变username变量值时，我们应当调用setState函数进行修改。

```javascript
class ControlledForm extends Component {
  state = {
    username: ''
  }
  updateUsername = (e) => {
    this.setState({
      username: e.target.value,
    })
  }
  handleSubmit = () => {}
  render () {
    return (
      <form onSubmit={this.handleSubmit}>
        <input
          type='text'
          value={this.state.username}
          onChange={this.updateUsername} />
        <button type='submit'>Submit</button>
      </form>
    )
  }
}
```

而非受控组件（Uncontrolled Component）则是由DOM存放表单数据，并非存放在 React 组件中。我们可以使用 refs 来操控DOM元素：

```javascript
class UnControlledForm extends Component {
  handleSubmit = () => {
    console.log("Input Value: ", this.input.value)
  }
  render () {
    return (
      <form onSubmit={this.handleSubmit}>
        <input
          type='text'
          ref={(input) => this.input = input} />
        <button type='submit'>Submit</button>
      </form>
    )
  }
}
```
竟然非受控组件看上去更好实现，我们可以直接从 DOM 中抓取数据，而不需要添加额外的代码。不过实际开发中我们并不提倡使用非受控组件，因为实际情况下我们需要更多的考虑表单验证、选择性的开启或者关闭按钮点击、强制输入格式等功能支持，而此时我们将数据托管到 React 中有助于我们更好地以声明式的方式完成这些功能。引入 React 或者其他 MVVM 框架最初的原因就是为了将我们从繁重的直接操作 DOM 中解放出来。

**setState使用**

```javascript
// 基本使用
this.setState({ username: 'tylermcginnis33' })
// 完成回调
this.setState(
  { username: 'tylermcginnis33' },
  () => console.log('setState has finished and the component has re-rendered.')
)
// 状态针
this.setState(function(prevState, props){
	return {showForm: !prevState.showForm}
});
```

**[Animation](http://reactjs.cn/react/docs/animation.html)**


```javascript
  render: function() {
    var items = this.state.items.map(function(item, i) {
      return (
        <div key={item} onClick={this.handleRemove.bind(this, i)}>
          {item}
        </div>
      );
    }.bind(this));
    return (
      <div>
        <button onClick={this.handleAdd}>Add Item</button>
        <ReactCSSTransitionGroup transitionName="example" transitionEnterTimeout={500} transitionLeaveTimeout={300}>
          {items}
        </ReactCSSTransitionGroup>
      </div>
    );
  }
```
```css
.example-enter {
  opacity: 0.01;
}

.example-enter.example-enter-active {
  opacity: 1;
  transition: opacity 500ms ease-in;
}

.example-leave {
  opacity: 1;
}

.example-leave.example-leave-active {
  opacity: 0.01;
  transition: opacity 300ms ease-in;
}
```

**概述下 React 中的事件处理逻辑**

为了解决跨浏览器兼容性问题，React 会将浏览器原生事件（Browser Native Event）封装为合成事件（SyntheticEvent）传入设置的事件处理器中。这里的合成事件提供了与原生事件相同的接口，不过它们屏蔽了底层浏览器的细节差异，保证了行为的一致性。另外有意思的是，React 并没有直接将事件附着到子元素上，而是以单一事件监听器的方式将所有的事件发送到顶层进行处理。这样 React 在更新 DOM 的时候就不需要考虑如何去处理附着在 DOM 上的事件监听器，最终达到优化性能的目的。

[Event System](http://reactjs.cn/react/docs/events.html)


**createElement 与 cloneElement 的区别**

createElement 函数是 JSX 编译之后使用的创建 React Element 的函数，而 cloneElement 则是用于复制某个元素并传入新的 Props。



# React性能优化 & 最佳实践

## Immutable

[Immutable.js](http://facebook.github.io/immutable-js/)

[Immutable 详解及 React 中实践](https://github.com/camsong/blog/issues/3)

结合 shouldComponentUpdate()

## Redux

[Redux有哪些最佳实践?](https://www.zhihu.com/question/47995437?sort=created)

1. action creators和reducer请用pure函数，不要副作用。
2. immutable.js配合效果很好。
3. 项目大了请用ducks module https://github.com/erikras/ducks-modular-redux
4. reducer里只存储必要的state, 需要推到的都放到selector里(connect函数的第一个参数)
5. reducer和selector请放到一个文件，请记住reducer之间可以compose.
6. 请善用higher order component. 如果某一项功能是多个组建通用，higher order component往往要比套一层container更灵活。
7. 请慎重选择组建树的哪一层使用connected component(连接到store)，通常是比较高层的组建用来和store沟通，最低层组建使用这防止太长的prop chain.
8. 项目大了之后请用**[redux-saga](http://leonshi.com/redux-saga-in-chinese/index.html)**或者**[redux-observables](https://github.com/redux-observable/redux-observable)**
9. 请慎用自定义的redux-middleware,错误的配置可能会影响到其他middleware.


# React Diff

[深入浅出React（四）：虚拟DOM Diff算法解析](http://www.infoq.com/cn/articles/react-dom-diff/)

* 不同节点直接删除
* 逐层节点比较，先消除，在重新创建

理解diff让我们更好的理解生命周期。


# 资源

[react-redux-universal-hot-example](https://github.com/erikras/react-redux-universal-hot-example) 一个使用全面技术的例子



























