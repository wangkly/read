

# Vue3 reactive ref

## reactive

reactive的源码在packages/reactivity/src/reactive.ts下

```typescript
export function reactive<T extends object>(target: T): UnwrapNestedRefs<T>
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap
  )
}
```

可以看到这边reactive 有一个重载，内部调用***createReactiveObject*** 并将结果返回，我们在看看***createReactiveObject***

```typescript
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  if (!isObject(target)) {//如果target 不是一个object,直接返回target
    return target
  }
  // target is already a Proxy, return it.targert 已经是一个proxy
  // exception: calling readonly() on a reactive object
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  // target already has corresponding Proxy
  const existingProxy = proxyMap.get(target)// 这里有个proxyMap缓存
  if (existingProxy) {
    return existingProxy
  }
  // only a whitelist of value types can be observed.
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target
  }
  const proxy = new Proxy(// 可以看到内部是用target 创建了一个ES6 proxy,
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers//根据targetType 使用不同handler，一般是baseHandler
  )
  proxyMap.set(target, proxy)// 做缓存
  return proxy
}
```

这里主要是对target new Proxy, 我们先看看baseHandlers的内容，这里面有一系列对target对象的拦截操作

packages/reactivity/src/baseHandlers.ts

```typescript
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}
```

我们主要看看 get,set的拦截

```typescript
function createGetter(isReadonly = false, shallow = false) { // isReadonly 默认是false的
  return function get(target: Target, key: string | symbol, receiver: object) {
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (
      key === ReactiveFlags.RAW &&
      receiver ===
        (isReadonly
          ? shallow
            ? shallowReadonlyMap
            : readonlyMap
          : shallow
            ? shallowReactiveMap
            : reactiveMap
        ).get(target)
    ) {
      return target
    }

    const targetIsArray = isArray(target)// 先判断target是不是数组

    if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
      return Reflect.get(arrayInstrumentations, key, receiver)
    }

    const res = Reflect.get(target, key, receiver)// 取出target对应的key

    if (
      isSymbol(key)
        ? builtInSymbols.has(key as symbol)
        : isNonTrackableKeys(key)
    ) {
      return res
    }

    if (!isReadonly) { // 注意这里的track(),在track内部 进行了依赖的收集
      track(target, TrackOpTypes.GET, key)
    }

    if (shallow) {
      return res
    }

    if (isRef(res)) {
      // ref unwrapping - does not apply for Array + integer key.
      const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
      return shouldUnwrap ? res.value : res
    }

    if (isObject(res)) {
      // Convert returned value into a proxy as well. we do the isObject check
      // here to avoid invalid value warning. Also need to lazy access readonly
      // and reactive here to avoid circular dependency.
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

其中重点的是调用了**track**进行了依赖的收集，我们看看track的逻辑

packages/reactivity/src/effect.ts  effects里主要有track trigger ，依赖收集，触发更新

```typescript
export function track(target: object, type: TrackOpTypes, key: unknown) {
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  let depsMap = targetMap.get(target) // targetMap 缓存，主要是target的依赖map
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Set()))
  }
  if (!dep.has(activeEffect)) { // 注意这里的  activeEffect ，容易引起误解，一开始进行proxy handler的注册的时候，这里是不会有值的，proxy 相当于在target上做了一个拦截的listener,如果不取target里的key对应的值，这里的代码不会执行，只有在代码中使用了target[key],才会触发这边的监听回调，在那个时候activeEffect 会有值，也就是watch computed这些内部会封装一个effects ,取target[key]的时候，会把它们内部的effects设置成activeEffect ，！！！注意这里有点疑惑，就把它理解成事件监听，在事件触发的时候肯定会有events
    dep.add(activeEffect) // 添加当前的effects 进依赖，这样，触发proxy set的时候能给通知effects 更新
    activeEffect.deps.push(dep)
    if (__DEV__ && activeEffect.options.onTrack) {
      activeEffect.options.onTrack({
        effect: activeEffect,
        target,
        type,
        key
      })
    }
  }
}
```



注意track里的 ***activeEffect***，它在运行时才会有值

再看set的逻辑

```typescript
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    let oldValue = (target as any)[key]//取出旧值
    if (!shallow) {
      value = toRaw(value)
      oldValue = toRaw(oldValue)
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {//如果值变了，触发更新 trigger
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```

看看trigger的逻辑

```typescript
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)// 取出target的depsMap,track的时候存的
  if (!depsMap) {
    // never been tracked
    return
  }
	//初始化一个effects的set,下面的add，主要是往这个set内添加effect,这个effects在下面会用于调用run
  const effects = new Set<ReactiveEffect>()
  const add = (effectsToAdd: Set<ReactiveEffect> | undefined) => {
    if (effectsToAdd) {
      effectsToAdd.forEach(effect => {
        if (effect !== activeEffect || effect.allowRecurse) {
          effects.add(effect)
        }
      })
    }
  }

  if (type === TriggerOpTypes.CLEAR) {
    // collection being cleared
    // trigger all effects for target
    depsMap.forEach(add)
  } else if (key === 'length' && isArray(target)) {
    depsMap.forEach((dep, key) => {
      if (key === 'length' || key >= (newValue as number)) {
        add(dep)
      }
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    if (key !== void 0) {
      add(depsMap.get(key))
    }

    // also run for iteration key on ADD | DELETE | Map.SET
    switch (type) {
      case TriggerOpTypes.ADD://这个在baseHandler里触发trigger type一般是TriggerOpTypes.ADD，TriggerOpTypes.SET
        if (!isArray(target)) {
          add(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            add(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        } else if (isIntegerKey(key)) {
          // new index added to array -> length changes
          add(depsMap.get('length'))
        }
        break
      case TriggerOpTypes.DELETE:
        if (!isArray(target)) {
          add(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            add(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        }
        break
      case TriggerOpTypes.SET:
        if (isMap(target)) {
          add(depsMap.get(ITERATE_KEY))
        }
        break
    }
  }

  const run = (effect: ReactiveEffect) => {
    if (__DEV__ && effect.options.onTrigger) {
      effect.options.onTrigger({
        effect,
        target,
        key,
        type,
        newValue,
        oldValue,
        oldTarget
      })
    }
    if (effect.options.scheduler) {// 这个在watch，computed内会设置，就是传入的callback
      effect.options.scheduler(effect)
    } else {
      effect()
    }
  }
	//遍历所有的effects，每一个都调用run
  //注意这里effects的类型是Set<ReactiveEffect> 
  effects.forEach(run)
}

// 来看下ReactiveEffect类型的定义，
export interface ReactiveEffect<T = any> {
  (): T // 这是一个function的类型定义
  _isEffect: true
  id: number
  active: boolean
  raw: () => T
  deps: Array<Dep>
  options: ReactiveEffectOptions
  allowRecurse: boolean
}
```

targetMap 是一个weakMap,取出targetMap[target] depsMap 里存的是 一个个的effect ,这些effect是watch ,computed等实现内部创建出来，存入depMaps里的，在track的逻辑里有这些将effects 存入 depMaps的逻辑，这里我们可以确认 effects 里存放的是 effect,这些effect 拿出来遍历执行

看到这里，我们可以明白，vue3的依赖收集，和触发更新的逻辑

Reactive  内部，把target 通过  new Proxy(target,handler) 创建代理，在handler 内部，主要看get 、set的逻辑，get的时候，通过添加的handler ,在取值的同时，将 activeEffect 存入 depMaps (这里effect 是通过 watch 、computed 内部 new reactiveEffect 创建 )，主要是track 进行依赖收集

在对target的key 进行 set操作的时候、触发trigger 然后取出depMaps内的effects ，然后遍历执行，关于effect的内容，在后面进行介绍



## Ref

```typescript
export function ref<T extends object>(value: T): ToRef<T>
export function ref<T>(value: T): Ref<UnwrapRef<T>>
export function ref<T = any>(): Ref<T | undefined>
export function ref(value?: unknown) {
  return createRef(value)
}

function createRef(rawValue: unknown, shallow = false) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}


class RefImpl<T> {
  private _value: T

  public readonly __v_isRef = true

  constructor(private _rawValue: T, public readonly _shallow = false) {
    this._value = _shallow ? _rawValue : convert(_rawValue)
  }

  get value() {
    track(toRaw(this), TrackOpTypes.GET, 'value') // 和reactive一样，收集依赖
    return this._value
  }

  set value(newVal) {
    if (hasChanged(toRaw(newVal), this._rawValue)) { // 如果值变了，触发更新
      this._rawValue = newVal
      this._value = this._shallow ? newVal : convert(newVal)
      trigger(toRaw(this), TriggerOpTypes.SET, 'value', newVal) // 触发更新
    }
  }
}
```

Reactive 和 Ref 内部，就是通过track,trigger 收集依赖，触发更新。其中proxy 的handler 是在真正取值的时候才会执行，这时<font color="red">**activeEffect**</font>才会有赋值