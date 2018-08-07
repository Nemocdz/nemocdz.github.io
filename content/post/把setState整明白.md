---
title: "把setState整明白"
date: 2018-08-06T11:17:00+08:00
draft: false
---

加入新团队后，团队项目使用了React Native。刚开始接触React Native，除了学习React Native的使用，更要了解React.js这个框架，才能更好的使用。而React框架中，笔者一开始就感觉奇妙的，就是这个看似同步，表现却不一定是同步的setState方法。看了网上一些文章，结论似乎都是从某几篇博客相互借鉴的结论，但里面笔者还是觉得有一些不太明白的地方，幸亏[React.js源码](https://github.com/facebook/react)是开源的。顺着源码看下去，一些困惑的问题终于有些眉目。

<!--more-->

开始之前，读者可思考以下几个问题：

* setState是异步还是同步的
* 在setTimeout方法中调用setState的值为何马上就能更新
* setState中传入一个Function为何值马上就能更新
* setState为何要如此设计
* setState的最佳实践是什么

以下源码基于我们团队在用的React 16.0.0版本，目前最新的React 16.4.0版本的类名和文件结构均有很大变化，但设计思想应该还是差不多的，可供参考。

### setState的入口

setState的最上层自然在ReactBaseClasses中。

```javascript
//ReactBaseClasses.js
ReactComponent.prototype.setState = function(partialState, callback) {
  // ...
  //调用内部updater
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```

而这个updater，在初始化时已经告诉我们，是实际使用时注入的。

```javascript
//ReactBaseClasses.js
function ReactComponent(props, context, updater) {
  // ...
  // 真正的updater在renderer注入
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}
```

找到注入的地方，目的是找到updater是个什么类型。

```javascript
//ReactCompositeComponet.js
mountComponent: function(
    transaction,
    hostParent,
    hostContainerInfo,
    context,
  ) {
        // ...
        // 由Transaction获取，这个是ReactReconcileTransaction
        var updateQueue = transaction.getUpdateQueue();
        // ...
        inst.updater = updateQueue;
        // ...
    }
```

```javascript
//ReactReconcileTransaction.js
getUpdateQueue: function() {
    return ReactUpdateQueue;
},
```

终于看到了具体enqueSetState方法的内容。

```javascript
//ReactUpdateQueue.js
enqueueSetState: function(
    publicInstance,
    partialState,
    callback,
    callerName,
  ) {
    // ...
    // 外部示例转化为内部实例
    var internalInstance = getInternalInstanceReadyForUpdate(publicInstance);
    // ...
    // 将需要更新的state放入等待队列中
    var queue =
      internalInstance._pendingStateQueue ||
      (internalInstance._pendingStateQueue = []);
    queue.push(partialState);
    // ...
    // callback也一样放入等待队列中
    if (callback !== null) {
      // ...
      if (internalInstance._pendingCallbacks) {
        internalInstance._pendingCallbacks.push(callback);
      } else {
        internalInstance._pendingCallbacks = [callback];
      }
    }

    enqueueUpdate(internalInstance);
  },
      
function enqueueUpdate(internalInstance) {
  ReactUpdates.enqueueUpdate(internalInstance);
}
```

而更新操作由ReactUpdates这个类负责。

```javascript
//ReactUpdates.js
function enqueueUpdate(component) {
  //...

  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }

  dirtyComponents.push(component);
  //...
}
```

而这个isBatchingUpdates的判断，就是代表是否在批量更新中。如果正在更新中，则整个组件放入dirtyComponents数组中，后面会讲到。这里这个batchingStrategy，其实就是ReactDefaultBatchingStrategy（外部注入的）。

```javascript
//ReactDOMStackInjection.js
ReactUpdates.injection.injectBatchingStrategy(ReactDefaultBatchingStrategy);
```

而这个类里的，则会让挂起更新状态，并调用transaction的perform。

```javascript
//ReactDefaultBatchingStrategy.js
var ReactDefaultBatchingStrategy = {
  isBatchingUpdates: false,
  batchedUpdates: function(callback, a, b, c, d, e) {
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;

    ReactDefaultBatchingStrategy.isBatchingUpdates = true;

    // 正常情况不会走入if中
    if (alreadyBatchingUpdates) {
      return callback(a, b, c, d, e);
    } else {
      return transaction.perform(callback, null, a, b, c, d, e);
    }
  },
};
```

### Transaction

这里简单解释下事务（Transaction）的概念，先看源码中对事务的一张解释图。

```javascript
 //Transaction.js
 * <pre>
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
```

简单来说，事务相当于对某个方法（anyMethod）执行前和执行后的多组钩子的集合。

可以方便的在某个方法前后分别做一些事情，而且可以分wrapper定义，一个Wrapper一对钩子。

具体来说，可以在Wrapper里定义initialize和close方法，initialize会在anyMethod执行前执行，close会在执行后执行。

### 更新策略里的Transaction

回到刚刚的batchedUpdates方法，里面那个transaction其实执行前都是空方法，而callback是外界传入的enqueueUpdate方法本身，也就是说，执行时会被isBatchingUpdates卡住进入加入dirtyCompoments中。之后就会执行close方法里面去改变isBatchingUpdates的值和执行flushBatchedUpdates方法。

```javascript
//ReactDefaultBatchingStrategy.js

// 更新状态isBatchingUpdates的wrapper
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function() {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  },
};

// 真正更新状态的wrapper
var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
};

// 两个wrapper
var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];

// 添加transaction的wrappers
Object.assign(ReactDefaultBatchingStrategyTransaction.prototype, Transaction, {
  getTransactionWrappers: function() {
    return TRANSACTION_WRAPPERS;
  },
});

var transaction = new ReactDefaultBatchingStrategyTransaction();
```

而这个flushBatchedUpdates方法，按照dirtyComonents里的数量，每次执行了一个transaction。

```javascript
//ReactUpdates.js
var flushBatchedUpdates = function() {
  while (dirtyComponents.length) {
    var transaction = ReactUpdatesFlushTransaction.getPooled();
    transaction.perform(runBatchedUpdates, null, transaction);
    ReactUpdatesFlushTransaction.release(transaction);
  }
};
```

而这个transaction的执行前后前后的钩子如下。

```javascript
//ReactUpdtes.js
// 开始时同步dirComponents数量，结束时通过检查是否在执行中间runBatchedUpdates方法时还有新加入的component，有的话就重新执行一遍
var NESTED_UPDATES = {
  initialize: function() {
    this.dirtyComponentsLength = dirtyComponents.length;
  },
  close: function() {
    if (this.dirtyComponentsLength !== dirtyComponents.length) {
      dirtyComponents.splice(0, this.dirtyComponentsLength);
      flushBatchedUpdates();
    } else {
      dirtyComponents.length = 0;
    }
  },
};

var TRANSACTION_WRAPPERS = [NESTED_UPDATES];


//添加wrapper
Object.assign(ReactUpdatesFlushTransaction.prototype, Transaction, {
  getTransactionWrappers: function() {
    return TRANSACTION_WRAPPERS;
  },
});
```

所以真正更新方法应该在runBatchedUpdates中。

```javascript
//ReactUpdates.js
function runBatchedUpdates(transaction) {
  
  // 排序，保证父组件比子组件先更新
  dirtyComponents.sort(mountOrderComparator);

  // ...
  for (var i = 0; i < len; i++) {
    
    var component = dirtyComponents[i];
    
    // 这里开始进入更新组件的方法
    ReactReconciler.performUpdateIfNecessary(
      component,
      transaction.reconcileTransaction,
      updateBatchNumber,
    );
  }
}
```

而ReactReconciler中的performUpateIfNecessary方法只是一个壳。

```javascript
performUpdateIfNecessary: function(
    internalInstance,
    transaction,
    updateBatchNumber,
  ) {
    // ...
    internalInstance.performUpdateIfNecessary(transaction);
    // ...
 },
```

而真正的方法在ReactCompositeComponent中，如果等待队列中有该更新的state，那么就调用updateComponent。

```javascript
//ReactCompositeComponent.js
performUpdateIfNecessary: function(transaction) {
    if (this._pendingElement != null) {
    // ...
    } else if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
      this.updateComponent(
        transaction,
        this._currentElement,
        this._currentElement,
        this._context,
        this._context,
      );
    } else {
	// ...
    }
},
```

这个方法判断了做了一些判断，而我们也看到了nextState的值才是最后被更新给state的值。

```javascript
//ReactCompositeComponent.js
updateComponent: function(
    transaction,
    prevParentElement,
    nextParentElement,
    prevUnmaskedContext,
    nextUnmaskedContext,
) {
    // 这个将排序队列里的state合并到nextState    
   	var nextState = this._processPendingState(nextProps, nextContext);
    var shouldUpdate = true;
        
    if (shouldUpdate) {
    	this._pendingForceUpdate = false;
      	// Will set `this.props`, `this.state` and `this.context`.
        // 这里才正式更新state
      	this._performComponentUpdate(
        	nextParentElement,
        	nextProps,
        	nextState,
        	nextContext,
        	transaction,
        	nextUnmaskedContext,
      	);
    } else {
      // 这里才正式更新state
      //...
      inst.state = nextState;
      //...
    }
}

_performComponentUpdate: function(
    nextElement,
    nextProps,
    nextState,
    nextContext,
    transaction,
    unmaskedContext,
  ) {
    var inst = this._instance;

    var hasComponentDidUpdate = !!inst.componentDidUpdate;
    var prevProps;
    var prevState;
    if (hasComponentDidUpdate) {
      prevProps = inst.props;
      prevState = inst.state;
    }

    // ...

    this._currentElement = nextElement;
    this._context = unmaskedContext;
    inst.props = nextProps;
    inst.state = nextState;
    inst.context = nextContext;

    // ...
},
```

这个方法也解释了为什么传入函数的state会更新。

```javascript
_processPendingState: function(props, context) {
    var inst = this._instance;
    var queue = this._pendingStateQueue;
    
    //更新了就可以置空了
    this._pendingStateQueue = null;

    
    var nextState = replace ? queue[0] : inst.state;
    var dontMutate = true;
    for (var i = replace ? 1 : 0; i < queue.length; i++) {
      //如果setState传入是函数，那么接收的state是上轮更新过的state
      var partial = queue[i];
      let partialState = typeof partial === 'function'
        ? partial.call(inst, nextState, props, context)
        : partial;
      if (partialState) {
        if (dontMutate) {
          dontMutate = false;
          nextState = Object.assign({}, nextState, partialState);
        } else {
          Object.assign(nextState, partialState);
        }
      }
    }
    return nextState;
},
```

### 好像setState是同步的耶

而如果按照这个流程看完，setState应该是同步的呀？是哪里出了问题呢。

别急，还记得更新策略里面那个Transaction么。那里中间调用的callback是外层传入的，也就说有可能还有其它调用了batchedUpdates呢。那么也就是说，中间的callback，并不止setState会引起。在代码里搜索后发现，果真还有几处调用了batchedUpdates方法。

比如ReactMount的这两个方法

```javascript
//ReactMount.js
_renderNewRootComponent: function(
  nextElement,
  container,
  shouldReuseMarkup,
  context,
  callback,
) {
    
    // ...
    // The initial render is synchronous but any updates that happen during
    // rendering, in componentWillMount or componentDidMount, will be batched
    // according to the current batching strategy.

    ReactUpdates.batchedUpdates(
      batchedMountComponentIntoNode,
      componentInstance,
      container,
      shouldReuseMarkup,
      context,
    );
     
    // ...
},
      
unmountComponentAtNode: function(container) {
    // ...   
    ReactUpdates.batchedUpdates(
      unmountComponentFromNode,
      prevComponent,
      container,
    );
    return true;
    // ...
},      
```

比如ReactDOMEventListener

```javascript
//ReactDOMEventListener.js
dispatchEvent: function(topLevelType, nativeEvent) {   
    // ...
    try {
      // Event queue being processed in the same cycle allows
      // `preventDefault`.
      ReactGenericBatching.batchedUpdates(handleTopLevelImpl, bookKeeping);
    } finally {
     // ...
    }
},
      
//ReactDOMStackInjection.js     
ReactGenericBatching.injection.injectStackBatchedUpdates(
  ReactUpdates.batchedUpdates,
);
```

所以比如在componentDidMount中直接调用时，ReactMount.js 中的**_renderNewRootComponent** 方法已经调用了，也就是说，整个将 React 组件渲染到 DOM 中的过程就处于一个大的 Transaction 中，而其中的callback没有马上被执行，那么自然state没有被马上更新。

### setState为什么这么设计

在react中，state代表UI的状态，也就是UI由state改变而改变，也就是UI=function（state）。笔者觉得，这体现了一种响应式的思想，而响应式与命令式的不同，在于命令式着重看如何命令的过程，而响应式看中数据变化如何输出。而React中对Rerender做出的努力，对渲染的优化，响应式的setState设计，其实也是其中搭配而不可少的一环。

### 最后

最前面的问题，相信每个人都有自己的答案，我这里给出我自己的理解。

Q：setState是异步还是同步的？

A：同步的，但有时候是异步的表现。

Q：在setTimeout方法中调用setState的值为何马上就能更新？

A：因为本身就是同步的，也没有别的因素阻塞。

Q：setState中传入一个Function为何值马上就能更新？

A：源码中的策略。

Q：setState为何要如此设计？

A：为了以响应式的方式改变UI。

Q：setState的最佳实践是什么？

A：以响应式的思路使用。

#### 参考链接

[React 源码剖析系列 － 解密 setState](https://zhuanlan.zhihu.com/p/20328570)

[setState：这个API设计到底怎么样](https://zhuanlan.zhihu.com/p/25954470)

[setState为什么不会同步更新组件状态](https://zhuanlan.zhihu.com/p/25990883)

