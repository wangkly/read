# vue3 watch computed

## watch

vue3提供了watch 、watchEffect 2种功能，内部都是调用的doWatch，区别是watch 有callback , watchEffect 没有

packages/runtime-core/src/apiWatch.ts

```typescript
// Simple effect.
export function watchEffect(
  effect: WatchEffect,
  options?: WatchOptionsBase
): WatchStopHandle {
  return doWatch(effect, null, options)
}

export type WatchEffect = (onInvalidate: InvalidateCbRegistrator) => void
export interface WatchOptionsBase {
  flush?: 'pre' | 'post' | 'sync'
  onTrack?: ReactiveEffectOptions['onTrack']
  onTrigger?: ReactiveEffectOptions['onTrigger']
}

export interface WatchOptions<Immediate = boolean> extends WatchOptionsBase {
  immediate?: Immediate
  deep?: boolean
}
watchEffect 有2个参数：effect是一个function 接受一个onInvalidate 参数，用于清除副作用
第二个参数options WatchOptionsBase flush 标志副作用的刷新时机sync 强制同步触发
最后调用doWatch,第二个参数为null

watch 有几个重载，看下具体实现
// implementation
export function watch<T = any, Immediate extends Readonly<boolean> = false>(
  source: T | WatchSource<T>,
  cb: any,
  options?: WatchOptions<Immediate>
): WatchStopHandle {
  return doWatch(source as any, cb, options)
}

export type WatchCallback<V = any, OV = any> = (
  value: V,
  oldValue: OV,
  onInvalidate: InvalidateCbRegistrator
) => any

其中cb是传进来的callback,参数有newValue ,oldValue,最终调用的也是doWatch


```

​	我们看下doWatch的具体实现

```typescript
function doWatch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb: WatchCallback | null,
  { immediate, deep, flush, onTrack, onTrigger }: WatchOptions = EMPTY_OBJ
): WatchStopHandle {
  const instance = currentInstance
  let getter: () => any
  let forceTrigger = false
  let isMultiSource = false
	// 判断watch的source类型，根据类型定义不同的getter，
  if (isRef(source)) {
    getter = () => source.value // ref类型取value
    forceTrigger = !!source._shallow
  } else if (isReactive(source)) {
    getter = () => source // reactive 直接取值
    deep = true
  } else if (isArray(source)) {// 数组的话遍历，然后判断每一个成员的类型，同上
    isMultiSource = true
    forceTrigger = source.some(isReactive)
    getter = () =>
      source.map(s => {
        if (isRef(s)) {
          return s.value
        } else if (isReactive(s)) {
          return traverse(s)
        } else if (isFunction(s)) {
          return callWithErrorHandling(s, instance, ErrorCodes.WATCH_GETTER)
        } else {
          __DEV__ && warnInvalidSource(s)
        }
      })
  } else if (isFunction(source)) {// 如果是function,有可能是watch,有可能是watchEffect
    if (cb) {
      // getter with cb
      getter = () =>
        callWithErrorHandling(source, instance, ErrorCodes.WATCH_GETTER)//callWithErrorHandling内部就是把source当做fn,执行一下
    } else {
      // no cb -> simple effect 对应watchEffect类型
      getter = () => {
        if (instance && instance.isUnmounted) {
          return
        }
        //存在cleanup,执行cleanup，这个cleanup、onInvalidate是在下面定义的，因为这里是定义getter的内容，不是真正执行getter,所以现在cleanup，onInvalidate可能是undefined,但是等到执行getter的时候，肯定是有值的，类似是注册一个listener,event在触发的时候肯定有值
        if (cleanup) {
          cleanup()
        }
        //export type WatchEffect = (onInvalidate: InvalidateCbRegistrator) => void
        return callWithAsyncErrorHandling(//watchEffect,执行并传入onInvalidate 参数，onInvalidate在下面定义
          source,
          instance,
          ErrorCodes.WATCH_CALLBACK,
          [onInvalidate]
        )
      }
    }
  } else {
    getter = NOOP
    __DEV__ && warnInvalidSource(source)
  }

  // 2.x array mutation watch compat
  if (__COMPAT__ && cb && !deep) {
    const baseGetter = getter
    getter = () => {
      const val = baseGetter()
      if (
        isArray(val) &&
        checkCompatEnabled(DeprecationTypes.WATCH_ARRAY, instance)
      ) {
        traverse(val)
      }
      return val
    }
  }

  if (cb && deep) {
    const baseGetter = getter
    getter = () => traverse(baseGetter())
  }

  let cleanup: () => void
  let onInvalidate: InvalidateCbRegistrator = (fn: () => void) => {
    cleanup = effect.onStop = () => {
      callWithErrorHandling(fn, instance, ErrorCodes.WATCH_CLEANUP)//这里的fn应该和上面的source是同一个
    }
  }

  // in SSR there is no need to setup an actual effect, and it should be noop
  // unless it's eager
  if (__SSR__ && isInSSRComponentSetup) {
    // we will also not call the invalidate callback (+ runner is not set up)
    onInvalidate = NOOP
    if (!cb) {
      getter()
    } else if (immediate) {
      callWithAsyncErrorHandling(cb, instance, ErrorCodes.WATCH_CALLBACK, [
        getter(),
        isMultiSource ? [] : undefined,
        onInvalidate
      ])
    }
    return NOOP
  }

  let oldValue = isMultiSource ? [] : INITIAL_WATCHER_VALUE
  // 定一个一个job,watch 执行effect -> 上面的source，得到newValue,判断newValue,oldValue是不是变了。然后执行cb
  const job: SchedulerJob = () => {
    if (!effect.active) {
      return
    }
    if (cb) {
      // watch(source, cb)
      const newValue = effect.run()
      if (
        deep ||
        forceTrigger ||
        (isMultiSource
          ? (newValue as any[]).some((v, i) =>
              hasChanged(v, (oldValue as any[])[i])
            )
          : hasChanged(newValue, oldValue)) ||
        (__COMPAT__ &&
          isArray(newValue) &&
          isCompatEnabled(DeprecationTypes.WATCH_ARRAY, instance))
      ) {
        // cleanup before running cb again
        if (cleanup) {
          cleanup()
        }
        //执行cb，用newValue,oldValue做参数
        callWithAsyncErrorHandling(cb, instance, ErrorCodes.WATCH_CALLBACK, [
          newValue,
          // pass undefined as the old value when it's changed for the first time
          oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue,
          onInvalidate
        ])
        oldValue = newValue
      }
    } else {
      // watchEffect
      effect.run() // watchEffect则执行effect
    }
  }

  // important: mark the job as a watcher callback so that scheduler knows
  // it is allowed to self-trigger (#1727)
  job.allowRecurse = !!cb

    //这里的scheduler 只是对上面job的封装，根据flush 判断执行的时机，把job封装成不同的任务
  let scheduler: EffectScheduler
  if (flush === 'sync') {
    scheduler = job as any // the scheduler function gets called directly
  } else if (flush === 'post') {
    scheduler = () => queuePostRenderEffect(job, instance && instance.suspense)
  } else {
    // default: 'pre'
    scheduler = () => {
      if (!instance || instance.isMounted) {
        queuePreFlushCb(job)
      } else {
        // with 'pre' option, the first call must happen before
        // the component is mounted so it is called synchronously.
        job()
      }
    }
  }
	
   // 利用getter，和scheduler构造effect,内部有个run,就是执行getter,执行的时候会把activeEffect，设置成当前的effect,这样当前的effect里执行getter,时就会触发reactive,ref的proxy 里 get handler,这样就会被里面的依赖收集起来。下次执行reacive的set时就会触发这里的依赖监听，重新执行effect的run,达到触发更新的作用!!!
  const effect = new ReactiveEffect(getter, scheduler)

  if (__DEV__) {
    effect.onTrack = onTrack
    effect.onTrigger = onTrigger
  }

  // initial run 初始执行一次，这样就能执行getter,达到依赖收集
  if (cb) {
    if (immediate) {
      job()
    } else {
      oldValue = effect.run()
    }
  } else if (flush === 'post') {
    queuePostRenderEffect(
      effect.run.bind(effect),
      instance && instance.suspense
    )
  } else {
    effect.run()
  }
	
   //返回一个清理函数
  return () => {
    effect.stop()
    if (instance && instance.scope) {
      remove(instance.scope.effects!, effect)
    }
  }
}
```



我们再看看***ReactiveEffect***

```typescript
packages/reactivity/src/effect.ts

export class ReactiveEffect<T = any> {
  active = true
  deps: Dep[] = [] // 依赖收集

  // can be attached after creation
  computed?: boolean
  allowRecurse?: boolean
  onStop?: () => void
  // dev only
  onTrack?: (event: DebuggerEvent) => void
  // dev only
  onTrigger?: (event: DebuggerEvent) => void
	// constructor里的public 会自动在this上注册属性吗？ this.fn ???
  constructor(
    public fn: () => T,
    public scheduler: EffectScheduler | null = null,
    scope?: EffectScope | null
  ) {
    recordEffectScope(this, scope)
  }

  run() {
    if (!this.active) {
      return this.fn() 
    }
    if (!effectStack.includes(this)) {
      try {
        effectStack.push((activeEffect = this))// 注意这里把activeEffect 设置成了this,这样下面执行this.fn，可以收集依赖，对应下面的effects 的track逻辑
        enableTracking()

        trackOpBit = 1 << ++effectTrackDepth

        if (effectTrackDepth <= maxMarkerBits) {
          initDepMarkers(this)
        } else {
          cleanupEffect(this)
        }
        return this.fn()// run 是执行 this.fn,即上面传入的getter
      } finally {
        if (effectTrackDepth <= maxMarkerBits) {
          finalizeDepMarkers(this)
        }

        trackOpBit = 1 << --effectTrackDepth

        resetTracking()
        effectStack.pop()
        const n = effectStack.length
        activeEffect = n > 0 ? effectStack[n - 1] : undefined
      }
    }
  }

  stop() {
    if (this.active) {
      cleanupEffect(this)
      if (this.onStop) {
        this.onStop()
      }
      this.active = false
    }
  }
}
```

## computed

```typescript
export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>,
  debugOptions?: DebuggerOptions
) {
  let getter: ComputedGetter<T>
  let setter: ComputedSetter<T>

  const onlyGetter = isFunction(getterOrOptions)
  if (onlyGetter) {// 只有get的computed
    getter = getterOrOptions
    setter = __DEV__
      ? () => {
          console.warn('Write operation failed: computed value is readonly')
        }
      : NOOP
  } else {
    getter = getterOrOptions.get// 对应get,set
    setter = getterOrOptions.set
  }
	//利用传入的get,set 构造一个ComputedRefImpl
  const cRef = new ComputedRefImpl(getter, setter, onlyGetter || !setter)

  if (__DEV__ && debugOptions) {
    cRef.effect.onTrack = debugOptions.onTrack
    cRef.effect.onTrigger = debugOptions.onTrigger
  }

  return cRef as any
}


class ComputedRefImpl<T> {
  public dep?: Dep = undefined

  private _value!: T
  private _dirty = true
  public readonly effect: ReactiveEffect<T>

  public readonly __v_isRef = true
  public readonly [ReactiveFlags.IS_READONLY]: boolean

  constructor(
    getter: ComputedGetter<T>,
    private readonly _setter: ComputedSetter<T>,
    isReadonly: boolean
  ) {
    this.effect = new ReactiveEffect(getter, () => {// 构造一个ReactiveEffect，getter是传入的get
      if (!this._dirty) {
        this._dirty = true
        triggerRefValue(this)
      }
    })
    this[ReactiveFlags.IS_READONLY] = isReadonly
  }

  get value() {
    // the computed ref may get wrapped by other proxies e.g. readonly() #3376
    const self = toRaw(this)
    trackRefValue(self)
    if (self._dirty) {
      self._dirty = false
      self._value = self.effect.run()!//执行上面的effect的run
    }
    return self._value
  }

  set value(newValue: T) {
    this._setter(newValue)
  }
}
```

