# performSyncWorkOnRoot

packages/react-reconciler/src/ReactFiberWorkLoop.new.js

```javascript
// This is the entry point for synchronous tasks that don't go
// through Scheduler
//同步任务的入口，Legacy 模式下默认的入口
function performSyncWorkOnRoot(root) {
  if (enableProfilerTimer && enableProfilerNestedUpdatePhase) {
    syncNestedUpdateFlag();
  }

  if ((executionContext & (RenderContext | CommitContext)) !== NoContext) {
    throw new Error('Should not already be working.');
  }

  flushPassiveEffects();

  let lanes = getNextLanes(root, NoLanes);
  if (!includesSomeLane(lanes, SyncLane)) {//判断是否有SyncLane的任务，syncLane的优先级最高
    // There's no remaining sync work left.
    ensureRootIsScheduled(root, now());
    return null;
  }

  let exitStatus = renderRootSync(root, lanes);// 调用renderRootSync
  //错误的场景
  if (root.tag !== LegacyRoot && exitStatus === RootErrored) { 
    // If something threw an error, try rendering one more time. We'll render
    // synchronously to block concurrent data mutations, and we'll includes
    // all pending updates are included. If it still fails after the second
    // attempt, we'll give up and commit the resulting tree.
    const errorRetryLanes = getLanesToRetrySynchronouslyOnError(root);
    if (errorRetryLanes !== NoLanes) {
      lanes = errorRetryLanes;
      exitStatus = recoverFromConcurrentError(root, errorRetryLanes);
    }
  }

  if (exitStatus === RootFatalErrored) {
    const fatalError = workInProgressRootFatalError;
    prepareFreshStack(root, NoLanes);
    markRootSuspended(root, lanes);
    ensureRootIsScheduled(root, now());
    throw fatalError;
  }

  // We now have a consistent tree. Because this is a sync render, we
  // will commit it even if something suspended.
  const finishedWork: Fiber = (root.current.alternate: any);
  root.finishedWork = finishedWork;
  root.finishedLanes = lanes;
  commitRoot(root); // commit进入commit 阶段

  // Before exiting, make sure there's a callback scheduled for the next
  // pending level.
  ensureRootIsScheduled(root, now());// 最后在确保一下

  return null;
}
```

来看看 **renderRootSync** 

React  更新分为2个阶段 **Reconcile** 和 **Commit**

```javascript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;// 标记进入了render阶段
  const prevDispatcher = pushDispatcher();

  // If the root or lanes have changed, throw out the existing stack
  // and prepare a fresh one. Otherwise we'll continue where we left off.
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) { // 初始 (workInProgressRoot = null) != root
    if (enableUpdaterTracking) {
      if (isDevToolsPresent) {
        const memoizedUpdaters = root.memoizedUpdaters;
        if (memoizedUpdaters.size > 0) {
          restorePendingUpdaters(root, workInProgressRootRenderLanes);
          memoizedUpdaters.clear();
        }

        // At this point, move Fibers that scheduled the upcoming work from the Map to the Set.
        // If we bailout on this work, we'll move them back (like above).
        // It's important to move them now in case the work spawns more work at the same priority with different updaters.
        // That way we can keep the current update and future updates separate.
        movePendingFibersToMemoized(root, lanes);
      }
    }
		
    // 在prepareFreshStack 内部会初始化workInProgressRoot
    prepareFreshStack(root, lanes);
  }

  if (__DEV__) {
    if (enableDebugTracing) {
      logRenderStarted(lanes);
    }
  }

  if (enableSchedulingProfiler) {
    markRenderStarted(lanes);
  }

  do {
    try {
      workLoopSync(); // 执行workLoopSync开始reconciler
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);//这里虽然是个死循环，但是上面workLoopSync()执行完后会有个break;只要workLoopSync()执行完，循环就会退出
  resetContextDependencies();

  executionContext = prevExecutionContext;//恢复executionContext为之前的状态
  popDispatcher(prevDispatcher);

  if (workInProgress !== null) {
    // This is a sync render, so we should have finished the whole tree.
    throw new Error(
      'Cannot commit an incomplete root. This error is likely caused by a ' +
        'bug in React. Please file an issue.',
    );
  }

  if (__DEV__) {
    if (enableDebugTracing) {
      logRenderStopped();
    }
  }

  if (enableSchedulingProfiler) {
    markRenderStopped();
  }

  // Set this to null to indicate there's no in-progress render.
  //重置状态为初始值
  workInProgressRoot = null; // 重新把 workInProgressRoot 设置为null，所以上面有个判断workInProgressRoot == null,在每次更新都会成立
  workInProgressRootRenderLanes = NoLanes;

  return workInProgressRootExitStatus;
}
```

prepareFreshStack 内部就是把workInProgressRoot设置成root,workInProgress设置为root.current （RootFiber）,初始化lanes和状态

初始化队列

```javascript
function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  const timeoutHandle = root.timeoutHandle;
  if (timeoutHandle !== noTimeout) {
    // The root previous suspended and scheduled a timeout to commit a fallback
    // state. Now that we have additional work, cancel the timeout.
    root.timeoutHandle = noTimeout;
    // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
    cancelTimeout(timeoutHandle);
  }

  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork, workInProgressRootRenderLanes);
      interruptedWork = interruptedWork.return;
    }
  }
 	//主要是初始化如下一些变量
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  workInProgressRootRenderLanes = subtreeRenderLanes = workInProgressRootIncludedLanes = lanes;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootInterleavedUpdatedLanes = NoLanes;
  workInProgressRootRenderPhaseUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;

  enqueueInterleavedUpdates();

  if (__DEV__) {
    ReactStrictModeWarnings.discardPendingWarnings();
  }
}
```

通过上面的代码，我们知道performSyncWorkOnRoot的主要逻辑是初始化workInProgressRoot 等一些关键变量，然后执行  workLoopSync，

```javascript
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {//只有在结束时才会重置为null
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork: Fiber): void {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;// react render 过程中会有2棵树，一个是当前的dom结构树，current,一棵是渲染过程中构建的树，workingProgressTree,alternate 属性指向 current Tree
  setCurrentDebugFiberInDEV(unitOfWork);

  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {//enableProfilerTimer 且 unitOfWork.mode 是 ProfileMode
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes); // beginWork 将结果赋值给next,后面我们会知道这是一个深度优先的递归，会从根节点一路向下找子节点。当子节点为null,即没有子节点时，会把兄弟节点当做下一个工作的节点，并在当前执行completeWork,当一个父节点下所有的子节点都经过beginWork,并且不存在兄弟节点时，会在父节点上执行completeWork,把状态merge到父节点上
  }

  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {/
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;// 把next设置成workInProgress 继续递归
  }

  ReactCurrentOwner.current = null;
}
```



