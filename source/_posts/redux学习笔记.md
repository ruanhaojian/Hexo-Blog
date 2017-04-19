---
title: react-redux学习笔记
date: 2017-04-19 10:00:56
tags: react
---

这里记录react-redux的一些关键点，并不是做为redux的介绍，自己备忘用。

<!--more-->

参考资源
* [redux](https://github.com/reactjs/redux)
* [react-router-redux](https://github.com/reactjs/react-router-redux)
* [ducks-modular-redux](https://github.com/erikras/ducks-modular-redux)
* [ga-react-tutorial](https://github.com/goopscoop/ga-react-tutorial/tree/6-reduxActionsAndReducers)
* [react-redux-universal-hot-example](https://github.com/erikras/react-redux-universal-hot-example)
* [redux-saga](https://github.com/redux-saga/redux-saga)
* [Redux有哪些最佳实践?](https://www.zhihu.com/question/47995437?sort=created)

# [redux](http://redux.js.org/)

Redux是JavaScript应用程序的可预测状态容器。[中文文档](http://cn.redux.js.org/)
You can use Redux together with React, or with any other view library.
[生态系统](http://redux.js.org/docs/introduction/Ecosystem.html)：很好的redux资源


### react-redux Sample Examples

实现容器组件

```javascript
// mapStateToProps
const getVisibleTodos = (todos, filter) => {
  switch (filter) {
    case 'SHOW_ALL':
      return todos
    case 'SHOW_COMPLETED':
      return todos.filter(t => t.completed)
    case 'SHOW_ACTIVE':
      return todos.filter(t => !t.completed)
  }
}

const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state.todos, state.visibilityFilter)
  }
}
// mapDispatchToProps
const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}

// Finally, we create the VisibleTodoList by calling connect() and passing these two functions:
import { connect } from 'react-redux'

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList

```

bindActionCreators 包装action

```javascript
// actionCreators
export function inputChange(value){
  return {
    type: INPUT_CHANGED,
    value
  }
}

// bind
import * as todoActionCreators from './todoActionCreators'
import * as counterActionCreators from './counterActionCreators'
import { bindActionCreators } from 'redux'

function mapStateToProps(state) {
  return { todos: state.todos }
}

function mapDispatchToProps(dispatch) {
  return {
    todoActions: bindActionCreators(todoActionCreators, dispatch),
    counterActions: bindActionCreators(counterActionCreators, dispatch)
  }
}

// or 
function mapDispatchToProps(dispatch) {
  return {
    actions: bindActionCreators(Object.assign({}, todoActionCreators, counterActionCreators), dispatch)
  }
}

// or 
function mapDispatchToProps(dispatch) {
  return bindActionCreators(Object.assign({}, todoActionCreators, counterActionCreators), dispatch)
}

```

mergeProps 使用

```javascript
// 根据组件的 props 注入特定用户的 todos 并把 props.userId 传入到 action 中

import * as actionCreators from './actionCreators'

function mapStateToProps(state) {
  return { todos: state.todos }
}

function mergeProps(stateProps, dispatchProps, ownProps) {
  return Object.assign({}, ownProps, {
    todos: stateProps.todos[ownProps.userId],
    addTodo: (text) => dispatchProps.addTodo(ownProps.userId, text)
  })
}

export default connect(mapStateToProps, actionCreators, mergeProps)(TodoApp)
```

Store

```javascript
import React from 'react'
import { render } from 'react-dom'
import { Provider } from 'react-redux'
import { createStore } from 'redux'
import todoApp from './reducers'
import App from './components/App'

let store = createStore(todoApp)

render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
)
```

### 异步Action

* 你可以使用 [redux-promise](https://github.com/acdlite/redux-promise) 或者 [redux-promise-middleware](https://github.com/pburtchaell/redux-promise-middleware) 来 dispatch Promise 来替代函数。
* 你可以使用 [redux-observable](https://github.com/redux-observable/redux-observable) 来 dispatch Observable。
* 你可以使用 [redux-saga](https://github.com/yelouafi/redux-saga/) 中间件来创建更加复杂的异步 action。
* 你可以使用 [redux-pack](https://github.com/lelandrichardson/redux-pack) 中间件 dispatch 基于 Promise 的异步 Action。
* 你甚至可以写一个自定义的 middleware 来描述 API 请求，就像这个 [真实场景的案例](http://cn.redux.js.org/docs/introduction/Examples.html#real-world) 中的做法一样。


### Middleware

在生态系统中，有几个比较常用的中间件。

#### react-thunk

处理异步Action

Redux Thunk middleware allows you to write action creators that return a function instead of an action. The thunk can be used to delay the dispatch of an action, or to dispatch only if a certain condition is met. The inner function receives the store methods dispatch and getState as parameters.

An action creator that returns a function to perform asynchronous dispatch:

```javascript
const INCREMENT_COUNTER = 'INCREMENT_COUNTER';

function increment() {
  return {
    type: INCREMENT_COUNTER
  };
}

function incrementAsync() {
  return dispatch => {
    setTimeout(() => {
      // Yay! Can invoke sync or async actions with `dispatch`
      dispatch(increment());
    }, 1000);
  };
}
```
An action creator that returns a function to perform conditional dispatch:

```javascript
function incrementIfOdd() {
  return (dispatch, getState) => {
    const { counter } = getState();

    if (counter % 2 === 0) {
      return;
    }

    dispatch(increment());
  };
}
```

2.1.0版本后，还添加注入参数

```javascript
const store = createStore(
  reducer,
  applyMiddleware(thunk.withExtraArgument(api))
)

// later
function fetchUser(id) {
  return (dispatch, getState, api) => {
    // you can use api here
  }
}
```

#### redux-promise

```javascript
import { isFSA } from 'flux-standard-action';

function isPromise(val) {
  return val && typeof val.then === 'function';
}

export default function promiseMiddleware({ dispatch }) {
  return next => action => {
    if (!isFSA(action)) {
      return isPromise(action)
        ? action.then(dispatch)
        : next(action);
    }

    return isPromise(action.payload)
      ? action.payload.then(
          result => dispatch({ ...action, payload: result }),
          error => {
            dispatch({ ...action, payload: error, error: true });
            return Promise.reject(error);
          }
        )
      : next(action);
  };
}
```

通过中间件处理reducer中的promise payload.

```javascript
// implicit promise
const foo = () => ({
  type: 'FOO',
  payload: new Promise()
});

// explicit promise
const foo = () => ({
  type: 'FOO',
  payload: {
    promise: new Promise()
  }
});
```

#### ducks-modular-redux

ducks modular主要是吧reducer封装起来，在component中调用无需写过多type等参数，只需调用方法。

例子：
```javascript
function mapDispatchToProps(dispatch) {
  return {
    inputChange: (value) => dispatch(inputChange(value)),
    inputSubmit: () => dispatch(inputSubmit()),
    deleteListItem: (i) => dispatch(deleteListItem(i)),
    listItemClick: (i) => dispatch(listItemClick(i))
  }; // here we're mapping actions to props
}

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(ToDoApp);
```
```javascript
import React from 'react';
import List from './List';
import Input from './Input';

class ToDoApp extends React.Component {

  onInputChange = (event) => {
    this.props.inputChange(event.target.value);
  };

  onInputSubmit = (event) => {
    event.preventDefault();
    this.props.inputSubmit();
  };

  onListItemClick = (i) => {
    this.props.listItemClick(i)
  };

  deleteListItem = (i) => {
    this.props.deleteListItem(i)
  };

  render(){
    console.log(this.props)
    return (
      <div className="row">
        <div className="col-md-8 col-md-offset-2">
          <div className="panel panel-default">
            <div className="panel-body">
              <h1>My To Do App</h1>
              <hr/>
              <List
                onClick={this.onListItemClick}
                listItems={this.props.toDoApp.list}
                deleteListItem={this.deleteListItem}
              />
              <Input
                value={this.props.toDoApp.newToDo}
                onChange={this.onInputChange}
                onSubmit={this.onInputSubmit}
              />
            </div>
          </div>
        </div>
      </div>
    );
  }
}

export default ToDoApp;
```

createActionDispathchers example

```javascript
/**
 * Creates a function which creates same-named action dispatchers from an object
 * whose function properties are action creators. Any non-functions in the actionCreators
 * object are ignored.
 */
var createActionDispatchers = actionCreators => dispatch =>
  Object.keys(actionCreators).reduce((actionDispatchers, name) => {
    var actionCreator = actionCreators[name];
    if (typeof actionCreator == 'function') {
      actionDispatchers[name] = (...args) => dispatch(actionCreator(...args));
    }
    return actionDispatchers;
  }, {})

var actionCreators = require('./ducks/widgets');
var mapStateToProps = state => state.widgets;
var mapDispatchToProps = createActionDispatchers(actionCreators);

var MyComponent = React.createClass({ /* ... */ });

module.exports = connect(mapStateToProps , mapDispatchToProps)(MyComponent);
```






















