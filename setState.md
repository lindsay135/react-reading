<h1>React 15.6.0</h1>

# setState
`setState()` 将对组件 state 的更改排入队列批量推迟更新，并通知 React 需要使用更新后的 state 重新渲染此组件及其子组件。其实setState实际上不是异步，只是代码执行顺序不同，有了异步的感觉。
> 使用方法 `setState(stateChange | updater [, callback])`
* stateChange - 作为被传入的对象，将被浅层合并到新的 state 中
* updater - `(state, props) => stateChange`，返回基于 state 和 props 构建的新对象，将被浅层合并到新的 state 中
* callback - 为可选的回调函数
> 使用 `setState()` 改变状态之后，立刻通过this.state拿不到最新的状态

可以使用 `componentDidUpdate()` 或者 `setState(updater, callback)` 中的回调函数 `callback` 保证在应用更新后触发，通常建议使用 `componentDidUpdate()`

> 多次`setState()`函数调用产生的效果会合并

为了更好的感知性能，React 会在同一周期内会对多个 `setState()` 进行批处理。通过触发一次组件的更新来引发回流。后调用的 `setState()` 将覆盖同一周期内先调用 `setState()` 的值。所以如果是下一个 state 依赖前一个 state 的话，推荐给 `setState()` 传 function
```
onClick = () => {
    this.setState({ quantity: this.state.quantity + 1 });
    this.setState({ quantity: this.state.quantity + 1 });
}
// react中，这个方法最终会变成
Object.assign(
    previousState,
    {quantity: state.quantity + 1},
    {quantity: state.quantity + 1},
    ...
)
```
> 同步 | 异步更新
* 同步更新
    * React 引发的事件处理（比如通过onClick引发的事件处理）
    * React 生命周期函数
* 异步更新
    * 绕过React通过 addEventListener 直接添加的事件处理函数
    * 通过 setTimeout || setInterval 产生的异步调用

# setState()被调用之后，源码执行栈
> react 参照版本 **15.6.0** 最新版本是**16.12.0**
## 1. `setState()`
> 源码路径 [`src/isomorphic/modern/class/ReactBaseClasses.js`](https://github.com/facebook/react/blob/v15.6.0/src/isomorphic/modern/class/ReactBaseClasses.js#L35)

React组件继承自React.Component，而setState是React.Component的方法，因此对于组件来讲setState属于其原型方法 
```
ReactComponent.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
}
```
## 2. `enqueueSetState(); enqueueCallback()`
> 源码路径 [`src/renderers/shared/stack/reconciler/ReactUpdateQueue.js`](https://github.com/facebook/react/blob/v15.6.0/src/renderers/shared/stack/reconciler/ReactUpdateQueue.js#L222)
 
这个文件导出了一个 ReactUpdateQueue 对象（React更新队列）
```
enqueueSetState: function(publicInstance, partialState) {
    var internalInstance = getInternalInstanceReadyForUpdate(
      publicInstance,
      'setState',
    );
    if (!internalInstance) {
      return;
    }
    var queue =
      internalInstance._pendingStateQueue ||
      (internalInstance._pendingStateQueue = []);
    queue.push(partialState);
    enqueueUpdate(internalInstance);
}
enqueueCallback: function(publicInstance, callback, callerName) {
    ReactUpdateQueue.validateCallback(callback, callerName);
    var internalInstance = getInternalInstanceReadyForUpdate(publicInstance);
    if (!internalInstance) {
      return null;
    }
    if (internalInstance._pendingCallbacks) {
      internalInstance._pendingCallbacks.push(callback);
    } else {
      internalInstance._pendingCallbacks = [callback];
    }
    enqueueUpdate(internalInstance);
  }
```
## 3. `enqueueUpdate()`
> 源码路径 [`src/renderers/shared/stack/reconciler/ReactUpdates.js`](https://github.com/facebook/react/blob/v15.6.0/src/renderers/shared/stack/reconciler/ReactUpdates.js#L209)

如果处于批量更新模式，也就是 isBatchingUpdates 为 true 时，不进行state的更新操作，而是将需要更新的 component 添加到 dirtyComponents 数组中。

如果不处于批量更新模式，对所有队列中的更新执行 batchedUpdates 方法。
```
function enqueueUpdate(component) {
  ensureInjected();
  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }
  dirtyComponents.push(component);
  if (component._updateBatchNumber == null) {
    component._updateBatchNumber = updateBatchNumber + 1;
  }
}
```
## 4. `batchedUpdates()`
> 源码路径 [`src/renderers/shared/stack/reconciler/ReactDefaultBatchingStrategy.js`](https://github.com/facebook/react/blob/v15.6.0/src/renderers/shared/stack/reconciler/ReactDefaultBatchingStrategy.js)

如果 isBatchingUpdates 为 true，当前正处于更新事务状态中，则将 Component 存入 dirtyComponent 中， 否则调用 batchedUpdates 处理，发起一个 transaction.perform()。
```
var transaction = new ReactDefaultBatchingStrategyTransaction();
var ReactDefaultBatchingStrategy = {
  isBatchingUpdates: false,
  batchedUpdates: function(callback, a, b, c, d, e) {
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;
    ReactDefaultBatchingStrategy.isBatchingUpdates = true;
    if (alreadyBatchingUpdates) {
      return callback(a, b, c, d, e);
    } else {
      return transaction.perform(callback, null, a, b, c, d, e);
    }
  },
};
```
## 5. transaction initialize and close
> 源码路径 [`src/renderers/shared/stack/reconciler/ReactDefaultBatchingStrategy.js`](https://github.com/facebook/react/blob/v15.6.0/src/renderers/shared/stack/reconciler/ReactDefaultBatchingStrategy.js)

Transaction 中注册了两个 wrapper，`RESET_BATCHED_UPDATES` 和 `FLUSH_BATCHED_UPDATES`。

initialize 阶段，两个 wrapper 都是空方法，什么都不做。

close 阶段，`RESET_BATCHED_UPDATES` 将 isBatchingUpdates 设置为false；`FLUSH_BATCHED_UPDATES` 运行 flushBatchedUpdates 执行update。
```
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function() {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  },
};
var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
};
```
## 6. 渲染更新
> `ReactUpdates.flushBatchedUpdates() - ReactUpdates.runBatchedUpdates() - ReactCompositeComponent.performUpdateIfNecessary() - receiveComponent() + updateComponent()`

1. runBatchedUpdates循环遍历dirtyComponents数组，主要干两件事:
    * 首先执行 performUpdateIfNecessary 来刷新组件的 view
    * 执行之前阻塞的 callback

2. receiveComponent 最后会调用 updateComponent 
3. updateComponent 中会执行 React 组件存在期的生命周期方法，如 componentWillReceiveProps， shouldComponentUpdate， componentWillUpdate，render, componentDidUpdate。 从而完成组件更新的整套流程

4. 在shouldComponentUpdate之前，执行了_processPendingState方法,该方法主要对state进行处理：
    * 如果更新队列为null，那么返回原来的state；
    * 如果更新队列有一个更新，那么返回更新值；
    * 如果更新队列有多个更新，那么通过for循环将它们合并；

5. 在一个生命周期内，在componentShouldUpdate执行之前，所有的state变化都会被合并，最后统一处理。

`flushBatchedUpdates(); runBatchedUpdates()` 源码路径 [`src/renderers/shared/stack/reconciler/ReactUpdates.js`](https://github.com/facebook/react/blob/v15.6.0/src/renderers/shared/stack/reconciler/ReactUpdates.js#L187)
```
var flushBatchedUpdates = function() {
  while (dirtyComponents.length || asapEnqueued) {
    if (dirtyComponents.length) {
      var transaction = ReactUpdatesFlushTransaction.getPooled();
      transaction.perform(runBatchedUpdates, null, transaction);
      ReactUpdatesFlushTransaction.release(transaction);
    }

    if (asapEnqueued) {
      asapEnqueued = false;
      var queue = asapCallbackQueue;
      asapCallbackQueue = CallbackQueue.getPooled();
      queue.notifyAll();
      CallbackQueue.release(queue);
    }
  }
};
function runBatchedUpdates(transaction) {
  var len = transaction.dirtyComponentsLength;
  dirtyComponents.sort(mountOrderComparator);
  updateBatchNumber++;

  for (var i = 0; i < len; i++) {
    // dirtyComponents中取出一个component
    var component = dirtyComponents[i];
    // 取出dirtyComponent中的未执行的callback
    var callbacks = component._pendingCallbacks;
    component._pendingCallbacks = null;

    var markerName;
    if (ReactFeatureFlags.logTopLevelRenders) {
      var namedComponent = component;
      if (component._currentElement.type.isReactTopLevelWrapper) {
        namedComponent = component._renderedComponent;
      }
      markerName = 'React update: ' + namedComponent.getName();
      console.time(markerName);
    }
    // 执行updateComponent
    ReactReconciler.performUpdateIfNecessary(
      component,
      transaction.reconcileTransaction,
      updateBatchNumber,
    );
    // 执行dirtyComponent中之前未执行的callback
    if (callbacks) {
      for (var j = 0; j < callbacks.length; j++) {
        transaction.callbackQueue.enqueue(
          callbacks[j],
          component.getPublicInstance(),
        );
      }
    }
  }
}
```
`performUpdateIfNecessary()` 源码路径 [`src/renderers/shared/stack/reconciler/ReactReconciler.js`](https://github.com/facebook/react/blob/v15.6.0/src/renderers/shared/stack/reconciler/ReactReconciler.js#L177)
```
performUpdateIfNecessary: function(
    internalInstance,
    transaction,
    updateBatchNumber,
  ) {
    internalInstance.performUpdateIfNecessary(transaction);
  },
};
```
`performUpdateIfNecessary()` 源码路径 [`src/renderers/shared/stack/reconciler/ReactCompositeComponent.js`](https://github.com/facebook/react/blob/v15.6.0/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L745)

```
  performUpdateIfNecessary: function(transaction) {
    if (this._pendingElement != null) {
      ReactReconciler.receiveComponent(
        this,
        this._pendingElement,
        transaction,
        this._context,
      );
    } else if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
      this.updateComponent(
        transaction,
        this._currentElement,
        this._currentElement,
        this._context,
        this._context,
      );
    } else {
      this._updateBatchNumber = null;
    }
  },
```
`updateComponent()` 源码路径 [`src/renderers/shared/stack/reconciler/ReactCompositeComponent.js`](https://github.com/facebook/react/blob/v15.6.0/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L773)

# 事务概念

简单地说，一个 Transaction 就是将需要执行的 anyMethod 使用 wrapper 封装起来，在通过 Transaction 提供的 perform 方法执行，而在 perform 之前，先执行所有 wrapper 中的 initialize 方法，perform 之后再执行所有 wrapper 中的 close 方法。一组 initialize 及 close 方法称为一个 wrapper，Transaction 支持多个 wrapper 叠加。
![](https://user-gold-cdn.xitu.io/2020/2/26/17080bf6d2991948?w=1488&h=1052&f=png&s=117067)

React 中的 Transaction 提供了一个 Mixin 方便其他模块实现自己需要的事务。要实现自己的事务，需要额外实现一个抽象的 `getTransactionWrappers()` 接口，这个接口是 Transaction 用来获取所有 wrapper 的 initialize 和 close 方法，因此需要返回一个数组对象，每个对象分别有 key 为 initialize 和 close 的方法。
```
var Transaction = require('Transaction');
var emptyFunction = require('emptyFunction');

var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function() {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  },
};

var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
};

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];

function ReactDefaultBatchingStrategyTransaction() {
  this.reinitializeTransaction();
}

Object.assign(ReactDefaultBatchingStrategyTransaction.prototype, Transaction, {
  getTransactionWrappers: function() {
    return TRANSACTION_WRAPPERS;
  },
});
```

# setState 流程

> setState 流程还是很复杂的，设计也很精巧，避免了重复无谓的刷新组件，React大量运用了注入机制，这样每次注入的都是同一个实例化对象，防止多次实例化

1. enqueueSetState 将 state 放入队列中，并调用 enqueueUpdate 处理要更新的 Component
2. 如果组件当前正处于 update 事务中，则先将 Component 存入 dirtyComponent 中。否则调用 batchedUpdates 处理
3. batchedUpdates 发起一次 transaction.perform() 事务
4. 开始执行事务初始化，运行，结束三个阶段
    * 初始化：事务初始化阶段没有注册方法，故无方法要执行
    * 运行：执行 setSate 时传入的 callback 方法，一般不会传 callback 参数
    * 结束：执行 `RESET_BATCHED_UPDATES FLUSH_BATCHED_UPDATES ` 这两个 wrapper 中的 close 方法
5. `FLUSH_BATCHED_UPDATES` 在 close 阶段，flushBatchedUpdates 方法会循环遍历所有的 dirtyComponents ，调用 updateComponent 刷新组件，并执行它的 pendingCallbacks , 也就是 setState 中设置的 callback

> 组件挂载后，setState一般是通过DOM交互事件触发，如 click

1. 点击button按钮
2. ReactEventListener 会触发 dispatchEvent方法
3. dispatchEvent 调用 ReactUpdates.batchedUpdates
4. 进入事务，init 为空， anyMethod 为 `ReactUpdates.batchedUpdates(handleTopLevelImpl, bookKeeping);`
    * handleTopLevelImpl 是在这边调用DOM事件对应的回调方法
    * 然后是`setState()`
        * 将state的变化和对应的回调函数放置到 _pendingStateQueue ，和 _pendingCallback 中
        * 把需要更新的组件放到 dirtyComponents 序列中
    * 执行 perform()
    * 执行 close 渲染更新
```
dispatchEvent: function(topLevelType, nativeEvent) {
    if (!ReactEventListener._enabled) {
      return;
    }

    var bookKeeping = TopLevelCallbackBookKeeping.getPooled(
      topLevelType,
      nativeEvent,
    );
    try {
      // Event queue being processed in the same cycle allows
      // `preventDefault`.
      ReactUpdates.batchedUpdates(handleTopLevelImpl, bookKeeping);
    } finally {
      TopLevelCallbackBookKeeping.release(bookKeeping);
    }
}
```
