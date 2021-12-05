从上面的分析我们知道，reactDom.render 内部经历2个步骤

1、createContainer ，创建一个root ( FiberRoot ),一个 rootFiber ，并设置root.current = rootFiber ,rootFiber.stateNode = root

​	2者相互保存对方的引用

2、updateContainer ,创建完fiberRoot 和rootFiber后，执行updateContainer 。 new Update(),设置payload 为App. 放入rootFiber的updateQueue , 然后调用**scheduleUpdateOnFiber**，触发整个应用的更新

现在我们看看**scheduleUpdateOnFiber** 里做了什么

**scheduleUpdateOnFiber** 位于packages/react-reconciler/src/ReactFiberWorkLoop.new.js

```javascript
export function scheduleUpdateOnFiber(
  fiber: Fiber, // fiber这里是rootFiber
  lane: Lane,// 通过requestUpdatelane 方法可知，当前非ConcurrentMode ,lane = SyncLane ,即同步模式
  eventTime: number,
): FiberRoot | null {
  // 先检查是否有嵌套的更新，防止有repeatedly calls setState inside componentWillUpdate or ' +
  //'componentDidUpdate
  checkForNestedUpdates();
	//标记fiber更新，设置updateLane,这个方法会从当前的fiber节点，一路向上查找，直到根节点，把updateLane merge 到parent.childLanes 中, 然后返回根节点
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    return null;
  }

  // Mark that the root has a pending update.
  // 内部把  root.pendingLanes |= updateLane; 
  markRootUpdated(root, lane, eventTime);

  if (
    (executionContext & RenderContext) !== NoLanes && // 二进制按位与 NoLanes 是0，则executionContext = RenderContext 这里条件才成立
    root === workInProgressRoot// 初次的时候 workInProgressRoot = null
  ) {//判断当前update是不是在render阶段触发。
    // This update was dispatched during the render phase. This is a mistake
    // if the update originates from user space (with the exception of local
    // hook updates, which are handled differently and don't reach this
    // function), but there are some internal React features that use this as
    // an implementation detail, like selective hydration.
    warnAboutRenderPhaseUpdatesInDEV(fiber);

    // Track lanes that were updated during the render phase
    workInProgressRootRenderPhaseUpdatedLanes = mergeLanes(// mergeLanes 把两个参数按位与
      workInProgressRootRenderPhaseUpdatedLanes,
      lane,
    );
  } else {
    //普通的update
    // This is a normal update, scheduled from outside the render phase. For
    // example, during an input event.
    if (enableUpdaterTracking) {
      if (isDevToolsPresent) {
        addFiberToLanesMap(root, fiber, lane);
      }
    }

    warnIfUpdatesNotWrappedWithActDEV(fiber);

    if (enableProfilerTimer && enableProfilerNestedUpdateScheduledHook) {//enableProfilerNestedUpdateScheduledHook 貌似是false ....,那这里不会进入？？？
      if (
        (executionContext & CommitContext) !== NoContext &&
        root === rootCommittingMutationOrLayoutEffects
      ) {
        if (fiber.mode & ProfileMode) {
          let current = fiber;
          while (current !== null) {
            if (current.tag === Profiler) {
              const {id, onNestedUpdateScheduled} = current.memoizedProps;
              if (typeof onNestedUpdateScheduled === 'function') {
                onNestedUpdateScheduled(id);
              }
            }
            current = current.return;
          }
        }
      }
    }

    // TODO: Consolidate with `isInterleavedUpdate` check
    if (root === workInProgressRoot) { // 初次渲染的时候workInProgressRoot = null,应该不会进入这里，workInProgressRoot 应该是在prepareFreshStack中设置值的
      // rendering 的中途收到的更新，interleaved(交错的)
    	// Received an update to a tree that's in the middle of rendering. Mark
      // that there was an interleaved update work on this root. Unless the
      // `deferRenderPhaseUpdateToNextBatch` flag is off and this is a render
      // phase update. In that case, we don't treat render phase updates as if
      // they were interleaved, for backwards compat reasons.
      if (
        deferRenderPhaseUpdateToNextBatch ||
        (executionContext & RenderContext) === NoContext
      ) {
        workInProgressRootInterleavedUpdatedLanes = mergeLanes(
          workInProgressRootInterleavedUpdatedLanes,
          lane,
        );
      }
      if (workInProgressRootExitStatus === RootSuspendedWithDelay) {
        // The root already suspended with a delay, which means this render
        // definitely won't finish. Since we have a new update, let's mark it as
        // suspended now, right before marking the incoming update. This has the
        // effect of interrupting the current render and switching to the update.
        // TODO: Make sure this doesn't override pings that happen while we've
        // already started rendering.
        markRootSuspended(root, workInProgressRootRenderLanes);
      }
    }

    ensureRootIsScheduled(root, eventTime);//初次渲染 这里调用ensureRootIsScheduled！！！
    if (
      lane === SyncLane &&
      executionContext === NoContext &&
      (fiber.mode & ConcurrentMode) === NoMode &&
      // Treat `act` as if it's inside `batchedUpdates`, even in legacy mode.
      !(__DEV__ && ReactCurrentActQueue.isBatchingLegacy)
    ) {
      // Flush the synchronous work now, unless we're already working or inside
      // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
      // scheduleCallbackForFiber to preserve the ability to schedule a callback
      // without immediately flushing it. We only do this for user-initiated
      // updates, to preserve historical behavior of legacy mode.
      resetRenderTimer();
      flushSyncCallbacksOnlyInLegacyMode();
    }
  }
  return root;
}
```

看看markUpdateLaneFromFiberToRoot,主要的逻辑是从当前的节点开始一直向上查找，直到fiberRoot ,把这个update的lane,merege 到**childLanes** 到了最顶层后，返回FiberRoot 。这样每一层的childLanes都会有这个lane

```javascript
// This is split into a separate function so we can mark a fiber with pending
// work without treating it as a typical update that originates from an event;
// e.g. retrying a Suspense boundary isn't an update, but it does schedule work
// on a fiber.
function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,// rootFiber
  lane: Lane,
): FiberRoot | null {
  // Update the source fiber's lanes
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;// alternate react更新的时候会构建2棵树，一个是current,一个是workingProgress ,workingProgress上的节点 alternate 属性指向current tree 对应的节点
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  if (__DEV__) {
    if (
      alternate === null &&
      (sourceFiber.flags & (Placement | Hydrating)) !== NoFlags
    ) {
      warnAboutUpdateOnNotYetMountedFiberInDEV(sourceFiber);
    }
  }
  // Walk the parent path to the root and update the child lanes.
  // 一直向上查找 然后把lane merge到parent.childLanes上 ，到了顶层root的时候root上的时候，所有children上都会有这个lane
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    } else {
      if (__DEV__) {
        if ((parent.flags & (Placement | Hydrating)) !== NoFlags) {
          warnAboutUpdateOnNotYetMountedFiberInDEV(sourceFiber);
        }
      }
    }
    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {// 到了rootFiber
    const root: FiberRoot = node.stateNode;// 返回FiberRoot
    return root;
  } else {
    return null;
  }
}
```

我们再看看**ensureRootIsScheduled**,可以看到ensureRootIsScheduled是root上调度的入口

```javascript
// Use this function to schedule a task for a root. There's only one task per
// root; if a task was already scheduled, we'll check to make sure the priority
// of the existing task is the same as the priority of the next level that the
// root has work on. This function is called on every update, and right before
// exiting a task.
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;

  // Check if any lanes are being starved by other work. If so, mark them as
  // expired so we know to work on those next.
  //检查是否存在被饿死的update ,把它们标记成expired，提高优先级 这样会在下一次的更新中处理它们
  markStarvedLanesAsExpired(root, currentTime);

  // Determine the next lanes to work on, and their priority.
  // 取出root.pendingLanes 中优先级最高的lane
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );

  if (nextLanes === NoLanes) {
    // Special case: There's nothing to work on.
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
  }

  // We use the highest priority lane to represent the priority of the callback.
  const newCallbackPriority = getHighestPriorityLane(nextLanes);

  // Check if there's an existing task. We may be able to reuse it.
  const existingCallbackPriority = root.callbackPriority;
  if (
    existingCallbackPriority === newCallbackPriority &&
    // Special case related to `act`. If the currently scheduled task is a
    // Scheduler task, rather than an `act` task, cancel it and re-scheduled
    // on the `act` queue.
    !(
      __DEV__ &&
      ReactCurrentActQueue.current !== null &&
      existingCallbackNode !== fakeActCallbackNode
    )
  ) {
    if (__DEV__) {
      // If we're going to re-use an existing task, it needs to exist.
      // Assume that discrete update microtasks are non-cancellable and null.
      // TODO: Temporary until we confirm this warning is not fired.
      if (
        existingCallbackNode == null &&
        existingCallbackPriority !== SyncLane
      ) {
        console.error(
          'Expected scheduled callback to exist. This error is likely caused by a bug in React. Please file an issue.',
        );
      }
    }
    // The priority hasn't changed. We can reuse the existing task. Exit.
    return;
  }

  if (existingCallbackNode != null) {
    // Cancel the existing callback. We'll schedule a new one below.
    cancelCallback(existingCallbackNode);
  }

  // Schedule a new callback.
  let newCallbackNode;
  if (newCallbackPriority === SyncLane) {// 看代码应该会走到这里。。。。
    // Special case: Sync React callbacks are scheduled on a special
    // internal queue
    if (root.tag === LegacyRoot) {
      if (__DEV__ && ReactCurrentActQueue.isBatchingLegacy !== null) {
        ReactCurrentActQueue.didScheduleLegacyUpdate = true;
      }
      scheduleLegacySyncCallback(performSyncWorkOnRoot.bind(null, root));// 把 performSyncWorkOnRoot 放入一个调度的队列
    } else {
      scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));// 同样也是performSyncWorkOnRoot
    }
    if (supportsMicrotasks) {//是否支持微任务
      // Flush the queue in a microtask.
      if (__DEV__ && ReactCurrentActQueue.current !== null) {
        // Inside `act`, use our internal `act` queue so that these get flushed
        // at the end of the current scope even when using the sync version
        // of `act`.
        ReactCurrentActQueue.current.push(flushSyncCallbacks);
      } else {
        scheduleMicrotask(flushSyncCallbacks);//把 flushSyncCallbacks 放入队列
      }
    } else {
      // Flush the queue in an Immediate task.
      scheduleCallback(ImmediateSchedulerPriority, flushSyncCallbacks); // 这里会调用Scheduler_scheduleCallback react的内部调度器， 作用同样是把 flushSyncCallbacks 放入一个队列等待执行
    }
    newCallbackNode = null;
  } else { // 这边应该对应 concurrentMode 的逻辑
    let schedulerPriorityLevel;
    switch (lanesToEventPriority(nextLanes)) {
      case DiscreteEventPriority:
        schedulerPriorityLevel = ImmediateSchedulerPriority;
        break;
      case ContinuousEventPriority:
        schedulerPriorityLevel = UserBlockingSchedulerPriority;
        break;
      case DefaultEventPriority:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
      case IdleEventPriority:
        schedulerPriorityLevel = IdleSchedulerPriority;
        break;
      default:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
    }
    newCallbackNode = scheduleCallback(// Scheduler_scheduleCallback 
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root), //performConcurrentWorkOnRoot concurrentMode
    );
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}

//flushSyncCallbacks 逻辑是把前面存入syncQueue 中的 callback 即 performSyncWorkOnRoot 取出来，一个一个执行，同步模式下的更新逻辑都在performSyncWorkOnRoot中
export function flushSyncCallbacks() {
  if (!isFlushingSyncQueue && syncQueue !== null) {
    // Prevent re-entrance.
    isFlushingSyncQueue = true;
    let i = 0;
    const previousUpdatePriority = getCurrentUpdatePriority();
    try {
      const isSync = true;
      const queue = syncQueue;
      // TODO: Is this necessary anymore? The only user code that runs in this
      // queue is in the render or commit phases.
      setCurrentUpdatePriority(DiscreteEventPriority);
      for (; i < queue.length; i++) {
        let callback = queue[i];
        do {
          callback = callback(isSync);
        } while (callback !== null);
      }
      syncQueue = null;
      includesLegacySyncCallbacks = false;
    } catch (error) {
      // If something throws, leave the remaining callbacks on the queue.
      if (syncQueue !== null) {
        syncQueue = syncQueue.slice(i + 1);
      }
      // Resume flushing in the next tick
      scheduleCallback(ImmediatePriority, flushSyncCallbacks);
      throw error;
    } finally {
      setCurrentUpdatePriority(previousUpdatePriority);
      isFlushingSyncQueue = false;
    }
  }
  return null;
}
```

**ensureRootIsScheduled**会调用**performSyncWorkOnRoot**或者**performConcurrentWorkOnRoot** (concurrent模式下)

貌似现在默认会走**performConcurrentWorkOnRoot** ？？？

现在我们知道，在scheduleUpdateOnFiber - > **ensureRootIsScheduled** - > performSyncWorkOnRoot

