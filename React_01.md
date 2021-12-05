# React 17.0.2

## concurrentMode

React 从16 的版本开始，引入了一个实验性的**concurrentMode** ,在concurrent 的模式下，react的渲染过程可以中断，可以显著提升react的渲染性能。但是concurrentMode还是处于实验的阶段，官网也只有一些简单的介绍,（其实16.8后引入了hook,背后也有concurrentMode,的因素）

在concurrent模式下应用的入口改成了**ReactDOM.createRoot(rootNode).render(<App />)** ,

下面是一些介绍文档和视频

[concurrentMode](https://reactjs.org/docs/concurrent-mode-reference.html)

[Flarnie Marchan - Ready for Concurrent Mode? | React Next 2019](https://www.youtube.com/watch?v=V1Ly-8Z1wQA&list=PLV91uluYvkQ6cR01PsYnLnJvX-DiwE8-o)

[Lin Clark - A Cartoon Intro to Fiber - React Conf 2017](https://www.youtube.com/watch?v=V1Ly-8Z1wQA&list=PLV91uluYvkQ6cR01PsYnLnJvX-DiwE8-o)



## 入口

React18之前通常react的应用入口如下：（react有一中concurrent 模式，入口不是这个）

```javascript
ReactDOM.render(<App />, document.getElementById('root'));
```

调用ReactDOM.render 将根节点渲染到对应的dom节点上，我们来看看ReactDOM.render 做了什么

从react-dom包的index.js中看到render从packages/react-dom/src/client/ReactDOMLegacy.js引入

```javascript
export function render(
  element: React$Element<any>,
  container: Container,
  callback: ?Function,
) {
// 省略。。。
  return legacyRenderSubtreeIntoContainer(
    null,
    element,
    container,
    false,
    callback,
  );
}

function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,//注意上面调用的时候parentComponent传的null
  children: ReactNodeList,//即render对应的App
  container: Container,// 即挂载的dom节点
  forceHydrate: boolean,//上面传的是false
  callback: ?Function,
) {
	//。。。
  // 注意这里声明container._reactRootContainer 赋值给root,container是我们传进来渲染的那个根dom节点  
  let root = container._reactRootContainer; 
  let fiberRoot: FiberRoot;// 声明一个fiberRoot，它是react虚拟dom的根节点，上面记录了effects等各种状态。后面会有一个rootFiber，rootFiber只是一个fiber，
  if (!root) {// 初始root为undefined,
    // Initial mount
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(//注意 这里设置了root=container._reactRootContainer
      container,
      forceHydrate,
    );
    fiberRoot = root;//设置fiberRoot为root
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched.
    flushSync(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);//创建完fiberRoot,调用updateContainer触发整个树的更新
    });
  } else {// root存在
    fiberRoot = root;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

我们先看 **legacyCreateRootFromDOMContainer** 逻辑，看它返回了什么

调用legacyCreateRootFromDOMContainer 传了container是对应的dom,forceHydrate应该是false

```javascript
function legacyCreateRootFromDOMContainer(
  container: Container,
  forceHydrate: boolean,//上面传的是false
): FiberRoot {
  // First clear any existing content.清空container下的所有子节点（dom节点）
  if (!forceHydrate) {
    let rootSibling;
    while ((rootSibling = container.lastChild)) {
      container.removeChild(rootSibling);
    }
  }
	//调用createContainer 创建根几点
  const root = createContainer(//记得前面得到root后会调用一个updateContainer
    container,
    LegacyRoot,// 值为0
    forceHydrate,// false
    null, // hydrationCallbacks
    false, // isStrictMode
    false, // concurrentUpdatesByDefaultOverride,
  );
  //打标记  设置container['__reactContainer$+randomKey] = root.current,在container上设置属性
  markContainerAsRoot(root.current, container);
	//这里listenToAllSupportedEvents 是注册所有的事件监听，react的合成事件，我们后面再说
  const rootContainerElement =
    container.nodeType === COMMENT_NODE ? container.parentNode : container;
  listenToAllSupportedEvents(rootContainerElement);

  return root;
}
```

我们看看**createContainer**，它是从**react-reconciler/src/ReactFiberReconciler**引入的，这个文件在react-reconciler包下面，**ReactFiberReconciler**有3个文件，一个".old.js",一个"new.js",我们找的应该在".new.js"内,这个reconciler内就是react的虚拟dom的主要操作逻辑

packages/react-reconciler/src/ReactFiberReconciler.new.js

```javascript
export function createContainer(
  containerInfo: Container,
  tag: RootTag,// 值为LegacyRoot = 0
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
): OpaqueRoot {
  return createFiberRoot(//直接调用createFiberRoot
    containerInfo,
    tag,// 值为LegacyRoot = 0
    hydrate,//false
    hydrationCallbacks, // null
    isStrictMode,// false
    concurrentUpdatesByDefaultOverride,//false
  );
}

export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
): FiberRoot {// 注意这个方法返回了一个FiberRoot
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);//直接new 一个FiberRootNode,有FiberRootNode，FiberNode 2种，下面介绍
  if (enableSuspenseCallback) {// false
    root.hydrationCallbacks = hydrationCallbacks;
  }

  // Cyclic construction. This cheats the type system right now because
  // stateNode is any.
  //主要是构建一个HostRoot类型的fiberNode,因tag的不同，fiberNode的mode 不同
  const uninitializedFiber = createHostRootFiber(
    tag,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
  );
  root.current = uninitializedFiber;// 设置创建的FiberRootNode.current = HostRoot的FiberNode（rootFiber）
  uninitializedFiber.stateNode = root; // 设置FiberNode.stateNode = FiberRootNode， root 和 fiber相互引用

  if (enableCache) {
    const initialCache = createCache();
    retainCache(initialCache);

    // The pooledCache is a fresh cache instance that is used temporarily
    // for newly mounted boundaries during a render. In general, the
    // pooledCache is always cleared from the root at the end of a render:
    // it is either released when render commits, or moved to an Offscreen
    // component if rendering suspends. Because the lifetime of the pooled
    // cache is distinct from the main memoizedState.cache, it must be
    // retained separately.
    root.pooledCache = initialCache;
    retainCache(initialCache);
    const initialState = {
      element: null,
      cache: initialCache,
    };
    uninitializedFiber.memoizedState = initialState;
  } else {
    const initialState = {
      element: null,
    };
    uninitializedFiber.memoizedState = initialState;// 设置fiberRoot的memoizedState
  }

  initializeUpdateQueue(uninitializedFiber);// 初始化fiber的updateQueue

  return root;
}

//createHostRootFiber
export function createHostRootFiber(
  tag: RootTag,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
): Fiber {
  let mode;
  if (tag === ConcurrentRoot) {//传进来的tag是LegacyRoot，这里是concurrent模式下对应的逻辑，暂时先不看
    mode = ConcurrentMode;
    if (isStrictMode === true) {
      mode |= StrictLegacyMode;

      if (enableStrictEffects) {
        mode |= StrictEffectsMode;
      }
    } else if (enableStrictEffects && createRootStrictEffectsByDefault) {
      mode |= StrictLegacyMode | StrictEffectsMode;
    }
    if (
      // We only use this flag for our repo tests to check both behaviors.
      // TODO: Flip this flag and rename it something like "forceConcurrentByDefaultForTesting"
      !enableSyncDefaultUpdates ||
      // Only for internal experiments.
      (allowConcurrentByDefault && concurrentUpdatesByDefaultOverride)
    ) {
      mode |= ConcurrentUpdatesByDefaultMode;
    }
  } else {
    mode = NoMode;// LegacyRoot 下 mode = NoMode
  }

  if (enableProfilerTimer && isDevToolsPresent) {
    // Always collect profile timings when DevTools are present.
    // This enables DevTools to start capturing timing at any point–
    // Without some nodes in the tree having empty base times.
    mode |= ProfileMode;
  }

  return createFiber(HostRoot, null, null, mode);//创建一个Fiber,HostRoot=3,就是用HostRoot 这个tag new FiberNode
}

export function initializeUpdateQueue<State>(fiber: Fiber): void {
  const queue: UpdateQueue<State> = {// queue 是一个链表
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null,
      interleaved: null,
      lanes: NoLanes,
    },
    effects: null,
  };
  fiber.updateQueue = queue;//设置rootFiber的updateQueue
}
```

我们看看FiberRootNode和FiberNode

```javascript
function FiberRootNode(containerInfo, tag, hydrate) {
  this.tag = tag; // tag LegacyRoot|ConcurrentRoot
  this.containerInfo = containerInfo;// render的dom节点
  this.pendingChildren = null;
  this.current = null;//会指向rootFiber
  this.pingCache = null;
  this.finishedWork = null;//
  this.timeoutHandle = noTimeout;
  this.context = null;
  this.pendingContext = null;
  this.isDehydrated = hydrate;
  this.callbackNode = null;
  this.callbackPriority = NoLane;
  this.eventTimes = createLaneMap(NoLanes);
  this.expirationTimes = createLaneMap(NoTimestamp);

  this.pendingLanes = NoLanes; //对应updatelanes react采用lane 模型管理effects的优先级，取代之前的时间片的模型
  this.suspendedLanes = NoLanes;
  this.pingedLanes = NoLanes;
  this.expiredLanes = NoLanes;
  this.mutableReadLanes = NoLanes;
  this.finishedLanes = NoLanes;

  this.entangledLanes = NoLanes;
  this.entanglements = createLaneMap(NoLanes);

  if (enableCache) {
    this.pooledCache = null;
    this.pooledCacheLanes = NoLanes;
  }

  if (supportsHydration) {
    this.mutableSourceEagerHydrationData = null;
  }

  if (enableSuspenseCallback) {
    this.hydrationCallbacks = null;
  }

  if (enableProfilerTimer && enableProfilerCommitHooks) {
    this.effectDuration = 0;
    this.passiveEffectDuration = 0;
  }

  if (enableUpdaterTracking) {
    this.memoizedUpdaters = new Set();
    const pendingUpdatersLaneMap = (this.pendingUpdatersLaneMap = []);
    for (let i = 0; i < TotalLanes; i++) {
      pendingUpdatersLaneMap.push(new Set());
    }
  }

  if (__DEV__) {
    switch (tag) {
      case ConcurrentRoot:
        this._debugRootType = hydrate ? 'hydrateRoot()' : 'createRoot()';
        break;
      case LegacyRoot:
        this._debugRootType = hydrate ? 'hydrate()' : 'render()';
        break;
    }
  }
}


//packages/react-reconciler/src/ReactFiber.new.js
function FiberNode(
  tag: WorkTag,// HostRoot
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,// NoMode
) {
  // Instance
  this.tag = tag; // 哪一种类型的fiber
  this.key = key;// key
  this.elementType = null;
  this.type = null;
  this.stateNode = null;

  // Fiber
  this.return = null;// 指向父节点
  this.child = null; // 指向子节点
  this.sibling = null; // 指向兄弟节点
  this.index = 0;

  this.ref = null;

  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;

  // Effects
  this.flags = NoFlags;
  this.subtreeFlags = NoFlags;
  this.deletions = null;

  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  this.alternate = null;

  if (enableProfilerTimer) {
    // Note: The following is done to avoid a v8 performance cliff.
    //
    // Initializing the fields below to smis and later updating them with
    // double values will cause Fibers to end up having separate shapes.
    // This behavior/bug has something to do with Object.preventExtension().
    // Fortunately this only impacts DEV builds.
    // Unfortunately it makes React unusably slow for some applications.
    // To work around this, initialize the fields below with doubles.
    //
    // Learn more about this here:
    // https://github.com/facebook/react/issues/14365
    // https://bugs.chromium.org/p/v8/issues/detail?id=8538
    this.actualDuration = Number.NaN;
    this.actualStartTime = Number.NaN;
    this.selfBaseDuration = Number.NaN;
    this.treeBaseDuration = Number.NaN;

    // It's okay to replace the initial doubles with smis after initialization.
    // This won't trigger the performance cliff mentioned above,
    // and it simplifies other profiler code (including DevTools).
    this.actualDuration = 0;
    this.actualStartTime = -1;
    this.selfBaseDuration = 0;
    this.treeBaseDuration = 0;
  }

  if (__DEV__) {
    // This isn't directly used but is handy for debugging internals:

    this._debugSource = null;
    this._debugOwner = null;
    this._debugNeedsRemount = false;
    this._debugHookTypes = null;
    if (!hasBadMapPolyfill && typeof Object.preventExtensions === 'function') {
      Object.preventExtensions(this);
    }
  }
}
```

到这里我们知道，root 是 **new FiberRootNode** ，它是整个应用的根。然后又创建了一个**HostRoot**对应的**rootFiber**,并将FiberRoot 和rootFiber进行了关联，**root.current = rootFiber, rootFiber.stateNode = root, container._reactRootContainer=root** ,并初始化了**rootFiber**的**memoizedState**和**updateQueue**

回到**legacyRenderSubtreeIntoContainer**

```javascript
// Initial mount should not be batched.
flushSync(() => {
  updateContainer(children, fiberRoot, parentComponent, callback);//创建完fiberRoot,调用updateContainer触发整个树的更新
});
```
**updateContainer** 同样是在packages/react-reconciler/src/ReactFiberReconciler.new.js

```javascript
export function updateContainer(
  element: ReactNodeList,//render 的App
  container: OpaqueRoot,// render dom节点
  parentComponent: ?React$Component<any, any>,// null
  callback: ?Function,
): Lane {
  if (__DEV__) {
    onScheduleRoot(container, element);
  }
  const current = container.current; // fiberRoot
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(current);// 请求一个updateLane决定更新的优先级 lane 模型

  if (enableSchedulingProfiler) {
    markRenderScheduled(lane);
  }

  const context = getContextForSubtree(parentComponent);//这里 parentComponent 为null,返回的应该是一个空的context
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }
	// 。。。。

  const update = createUpdate(eventTime, lane);// 创建一个update
  // Caution: React DevTools currently depends on this property
  // being called "element".
  update.payload = {element};// 设置payload是 App

  callback = callback === undefined ? null : callback;// 没有传入callback ,可以忽略
  if (callback !== null) {
    if (__DEV__) {
      if (typeof callback !== 'function') {
        console.error(
          'render(...): Expected the last optional `callback` argument to be a ' +
            'function. Instead received: %s.',
          callback,
        );
      }
    }
    update.callback = callback;
  }

  enqueueUpdate(current, update, lane); // 把这个update 放入fiberRoot 的updateQueue 并设置lane （优先级）
  const root = scheduleUpdateOnFiber(current, lane, eventTime); // 调用scheduleUpdateOnFiber 触发更新
  if (root !== null) {
    entangleTransitions(root, current, lane);
  }

  return lane;
}
```

updateContainer主要是构造一个**update**,放入fiberRoot的**updateQueue**,然后调用**scheduleUpdateOnFiber**，从根节点开始



