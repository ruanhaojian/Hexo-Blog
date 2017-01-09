---
title: 【转】vuex2.0源码分析
date: 2016-12-23 10:17:20
tags: 
	- vue
	- vuex
categories:
  - vue
---

> 原文：[https://github.com/DDFE/vue-blog/issues/8](https://github.com/DDFE/vue-blog/issues/8)

当我们用 Vue.js 开发一个中到大型的单页应用时，经常会遇到如下问题：

* 如何让多个 Vue 组件共享状态
* Vue 组件间如何通讯

通常，在项目不是很复杂的时候，我们会利用全局事件总线 （global event bus）解决，但是随着复杂度的提升，这些代码将变的难以维护。因此，我们需要一种更加好用的解决方案，于是，Vuex 诞生了。

本文并不是 Vuex 的科普文章，对于还不了解 Vuex 的同学，建议先移步 [Vuex 官方文档](https://vuex.vuejs.org/en/index.html)；看英文文档吃力的同学，可以看 [Vuex 的中文文档](https://vuefe.cn/)。

<!--more-->

# 目录结构
-------------
Vuex 的源码托管在 github，我们首先通过 git 把代码 clone 到本地，选一款适合自己的 IDE 打开源码，展开 src 目录，如下图所示：
```
.
└── src
    ├── plugins
    │   ├── devtool.js
    │	└── logger.js
    ├── helpers.js
    ├── index.js
    ├── mixin.js
    └── util.js

```

src 目录下的文件并不多，包含几个 js 文件和 plugins 目录， plugins 目录里面包含 2 个 Vuex 的内置插件，整个源码加起来不过 500-600 行，可谓非常轻巧的一个库。

麻雀虽小，五脏俱全，我们先直观的感受一下源码的结构，接下来看一下其中的实现细节。

# 源码分析
-------------

本文的源码分析过程不会是自上而下的给代码加注释，我更倾向于是从 Vuex 提供的 API 和我们的使用方法等维度去分析。Vuex 的源码是基于 es6 的语法编写的，对于不了解 es6 的同学，建议还是先学习一下 es6。

## 从入口开始

看源码一般是从入口开始，Vuex 源码的入口是 src/index.js，先来打开这个文件。

我们首先看这个库的 export ，在 index.js 代码最后。


```javascript
export default {
  Store,
  install,
  mapState,
  mapMutations,
  mapGetters,
  mapActions
}
```
这里可以一目了然地看到 Vuex 对外暴露的 API。其中, Store 是 Vuex 提供的状态存储类，通常我们使用 Vuex 就是通过创建 Store 的实例，稍后我们会详细介绍。接着是 install 方法，这个方法通常是我们编写第三方 Vue 插件的“套路”，先来看一下“套路”代码：

```javascript
function install (_Vue) {
  if (Vue) {
    console.error(
      '[vuex] already installed. Vue.use(Vuex) should be called only once.'
    )
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}

// auto install in dist mode
if (typeof window !== 'undefined' && window.Vue) {
  install(window.Vue)
}
```

我们实现了一个 install 方法，这个方法当我们全局引用 Vue ，也就是 window 上有 Vue 对象的时候，会手动调用 install 方法，并传入 Vue 的引用；当 Vue 通过 npm 安装到项目中的时候，我们在代码中引入第三方 Vue 插件通常会编写如下代码：

```javascript
import Vue from 'vue'
import Vuex from 'vuex'
...
Vue.use(Vuex)
```

当我们执行 Vue.use(Vuex) 这句代码的时候，实际上就是调用了 install 的方法并传入 Vue 的引用。install 方法顾名思义，现在让我们来看看它的实现。它接受了一个参数 _Vue，函数体首先判断 Vue ，这个变量的定义在 index.js 文件的开头部分：

```javascript
let Vue // bind on install
```

对 Vue 的判断主要是保证 install 方法只执行一次，这里把 install 方法的参数 _Vue 对象赋值给 Vue 变量，这样我们就可以在 index.js 文件的其它地方使用 Vue 这个变量了。install 方法的最后调用了 applyMixin 方法，我们顺便来看一下这个方法的实现，在 src/mixin.js 文件里定义：

```javascript
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    const usesInit = Vue.config._lifecycleHooks.indexOf('init') > -1
    Vue.mixin(usesInit ? { init: vuexInit } : { beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```

这段代码的作用就是在 Vue 的生命周期中的初始化（1.0 版本是 init，2.0 版本是 beforeCreated）钩子前插入一段 Vuex 初始化代码。这里做的事情很简单——给 Vue 的实例注入一个 $store 的属性，这也就是为什么我们在 Vue 的组件中可以通过 this.$store.xxx 访问到 Vuex 的各种数据和状态。

## 认识 Store 构造函数

我们在使用 Vuex 的时候，通常会实例化 Store 类，然后传入一个对象，包括我们定义好的 actions、getters、mutations、state等，甚至当我们有多个子模块的时候，我们可以添加一个 modules 对象。那么实例化的时候，到底做了哪些事情呢？带着这个疑问，让我们回到 index.js 文件，重点看一下 Store 类的定义。Store 类定义的代码略长，我不会一下就贴上所有代码，我们来拆解分析它，首先看一下构造函数的实现：

```javascript
class Store {
  constructor (options = {}) {
    assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
    assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)

    const {
      state = {},
      plugins = [],
      strict = false
    } = options

    // store internal state
    this._options = options
    this._committing = false
    this._actions = Object.create(null)
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._runtimeModules = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()

    // bind commit and dispatch to self
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // strict mode
    this.strict = strict

    // init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    installModule(this, state, [], options)

    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreVM(this, state)

    // apply plugins
    plugins.concat(devtoolPlugin).forEach(plugin => plugin(this))
  }
  ...
}  
```

构造函数的一开始就用了“断言函数”，来判断是否满足一些条件。

```javascript
assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
```
这行代码的目的是确保 Vue 的存在，也就是在我们实例化 Store 之前，必须要保证之前的 install 方法已经执行了。

```javascript
assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
```

这行代码的目的是为了确保 Promsie 可以使用的，因为 Vuex 的源码是依赖 Promise 的。Promise 是 es6 提供新的 API，由于现在的浏览器并不是都支持 es6 语法的，所以通常我们会用 babel 编译我们的代码，如果想使用 Promise 这个 特性，我们需要在 package.json 中添加对 babel-polyfill 的依赖并在代码的入口加上 `import 'babel-polyfill'` 这段代码。

再来看看 assert 这个函数，它并不是浏览器原生支持的，它的实现在 src/util.js 里，代码如下：

```javascript
export function assert (condition, msg) {
  if (!condition) throw new Error(`[vuex] ${msg}`)
}
```
非常简单，对 condition 判断，如果不不为真，则抛出异常。这个函数虽然简单，但这种编程方式值得我们学习。

再来看构造函数接下来的代码：

```javascript
const {
  state = {},
  plugins = [],
  strict = false
} = options
```
这里就是利用 es6 的结构赋值拿到 options 里的 state，plugins 和 strict。state 表示 rootState，plugins 表示应用的插件、strict 表示是否开启严格模式。

接着往下看：

```javascript
// store internal state
this._options = options
this._committing = false
this._actions = Object.create(null)
this._mutations = Object.create(null)
this._wrappedGetters = Object.create(null)
this._runtimeModules = Object.create(null)
this._subscribers = []
this._watcherVM = new Vue()
```

这里主要是创建一些内部的属性：
this._options 存储参数 options。
this._committing 标志一个提交状态，作用是保证对 Vuex 中 state 的修改只能在 mutation 的回调函数中，而不能在外部随意修改 state。
this._actions 用来存储用户定义的所有的 actions。
this._mutations 用来存储用户定义所有的 mutatins。
this._wrappedGetters 用来存储用户定义的所有 getters 。
this._runtimeModules 用来存储所有的运行时的 modules。
this._subscribers 用来存储所有对 mutation 变化的订阅者。
this._watcherVM 是一个 Vue 对象的实例，主要是利用 Vue 实例方法 $watch 来观测变化的。

继续往下看：

```javascript
// bind commit and dispatch to self
const store = this
const { dispatch, commit } = this
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}

// strict mode
this.strict = strict
```

这里的代码也不难理解，把 Store 类的 dispatch 和 commit 的方法的 this 指针指向当前 store 的实例上，dispatch 和 commit 的实现我们稍后会分析。this.strict 表示是否开启严格模式，在严格模式下会观测所有的 state 的变化，建议在开发环境时开启严格模式，线上环境要关闭严格模式，否则会有一定的性能开销。

## Vuex 的初始化核心

### installModule

我们接着往下看：

```javascript
// init root module.
// this also recursively registers all sub-modules
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], options)

// initialize the store vm, which is responsible for the reactivity
// (also registers _wrappedGetters as computed properties)
resetStoreVM(this, state)

// apply plugins
plugins.concat(devtoolPlugin).forEach(plugin => plugin(this))
```

这段代码是 Vuex 的初始化的核心，其中，installModule 方法是把我们通过 options 传入的各种属性模块注册和安装；resetStoreVM 方法是初始化 store._vm，观测 state 和 getters 的变化；最后是应用传入的插件。

下面，我们先来看一下 installModule 的实现：

```javascript
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  const {
    state,
    actions,
    mutations,
    getters,
    modules
  } = module

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, state || {})
    })
  }

  if (mutations) {
    Object.keys(mutations).forEach(key => {
      registerMutation(store, key, mutations[key], path)
    })
  }

  if (actions) {
    Object.keys(actions).forEach(key => {
      registerAction(store, key, actions[key], path)
    })
  }

  if (getters) {
    wrapGetters(store, getters, path)
  }

  if (modules) {
    Object.keys(modules).forEach(key => {
      installModule(store, rootState, path.concat(key), modules[key], hot)
    })
  }
}
```

installModule 函数可接收5个参数，store、rootState、path、module、hot，store 表示当前 Store 实例，rootState 表示根 state，path 表示当前嵌套模块的路径数组，module 表示当前安装的模块，hot 当动态改变 modules 或者热更新的时候为 true。

先来看这部分代码：

```javascript
const isRoot = !path.length
 const {
   state,
   actions,
   mutations,
   getters,
   modules
 } = module
```

代码首先通过 path 数组的长度判断是否为根。我们在构造函数调用的时候是 installModule(this, state, [], options)，所以这里 isRoot 为 true。module 为传入的 options，我们拿到了 module 下的 state、actions、mutations、getters 以及嵌套的 modules。

接着看下面的代码：

```javascript
// set state
if (!isRoot && !hot) {
  const parentState = getNestedState(rootState, path.slice(0, -1))
  const moduleName = path[path.length - 1]
  store._withCommit(() => {
    Vue.set(parentState, moduleName, state || {})
  })
}
```

这里判断当不为根且非热更新的情况，然后设置级联状态，这里乍一看不好理解，我们先放一放，稍后来回顾。

再往下看代码：

```javascript
if (mutations) {
  Object.keys(mutations).forEach(key => {
    registerMutation(store, key, mutations[key], path)
  })
}

if (actions) {
  Object.keys(actions).forEach(key => {
    registerAction(store, key, actions[key], path)
  })
}

if (getters) {
  wrapGetters(store, getters, path)
}
```

这里分别是对 mutations、actions、getters 进行注册，如果我们实例化 Store 的时候通过 options 传入这些对象，那么会分别进行注册，我稍后再去介绍注册的具体实现。那么到这，如果 Vuex 没有 module ，这个 installModule 方法可以说已经做完了。但是 Vuex 巧妙了设计了 module 这个概念，因为 Vuex 本身是单一状态树，应用的所有状态都包含在一个大对象内，随着我们应用规模的不断增长，这个 Store 变得非常臃肿。为了解决这个问题，Vuex 允许我们把 store 分 module（模块）。每一个模块包含各自的 state、mutations、actions 和 getters，甚至是嵌套模块。所以，接下来还有一行代码：

```javascript
if (modules) {
  Object.keys(modules).forEach(key => {
    installModule(store, rootState, path.concat(key), modules[key], hot)
  })
}
```

这里通过遍历 modules，递归调用 installModule 去安装子模块。这里传入了 store、rootState、path.concat(key)、和 modules[key]，和刚才不同的是，path 不为空，module 对应为子模块，那么我们回到刚才那段代码：

```javascript
// set state
if (!isRoot && !hot) {
  const parentState = getNestedState(rootState, path.slice(0, -1))
  const moduleName = path[path.length - 1]
  store._withCommit(() => {
    Vue.set(parentState, moduleName, state || {})
  })
}
```

当递归初始化子模块的时候，isRoot 为 false，注意这里有个方法getNestedState(rootState, path)，来看一下 getNestedState 函数的定义：

```javascript
function getNestedState (state, path) {
  return path.length
    ? path.reduce((state, key) => state[key], state)
    : state
}
```

这个方法很简单，就是根据 path 查找 state 上的嵌套 state。在这里就是传入 rootState 和 path，计算出当前模块的父模块的 state，由于模块的 path 是根据模块的名称 concat 连接的，所以 path 的最后一个元素就是当前模块的模块名，最后调用

```javascript
store._withCommit(() => {
  Vue.set(parentState, moduleName, state || {})
}) 
```
把当前模块的 state 添加到 parentState 中。
这里注意一下我们用了 store._withCommit 方法，来看一下这个方法的定义：

```javascript
_withCommit (fn) {
  const committing = this._committing
  this._committing = true
  fn()
  this._committing = committing
}
```
由于我们是在修改 state，Vuex 中所有对 state 的修改都会用 _withCommit函数包装，保证在同步修改 state 的过程中 this._committing 的值始终为true。这样当我们观测 state 的变化时，如果 this._committing 的值不为 true，则能检查到这个状态修改是有问题的。

看到这里，有些同学可能会有点困惑，举个例子来直观感受一下，以 Vuex 源码中的 example/shopping-cart 为例，打开 store/index.js，有这么一段代码：


```javascript
export default new Vuex.Store({
  actions,
  getters,
  modules: {
    cart,
    products
  },
  strict: debug,
  plugins: debug ? [createLogger()] : []
})
```

这里有两个子 module，cart 和 products，我们打开 store/modules/cart.js，看一下 cart 模块中的 state 定义，代码如下：

```javascript
const state = {
  added: [],
  checkoutStatus: null
}
```

我们运行这个项目，打开浏览器，利用 Vue 的调试工具来看一下 Vuex 中的状态，如下图所示：

![vuex-state](https://camo.githubusercontent.com/2bd2ed9ea9f74792db15c5980666275330ab64a2/687474703a2f2f7374617469632e67616c696c656f2e7869616f6a756b656a692e636f6d2f7374617469632f746d732f736869656c642f767565782d746573742e706e67)

可以看到，在 rootState 下，分别有 cart 和 products 2个属性，key 根据模块名称而来，value 就是在每个模块文件中定义的 state，这就把模块 state 挂载到 rootState 上了。

我们了解完嵌套模块 state 是怎么一回事后，我们回过头来看一下 installModule 过程中的其它 3 个重要方法：registerMutation、registerAction 和 wrapGetters。顾名思义，这 3 个方法分别处理 mutations、actions 和 getters。我们先来看一下 registerMutation 的定义：

### registerMutation

```javascript
function registerMutation (store, type, handler, path = []) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    handler(getNestedState(store.state, path), payload)
  })
}
```
registerMutation 是对 store 的 mutation 的初始化，它接受 4 个参数，store为当前 Store 实例，type为 mutation 的 key，handler 为 mutation 执行的回调函数，path 为当前模块的路径。mutation 的作用就是同步修改当前模块的 state ，函数首先通过 type 拿到对应的 mutation 对象数组， 然后把一个 mutation 的包装函数 push 到这个数组中，这个函数接收一个参数 payload，这个就是我们在定义 mutation 的时候接收的额外参数。这个函数执行的时候会调用 mutation 的回调函数，并通过 getNestedState(store.state, path) 方法得到当前模块的 state，和 playload 一起作为回调函数的参数。举个例子：

```javascript
// ...
mutations: {
  increment (state, n) {
    state.count += n
  }
}
```

这里我们定义了一个 mutation，通过刚才的 registerMutation 方法，我们注册了这个 mutation，这里的 state 对应的就是当前模块的 state，n 就是额外参数 payload，接下来我们会从源码分析的角度来介绍这个 mutation 的回调是何时被调用的，参数是如何传递的。

我们有必要知道 mutation 的回调函数的调用时机，在 Vuex 中，mutation 的调用是通过 store 实例的 API 接口 commit 来调用的，来看一下 commit 函数的定义：

```javascript
commit (type, payload, options) {
  // check object-style commit
  if (isObject(type) && type.type) {
    options = payload
    payload = type
    type = type.type
  }
  const mutation = { type, payload }
  const entry = this._mutations[type]
  if (!entry) {
    console.error(`[vuex] unknown mutation type: ${type}`)
    return
  }
  this._withCommit(() => {
    entry.forEach(function commitIterator (handler) {
      handler(payload)
    })
  })
  if (!options || !options.silent) {
    this._subscribers.forEach(sub => sub(mutation, this.state))
  }
}
```
commit 支持 3 个参数，type 表示 mutation 的类型，payload 表示额外的参数，options 表示一些配置，比如 silent 等，稍后会用到。commit 函数首先对 type 的类型做了判断，处理了 type 为 object 的情况，接着根据 type 去查找对应的 mutation，如果找不到，则输出一条错误信息，否则遍历这个 type 对应的 mutation 对象数组，执行 handler(payload) 方法，这个方法就是之前定义的 wrappedMutationHandler(handler)，执行它就相当于执行了 registerMutation 注册的回调函数，并把当前模块的 state 和 额外参数 payload 作为参数传入。注意这里我们依然使用了 `this._withCommit` 的方法提交 mutation。commit 函数的最后，判断如果不是静默模式，则遍历 `this._subscribers`，调用回调函数，并把 mutation 和当前的根 state 作为参数传入。那么这个 `this._subscribers` 是什么呢？原来 Vuex 的 Store 实例提供了 subscribe API 接口，它的作用是订阅（注册监听） store 的 mutation。先来看一下它的实现：

```javascript
subscribe (fn) {
  const subs = this._subscribers
  if (subs.indexOf(fn) < 0) {
    subs.push(fn)
  }
  return () => {
    const i = subs.indexOf(fn)
    if (i > -1) {
      subs.splice(i, 1)
    }
  }
}
```
subscribe 方法很简单，他接受的参数是一个回调函数，会把这个回调函数保存到 this._subscribers 上，并返回一个函数，当我们调用这个返回的函数，就可以解除当前函数对 store 的 mutation 的监听。其实，Vuex 的内置 logger 插件就是基于 subscribe 接口实现对 store 的 muation的监听，稍后我们会详细介绍这个插件。

### registerAction

在了解完 registerMutation，我们再来看一下 registerAction 的定义：

```javascript
function registerAction (store, type, handler, path = []) {
const entry = store._actions[type] || (store._actions[type] = [])
  const { dispatch, commit } = store
  entry.push(function wrappedActionHandler (payload, cb) {
    let res = handler({
      dispatch,
      commit,
      getters: store.getters,
      state: getNestedState(store.state, path),
      rootState: store.state
    }, payload, cb)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
 }
```
registerAction 是对 store 的 action 的初始化，它和 registerMutation 的参数一致，和 mutation 不同一点，mutation 是同步修改当前模块的 state，而 action 是可以异步去修改 state，这里不要误会，在 action 的回调中并不会直接修改 state ，仍然是通过提交一个 mutation 去修改 state（在 Vuex 中，mutation 是修改 state 的唯一途径）。那我们就来看看 action 是如何做到这一点的。

函数首先也是通过 type 拿到对应 action 的对象数组，然后把一个 action 的包装函数 push 到这个数组中，这个函数接收 2 个参数，payload 表示额外参数 ，cb 表示回调函数（实际上我们并没有使用它）。这个函数执行的时候会调用 action 的回调函数，传入一个 context 对象，这个对象包括了 store 的 commit 和 dispatch 方法、getter、当前模块的 state 和 rootState 等等。接着对这个函数的返回值做判断，如果不是一个 Promise 对象，则调用 `Promise.resolve（res）` 给res 包装成了一个 Promise 对象。这里也就解释了为何 Vuex 的源码依赖 Promise，这里对 Promise 的判断也和简单，参考代码 src/util.js，对 isPromise 的判断如下：

```javascript
export function isPromise (val) {
  return val && typeof val.then === 'function'
}
```
其实就是简单的检查对象的 then 方法，如果包含说明就是一个 Promise 对象。

接着判断 store._devtoolHook，这个只有当用到 Vuex devtools 开启的时候，我们才能捕获 promise 的过程中的 。 action 的包装函数最后返回 res ，它就是一个地地道道的 Promise 对象。来看个例子：

```javascript
actions: {
  checkout ({ commit, state }, payload) {
    // 把当前购物车的商品备份起来
    const savedCartItems = [...state.cart.added]
    // 发送结帐请求，并愉快地清空购物车
    commit(types.CHECKOUT_REQUEST)
    // 购物 API 接收一个成功回调和一个失败回调
    shop.buyProducts(
      products,
      // 成功操作
      () => commit(types.CHECKOUT_SUCCESS),
      // 失败操作
      () => commit(types.CHECKOUT_FAILURE, savedCartItems)
    )
  }
}
```
这里我们定义了一个 action，通过刚才的 registerAction 方法，我们注册了这个 action，这里的 commit 就是 store 的 API 接口，可以通过它在 action 里提交一个 mutation。state 对应的就是当前模块的 state，我们在这个 action 里即可以同步提交 mutation，也可以异步提交。接下来我们会从源码分析的角度来介绍这个 action 的回调是何时被调用的，参数是如何传递的。

我们有必要知道 action 的回调函数的调用时机，在 Vuex 中，action 的调用是通过 store 实例的 API 接口 dispatch 来调用的，来看一下 dispatch 函数的定义：

```javascript
 dispatch (type, payload) {
  // check object-style dispatch
   if (isObject(type) && type.type) {
     payload = type
     type = type.type
   }
   const entry = this._actions[type]
   if (!entry) {
     console.error(`[vuex] unknown action type: ${type}`)
     return
   }
   return entry.length > 1
     ? Promise.all(entry.map(handler => handler(payload)))
     : entry[0](payload)
 }
```
dispatch 支持2个参数，type 表示 action 的类型，payload 表示额外的参数。前面几行代码和 commit 接口非常类似，都是找到对应 type 下的 action 对象数组，唯一和 commit 不同的地方是最后部分，它对 action 的对象数组长度做判断，如果长度为 1 则直接调用 entry[0](payload)， 这个方法就是之前定义的 wrappedActionHandler(payload, cb)，执行它就相当于执行了 registerAction 注册的回调函数，并把当前模块的 context 和 额外参数 payload 作为参数传入。所以我们在 action 的回调函数里，可以拿到当前模块的上下文包括 store 的 commit 和 dispatch 方法、getter、当前模块的 state 和 rootState，可见 action 是非常灵活的。

### wrapGetters

了解完 registerAction 后，我们来看看 wrapGetters的定义：

```javascript
function wrapGetters (store, moduleGetters, modulePath) {
  Object.keys(moduleGetters).forEach(getterKey => {
    const rawGetter = moduleGetters[getterKey]
    if (store._wrappedGetters[getterKey]) {
      console.error(`[vuex] duplicate getter key: ${getterKey}`)
      return
    }
    store._wrappedGetters[getterKey] = function wrappedGetter (store) {
      return rawGetter(
        getNestedState(store.state, modulePath), // local state
        store.getters, // getters
        store.state // root state
      )
    }
  })
}
```
wrapGetters 是对 store 的 getters 初始化，它接受 3个 参数， store 表示当前 Store 实例，moduleGetters 表示当前模块下的所有 getters, modulePath 对应模块的路径。细心的同学会发现，和刚才的 registerMutation 以及 registerAction 不同，这里对 getters 的循环遍历是放在了函数体内，并且 getters 和它们的一个区别是不允许 getter 的 key 有重复。

这个函数做的事情就是遍历 moduleGetters，把每一个 getter 包装成一个方法，添加到 store._wrappedGetters 对象中，注意 getter 的 key 是不允许重复的。在这个包装的方法里，会执行 getter 的回调函数，并把当前模块的 state，store 的 getters 和 store 的 rootState 作为它参数。来看一个例子：

```javascript
export const cartProducts = state => {
  return state.cart.added.map(({ id, quantity }) => {
    const product = state.products.all.find(p => p.id === id)
    return {
      title: product.title,
      price: product.price,
      quantity
    }
  })
}
```
这里我们定义了一个 getter，通过刚才的 wrapGetters 方法，我们把这个 getter 添加到 store._wrappedGetters 对象里，这和回调函数的参数 state 对应的就是当前模块的 state，接下来我们从源码的角度分析这个函数是如何被调用，参数是如何传递的。

我们有必要知道 getter 的回调函数的调用时机，在 Vuex 中，我们知道当我们在组件中通过 this.$store.getters.xxxgetters 可以访问到对应的 getter 的回调函数，那么我们需要把对应 getter 的包装函数的执行结果绑定到 ````this.$store``` 上。这部分的逻辑就在 resetStoreVM 函数里。我们在 Store 的构造函数中，在执行完 installModule 方法后，就会执行 resetStoreVM 方法。来看一下它的定义：

### resetStoreVM

```javascript
function resetStoreVM (store, state) {
  const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  Object.keys(wrappedGetters).forEach(key => {
    const fn = wrappedGetters[key]
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key]
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: { state },
    computed
  })
  Vue.config.silent = silent

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store)
  }

  if (oldVm) {
    // dispatch changes in all subscribed watchers
    // to force getter re-evaluation.
    store._withCommit(() => {
      oldVm.state = null
    })
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```

这个方法主要是重置一个私有的 _vm 对象，它是一个 Vue 的实例。这个 _vm 对象会保留我们的 state 树，以及用计算属性的方式存储了 store 的 getters。来具体看看它的实现过程。我们把这个函数拆成几个部分来分析：


```javascript
const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  Object.keys(wrappedGetters).forEach(key => {
    const fn = wrappedGetters[key]
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key]
    })
  })
```

这部分留了现有的 store._vm 对象，接着遍历 store._wrappedGetters 对象，在遍历过程中，依次拿到每个 getter 的包装函数，并把这个包装函数执行的结果用 computed 临时变量保存。接着用 es5 的 Object.defineProperty 方法为 store.getters 定义了 get 方法，也就是当我们在组件中调用this.$store.getters.xxxgetters 这个方法的时候，会访问 store._vm[xxxgetters]。我们接着往下看：


```javascript
// use a Vue instance to store the state tree
// suppress warnings just in case the user has added
 // some funky global mixins
 const silent = Vue.config.silent
 Vue.config.silent = true
 store._vm = new Vue({
   data: { state },
   computed
 })
 Vue.config.silent = silent

 // enable strict mode for new vm
 if (store.strict) {
   enableStrictMode(store)
 }
```

这部分的代码首先先拿全局 Vue.config.silent 的配置，然后临时把这个配置设成 true，接着实例化一个 Vue 的实例，把 store 的状态树 state 作为 data 传入，把我们刚才的临时变量 computed 作为计算属性传入。然后再把之前的 silent 配置重置。设置 silent 为 true 的目的是为了取消这个 _vm 的所有日志和警告。把 computed 对象作为 _vm 的 computed 属性，这样就完成了 getters 的注册。因为当我们在组件中访问 `this.$store.getters.xxxgetters` 的时候，就相当于访问 `store._vm[xxxgetters]`，也就是在访问 computed[xxxgetters]，这样就访问到了 xxxgetters 对应的回调函数了。这段代码最后判断 strict 属性决定是否开启严格模式，我们来看看严格模式都干了什么：

```javascript
function enableStrictMode (store) {
  store._vm.$watch('state', () => {
    assert(store._committing, `Do not mutate vuex store state outside mutation handlers.`)
  }, { deep: true, sync: true })
}
```

严格模式做的事情很简单，监测 store._vm.state 的变化，看看 state 的变化是否通过执行 mutation 的回调函数改变，如果是外部直接修改 state，那么 `store._committing` 的值为 false，这样就抛出一条错误。再次强调一下，Vuex 中对 state 的修改只能在 mutation 的回调函数里。

回到 resetStoreVM 函数，我们来看一下最后一部分：


```javascript
if (oldVm) {
  // dispatch changes in all subscribed watchers
  // to force getter re-evaluation.
  store._withCommit(() => {
    oldVm.state = null
  })
  Vue.nextTick(() => oldVm.$destroy())
}
```

这里的逻辑很简单，由于这个函数每次都会创建新的 Vue 实例并赋值到 store._vm 上，那么旧的 _vm 对象的状态设置为 null，并调用 $destroy 方法销毁这个旧的 _vm 对象。

那么到这里，Vuex 的初始化基本告一段落了，初始化核心就是 installModule 和
resetStoreVM 函数。通过对 mutations 、actions 和 getters 的注册，我们了解到 state 的是按模块划分的，按模块的嵌套形成一颗状态树。而 actions、mutations 和 getters 的全局的，其中 actions 和 mutations 的 key 允许重复，但 getters 的 key 是不允许重复的。官方推荐我们给这些全局的对象在定义的时候加一个名称空间来避免命名冲突。
从源码的角度介绍完 Vuex 的初始化的玩法，我们再从 Vuex 提供的 API 方向来分析其中的源码，看看这些 API 是如何实现的。

## Vuex API 分析

Vuex 常见的 API 如 dispatch、commit 、subscribe �我们前面已经介绍过了，这里就不再赘述了，下面介绍的一些 Store 的 API，虽然不常用，�但是了解一下也不错。�

### watch(getter, cb, options)

watch 作用是响应式的监测一个 getter 方法的返回值，当值改变时调用回调。getter 接收 store 的 state 作为唯一参数。来看一下它的实现：


```javascript
watch (getter, cb, options) {
    assert(typeof getter === 'function', `store.watch only accepts a function.`)
    return this._watcherVM.$watch(() => getter(this.state), cb, options)
  }
```

函数首先断言 watch 的 getter 必须是一个方法，接着利用了内部一个 Vue 的实例对象 `this._watcherVM` 的 $watch 方法，观测 getter 方法返回值的变化，如果有变化则调用 cb 函数，回调函数的参数为新值和旧值。watch 方法返回的是一个方法，调用它则取消观测。

### registerModule(path, module)

registerModule 的作用是注册一个动态模块，有的时候当我们异步加载一些业务的时候，可以通过这个 API 接口去动态注册模块，来看一下它的实现：

```javascript
registerModule (path, module) {
    if (typeof path === 'string') path = [path]
    assert(Array.isArray(path), `module path must be a string or an Array.`)
    this._runtimeModules[path.join('.')] = module
    installModule(this, this.state, path, module)
    // reset store to update getters...
    resetStoreVM(this, this.state)
```
函数首先对 path 判断，如果 path 是一个 string 则把 path 转换成一个 Array。接着把 module 对象缓存到 this._runtimeModules 这个对象里，path 用点连接作为该对象的 key。接着和初始化 Store 的逻辑一样，调用 installModule 和 resetStoreVm 方法安装一遍动态注入的 module。

### unregisterModule(path)

和 registerModule 方法相对的就是 unregisterModule 方法，它的作用是注销一个动态模块，来看一下它的实现：

```javascript
unregisterModule (path) {
    if (typeof path === 'string') path = [path]
    assert(Array.isArray(path), `module path must be a string or an Array.`)
    delete this._runtimeModules[path.join('.')]
    this._withCommit(() => {
      const parentState = getNestedState(this.state, path.slice(0, -1))
      Vue.delete(parentState, path[path.length - 1])
    })
    resetStore(this)
  }
```
函数首先还是对 path 的类型做了判断，这部分逻辑和注册是一样的。接着从 this._runtimeModules 里删掉以 path 点连接的 key 对应的模块。接着通过 `this._withCommit` 方法把当前模块的 state 对象从父 state 上删除。最后调用 resetStore(this) 方法，来看一下这个方法的定义：

```javascript
function resetStore (store) {
  store._actions = Object.create(null)
  store._mutations = Object.create(null)
  store._wrappedGetters = Object.create(null)
  const state = store.state
  // init root module
  installModule(store, state, [], store._options, true)
  // init all runtime modules
  Object.keys(store._runtimeModules).forEach(key => {
    installModule(store, state, key.split('.'), store._runtimeModules[key], true)
  })
  // reset vm
  resetStoreVM(store, state)
}
```
这个方法作用就是重置 store 对象，重置 store 的 _actions、_mutations、_wrappedGetters 等等属性。然后再次调用 installModules 去重新安装一遍 Module 对应的这些属性，注意这里我们的最后一个参数 hot 为true，表示它是一次热更新。这样在 installModule 这个方法体类，如下这段逻辑就不会执行


```javascript
function installModule (store, rootState, path, module, hot) {
  ... 
  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, state || {})
    })
  }
  ...
}
```

由于 hot 始终为 true，这里我们就不会重新对状态树做设置，我们的 state 保持不变。因为我们已经明确的删除了对应 path 下的 state 了，要做的事情只不过就是重新注册一遍 muations、actions 以及 getters。

回调 resetStore 方法，接下来遍历 this._runtimeModules 模块，重新安装所有剩余的 runtime Moudles。最后还是调用 resetStoreVM 方法去重置 Store 的 _vm 对象。

### hotUpdate(newOptions)

hotUpdate 的作用是热加载新的 action 和 mutation。 来看一下它的实现：

```javascript
hotUpdate (newOptions) {
  updateModule(this._options, newOptions)
  resetStore(this)
}
```

函数首先调用 updateModule 方法去更新状态，其中当前 Store 的 opition 配置和要更新的 newOptions 会作为参数。来看一下这个函数的实现：

```javascript
function updateModule (targetModule, newModule) {
  if (newModule.actions) {
    targetModule.actions = newModule.actions
  }
  if (newModule.mutations) {
    targetModule.mutations = newModule.mutations
  }
  if (newModule.getters) {
    targetModule.getters = newModule.getters
  }
  if (newModule.modules) {
    for (const key in newModule.modules) {
      if (!(targetModule.modules && targetModule.modules[key])) {
        console.warn(
          `[vuex] trying to add a new module '${key}' on hot reloading, ` +
          'manual reload is needed'
        )
        return
      }
      updateModule(targetModule.modules[key], newModule.modules[key])
    }
  }
}
```

首先我们对 newOptions 对象的 actions、mutations 以及 getters 做了判断，如果有这些属性的话则替换 targetModule（当前 Store 的 options）对应的属性。最后判断如果 newOptions 包含 modules 这个 key，则遍历这个 modules 对象，如果 modules 对应的 key 不在之前的 modules 中，则报一条警告，因为这是添加一个新的 module ，需要手动重新加载。如果 key 在之前的 modules，则递归调用 updateModule，热更新子模块。

调用完 updateModule 后，回到 hotUpdate 函数，接着调用 resetStore 方法重新设置 store，刚刚我们已经介绍过了。

### replaceState

replaceState的作用是替换整个 rootState，一般在用于调试，来看一下它的实现：

```javascript
replaceState (state) {
    this._withCommit(() => {
      this._vm.state = state
    })
  }
```
函数非常简单，就是调用 this._withCommit 方法修改 Store 的 rootState，之所以提供这个 API 是由于在我们是不能在 muations 的回调函数外部去改变 state。

到此为止，API 部分介绍完了，其实整个 Vuex 源码下的 src/index.js 文件里的代码基本都过了一遍。

##辅助函数

Vuex 除了提供我们 Store 对象外，还对外提供了一系列的辅助函数，方便我们在代码中使用 Vuex，提供了操作 store 的各种属性的一系列语法糖，下面我们来一起看一下：

### mapState

mapState 工具函数会将 store 中的 state 映射到局部计算属性中。为了更好理解它的实现，先来看一下它的使用示例：

```javascript
// vuex 提供了独立的构建工具函数 Vuex.mapState
import { mapState } from 'vuex'
export default {
  // ...
  computed: mapState({
    // 箭头函数可以让代码非常简洁
    count: state => state.count,
    // 传入字符串 'count' 等同于 `state => state.count`
    countAlias: 'count',
    // 想访问局部状态，就必须借助于一个普通函数，函数中使用 `this` 获取局部状态
    countPlusLocalState (state) {
      return state.count + this.localCount
    }
  })
}
```

当计算属性名称和状态子树名称对应相同时，我们可以向 mapState 工具函数传入一个字符串数组。

```javascript
computed: mapState([
  // 映射 this.count 到 this.$store.state.count
  'count'
```
通过例子我们可以直观的看到，mapState 函数可以接受一个对象，也可以接收一个数组，那它底层到底干了什么事呢，我们一起来看一下源码这个函数的定义：

```javascript
export function mapState (states) {
  const res = {}
  normalizeMap(states).forEach(({ key, val }) => {
    res[key] = function mappedState () {
      return typeof val === 'function'
        ? val.call(this, this.$store.state, this.$store.getters)
        : this.$store.state[val]
    }
  })
  return res
}
```
函数首先对传入的参数调用 normalizeMap 方法，我们来看一下这个函数的定义：

```javascript
function normalizeMap (map) {
  return Array.isArray(map)
    ? map.map(key => ({ key, val: key }))
    : Object.keys(map).map(key => ({ key, val: map[key] }))
}
```
这个方法判断参数 map 是否为数组，如果是数组，则调用数组的 map 方法，把数组的每个元素转换成一个 `{key, val: key}`的对象；否则传入的 map 就是一个对象（从 mapState 的使用场景来看，传入的参数不是数组就是对象），我们调用 `Object.keys` 方法遍历这个 map 对象的 key，把数组的每个 key 都转换成一个 `{key, val: key}`的对象。最后我们把这个对象数组作为 normalizeMap 的返回值。

回到 mapState 函数，在调用了 normalizeMap 函数后，把传入的 states 转换成由 {key, val} 对象构成的数组，接着调用 forEach 方法遍历这个数组，构造一个新的对象，这个新对象每个元素都返回一个新的函数 mappedState，函数对 val 的类型判断，如果 val 是一个函数，则直接调用这个 val 函数，把当前 store 上的 state 和 getters 作为参数，返回值作为 mappedState 的返回值；否则直接把 `this.$store.state[val]` 作为 mappedState 的返回值。

那么为何 mapState 函数的返回值是这样一个对象呢，因为 mapState 的作用是把全局的 state 和 getters 映射到当前组件的 computed 计算属性中，我们知道在 Vue 中 每个计算属性都是一个函数。

为了更加直观地说明，回到刚才的例子：

```javascript
import { mapState } from 'vuex'
export default {
  // ...
  computed: mapState({
    // 箭头函数可以让代码非常简洁
    count: state => state.count,
    // 传入字符串 'count' 等同于 `state => state.count`
    countAlias: 'count',
    // 想访问局部状态，就必须借助于一个普通函数，函数中使用 `this` 获取局部状态
    countPlusLocalState (state) {
      return state.count + this.localCount
    }
  })
}
```
经过 mapState 函数调用后的结果，如下所示：

```javascript
import { mapState } from 'vuex'
export default {
  // ...
  computed: {
    count() {
	  return this.$store.state.count
    },
    countAlias() {
	  return this.$store.state['count']
    },
    countPlusLocalState() {
      return this.$store.state.count + this.localCount
    }
  }
}
```
我们再看一下 mapState 参数为数组的例子：

```javascript
computed: mapState([
  // 映射 this.count 到 this.$store.state.count
  'count'
])
```
经过 mapState 函数调用后的结果，如下所示：

```javascript
computed: {
  count() {
    return this.$store.state['count']
  }
}
```

### mapGetters

mapGetters 工具函数会将 store 中的 getter 映射到局部计算属性中。它的功能和 mapState 非常类似，我们来直接看它的实现：

```javascript
export function mapGetters (getters) {
  const res = {}
  normalizeMap(getters).forEach(({ key, val }) => {
    res[key] = function mappedGetter () {
      if (!(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
      }
      return this.$store.getters[val]
    }
  })
  return res
}
```

mapGetters 的实现也和 mapState 很类似，不同的是它的 val 不能是函数，只能是一个字符串，而且会检查 `val in this.$store.getters` 的值，如果为 false 会输出一条错误日志。为了更直观地理解，我们来看一个简单的例子：

```javascript
import { mapGetters } from 'vuex'
export default {
  // ...
  computed: {
    // 使用对象扩展操作符把 getter 混入到 computed 中
    ...mapGetters([
      'doneTodosCount',
      'anotherGetter',
      // ...
    ])
  }
}
```
经过 mapGetters 函数调用后的结果，如下所示：

```javascript
import { mapGetters } from 'vuex'
export default {
  // ...
  computed: {
    doneTodosCount() {
      return this.$store.getters['doneTodosCount']
    },
    anotherGetter() {
      return this.$store.getters['anotherGetter']
    }
  }
}
```

再看一个参数 mapGetters 参数是对象的例子：

```javascript
computed: mapGetters({
  // 映射 this.doneCount 到 store.getters.doneTodosCount
  doneCount: 'doneTodosCount'
})
```

经过 mapGetters 函数调用后的结果，如下所示：

```javascript
computed: {
  doneCount() {
    return this.$store.getters['doneTodosCount']
  }
}
```

### mapActions

mapActions 工具函数会将 store 中的 dispatch 方法映射到组件的 methods 中。和 mapState、mapGetters 也类似，只不过它映射的地方不是计算属性，而是组件的 methods 对象上。我们来直接看它的实现：

```javascript
export function mapActions (actions) {
  const res = {}
  normalizeMap(actions).forEach(({ key, val }) => {
    res[key] = function mappedAction (...args) {
      return this.$store.dispatch.apply(this.$store, [val].concat(args))
    }
  })
  return res
}
```

可以看到，函数的实现套路和 mapState、mapGetters 差不多，甚至更简单一些， 实际上就是做了一层函数包装。为了更直观地理解，我们来看一个简单的例子：

```javascript
import { mapActions } from 'vuex'
export default {
  // ...
  methods: {
    ...mapActions([
      'increment' // 映射 this.increment() 到 this.$store.dispatch('increment')
    ]),
    ...mapActions({
      add: 'increment' // 映射 this.add() to this.$store.dispatch('increment')
    })
  }
}
```
经过 mapActions 函数调用后的结果，如下所示：

```javascript
import { mapActions } from 'vuex'
export default {
  // ...
  methods: {
    increment(...args) {
      return this.$store.dispatch.apply(this.$store, ['increment'].concat(args))
	}
    add(...args) {
      return this.$store.dispatch.apply(this.$store, ['increment'].concat(args))
	}
  }
}
```

### mapMutations

mapMutations 工具函数会将 store 中的 commit 方法映射到组件的 methods 中。和 mapActions 的功能几乎一样，我们来直接看它的实现：

```javascript
export function mapMutations (mutations) {
  const res = {}
  normalizeMap(mutations).forEach(({ key, val }) => {
    res[key] = function mappedMutation (...args) {
      return this.$store.commit.apply(this.$store, [val].concat(args))
    }
  })
  return res
}
```

函数的实现几乎也和 mapActions 一样，唯一差别就是映射的是 store 的 commit 方法。为了更直观地理解，我们来看一个简单的例子：

```javascript
import { mapMutations } from 'vuex'
export default {
  // ...
  methods: {
    ...mapMutations([
      'increment' // 映射 this.increment() 到 this.$store.commit('increment')
    ]),
    ...mapMutations({
      add: 'increment' // 映射 this.add() 到 this.$store.commit('increment')
    })
  }
}
```
经过 mapMutations 函数调用后的结果，如下所示：

```javascript
import { mapActions } from 'vuex'
export default {
  // ...
  methods: {
    increment(...args) {
      return this.$store.commit.apply(this.$store, ['increment'].concat(args))
	}
    add(...args) {
      return this.$store.commit.apply(this.$store, ['increment'].concat(args))
	}
  }
}
```

###插件

Vuex 的 store 接收 plugins 选项，一个 Vuex 的插件就是一个简单的方法，接收 store 作为唯一参数。插件作用通常是用来监听每次 mutation 的变化，来做一些事情。

在 store 的构造函数的最后，我们通过如下代码调用插件：

```javascript
import devtoolPlugin from './plugins/devtool'

// apply plugins
plugins.concat(devtoolPlugin).forEach(plugin => plugin(this))
```

我们通常实例化 store 的时候，还会调用 logger 插件，代码如下：

```javascript
import Vue from 'vue'
import Vuex from 'vuex'
import createLogger from 'vuex/dist/logger'

Vue.use(Vuex)

const debug = process.env.NODE_ENV !== 'production'

export default new Vuex.Store({
  ...
  plugins: debug ? [createLogger()] : []
})
```

在上述 2 个例子中，我们分别调用了 devtoolPlugin 和 createLogger() 2 个插件，它们是 Vuex 内置插件，我们接下来分别看一下他们的实现。

### devtoolPlugin

devtoolPlugin 主要功能是利用 Vue 的开发者工具和 Vuex 做配合，通过开发者工具的面板展示 Vuex 的状态。它的源码在 src/plugins/devtool.js 中，来看一下这个插件到底做了哪些事情。

```javascript
const devtoolHook =
  typeof window !== 'undefined' &&
  window.__VUE_DEVTOOLS_GLOBAL_HOOK__

export default function devtoolPlugin (store) {
  if (!devtoolHook) return

  store._devtoolHook = devtoolHook

  devtoolHook.emit('vuex:init', store)

  devtoolHook.on('vuex:travel-to-state', targetState => {
    store.replaceState(targetState)
  })

  store.subscribe((mutation, state) => {
    devtoolHook.emit('vuex:mutation', mutation, state)
  })
}
```

我们直接从对外暴露的 devtoolPlugin 函数看起，函数首先判断了devtoolHook 的值，如果我们浏览器装了 Vue 开发者工具，那么在 window 上就会有一个 __VUE_DEVTOOLS_GLOBAL_HOOK__ 的引用， 那么这个 devtoolHook 就指向这个引用。

接下来通过 `devtoolHook.emit('vuex:init', store)` 派发一个 Vuex 初始化的事件，这样开发者工具就能拿到当前这个 store 实例。

接下来通过 `devtoolHook.on('vuex:travel-to-state', targetState => { store.replaceState(targetState) })`监听 Vuex 的 traval-to-state 的事件，把当前的状态树替换成目标状态树，这个功能也是利用 Vue 开发者工具替换 Vuex 的状态。

最后通过 `store.subscribe((mutation, state) => { devtoolHook.emit('vuex:mutation', mutation, state) })` 方法订阅 store 的 state 的变化，当 store 的 mutation 提交了 state 的变化， 会触发回调函数——通过 devtoolHook 派发一个 Vuex mutation 的事件，mutation 和 rootState 作为参数，这样开发者工具就可以观测到 Vuex state 的实时变化，在面板上展示最新的状态树。

### loggerPlugin

通常在开发环境中，我们希望实时把 mutation 的动作以及 store 的 state 的变化实时输出，那么我们可以用 loggerPlugin 帮我们做这个事情。它的源码在 src/plugins/logger.js 中，来看一下这个插件到底做了哪些事情。

```javascript
// Credits: borrowed code from fcomb/redux-logger

import { deepCopy } from '../util'

export default function createLogger ({
  collapsed = true,
  transformer = state => state,
  mutationTransformer = mut => mut
} = {}) {
  return store => {
    let prevState = deepCopy(store.state)

    store.subscribe((mutation, state) => {
      if (typeof console === 'undefined') {
        return
      }
      const nextState = deepCopy(state)
      const time = new Date()
      const formattedTime = ` @ ${pad(time.getHours(), 2)}:${pad(time.getMinutes(), 2)}:${pad(time.getSeconds(), 2)}.${pad(time.getMilliseconds(), 3)}`
      const formattedMutation = mutationTransformer(mutation)
      const message = `mutation ${mutation.type}${formattedTime}`
      const startMessage = collapsed
        ? console.groupCollapsed
        : console.group

      // render
      try {
        startMessage.call(console, message)
      } catch (e) {
        console.log(message)
      }

      console.log('%c prev state', 'color: #9E9E9E; font-weight: bold', transformer(prevState))
      console.log('%c mutation', 'color: #03A9F4; font-weight: bold', formattedMutation)
      console.log('%c next state', 'color: #4CAF50; font-weight: bold', transformer(nextState))

      try {
        console.groupEnd()
      } catch (e) {
        console.log('—— log end ——')
      }

      prevState = nextState
    })
  }
}

function repeat (str, times) {
  return (new Array(times + 1)).join(str)
}

function pad (num, maxLength) {
  return repeat('0', maxLength - num.toString().length) + num
}
```

插件对外暴露的是 createLogger 方法，它实际上接受 3 个参数，它们都有默认值，通常我们用默认值就可以。createLogger 的返回的是一个函数，当我执行 logger 插件的时候，实际上执行的是这个函数，下面来看一下这个函数做了哪些事情。

函数首先执行了 `let prevState = deepCopy(store.state)` 深拷贝当前 store 的 rootState。这里为什么要深拷贝，因为如果是单纯的引用，那么 `store.state` 的任何变化都会影响这个引用，这样就无法记录上一个状态了。我们来了解一下 deepCopy 的实现，在 src/util.js 里定义：

```javascript
function find (list, f) {
  return list.filter(f)[0]
}

export function deepCopy (obj, cache = []) {
  // just return if obj is immutable value
  if (obj === null || typeof obj !== 'object') {
    return obj
  }

  // if obj is hit, it is in circular structure
  const hit = find(cache, c => c.original === obj)
  if (hit) {
    return hit.copy
  }

  const copy = Array.isArray(obj) ? [] : {}
  // put the copy into cache at first
  // because we want to refer it in recursive deepCopy
  cache.push({
    original: obj,
    copy
  })

  Object.keys(obj).forEach(key => {
    copy[key] = deepCopy(obj[key], cache)
  })

  return copy
}
```

deepCopy 并不陌生，很多开源库如 loadash、jQuery 都有类似的实现，原理也不难理解，主要是构造一个新的对象，遍历原对象或者数组，递归调用 deepCopy。不过这里的实现有一个有意思的地方，在每次执行 deepCopy 的时候，会用 cache 数组缓存当前嵌套的对象，以及执行 deepCopy 返回的 copy。如果在 deepCopy 的过程中通过 `find(cache, c => c.original === obj)` 发现有循环引用的时候，直接返回 cache 中对应的 copy，这样就避免了无限循环的情况。

回到 loggerPlugin 函数，通过 deepCopy 拷贝了当前 state 的副本并用 prevState 变量保存，接下来调用 `store.subscribe` 方法订阅 store 的 state 的变。 在回调函数中，也是先通过 deepCopy 方法拿到当前的 state 的副本，并用 nextState 变量保存。接下来获取当前格式化时间已经格式化的 mutation 变化的字符串，然后利用 console.group 以及 console.log 分组输出 prevState、mutation以及 nextState，这里可以通过我们 createLogger 的参数 collapsed、transformer 以及 mutationTransformer 来控制我们最终 log 的显示效果。在函数的最后，我们把 nextState 赋值给 prevState，便于下一次 mutation。

# 总结
--------
Vuex 2.0 的源码分析到这就告一段落了，最后我再分享一下看源码的小心得：对于一个库或者框架源码的研究前，首先了解他们的使用场景、官网文档等；然后一定要用他，至少也要写几个小 demo，达到熟练掌握的程度；最后再从入口、API、使用方法等等多个维度去了解他内部的实现细节。如果这个库过于庞大，那就先按模块和功能拆分，一点点地消化。

最后还有一个问题，有些同学会问，源码那么枯燥，我们分析学习它的有什么好处呢？首先，学习源码有助于我们更深入掌握和应用这个库或者框架；其次，我们还可以学习到源码中很多编程技巧，可以迁移到我们平时的开发工作中；最后，对于一些高级开发工程师而言，我们可以学习到它的设计思想，对将来有一天我们也去设计一个库或者框架是非常有帮助的，这也是提升自身能力水平的非常好的途径。


