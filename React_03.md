# React  lane 模型

react lane 是一个32位的二进制数

相关定义在**packages/react-reconciler/src/ReactFiberLane.new.js** 文件中,内部预定义了一系列的lane,用来标识不同任务的优先级

如下所示：

```javascript
export const TotalLanes = 31;

export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;

export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000010;
export const InputContinuousLane: Lanes = /*            */ 0b0000000000000000000000000000100;

export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000001000;
export const DefaultLane: Lanes = /*                    */ 0b0000000000000000000000000010000;

const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000000000000100000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111111111111000000;
const TransitionLane1: Lane = /*                        */ 0b0000000000000000000000001000000;
const TransitionLane2: Lane = /*                        */ 0b0000000000000000000000010000000;
const TransitionLane3: Lane = /*                        */ 0b0000000000000000000000100000000;
const TransitionLane4: Lane = /*                        */ 0b0000000000000000000001000000000;
const TransitionLane5: Lane = /*                        */ 0b0000000000000000000010000000000;
const TransitionLane6: Lane = /*                        */ 0b0000000000000000000100000000000;
const TransitionLane7: Lane = /*                        */ 0b0000000000000000001000000000000;
const TransitionLane8: Lane = /*                        */ 0b0000000000000000010000000000000;
const TransitionLane9: Lane = /*                        */ 0b0000000000000000100000000000000;
const TransitionLane10: Lane = /*                       */ 0b0000000000000001000000000000000;
const TransitionLane11: Lane = /*                       */ 0b0000000000000010000000000000000;
const TransitionLane12: Lane = /*                       */ 0b0000000000000100000000000000000;
const TransitionLane13: Lane = /*                       */ 0b0000000000001000000000000000000;
const TransitionLane14: Lane = /*                       */ 0b0000000000010000000000000000000;
const TransitionLane15: Lane = /*                       */ 0b0000000000100000000000000000000;
const TransitionLane16: Lane = /*                       */ 0b0000000001000000000000000000000;

const RetryLanes: Lanes = /*                            */ 0b0000111110000000000000000000000;
const RetryLane1: Lane = /*                             */ 0b0000000010000000000000000000000;
const RetryLane2: Lane = /*                             */ 0b0000000100000000000000000000000;
const RetryLane3: Lane = /*                             */ 0b0000001000000000000000000000000;
const RetryLane4: Lane = /*                             */ 0b0000010000000000000000000000000;
const RetryLane5: Lane = /*                             */ 0b0000100000000000000000000000000;

export const SomeRetryLane: Lane = RetryLane1;

export const SelectiveHydrationLane: Lane = /*          */ 0b0001000000000000000000000000000;

const NonIdleLanes = /*                                 */ 0b0001111111111111111111111111111;

export const IdleHydrationLane: Lane = /*               */ 0b0010000000000000000000000000000;
export const IdleLane: Lanes = /*                       */ 0b0100000000000000000000000000000;

export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

其中，**数字越小优先级越高**，可以看到 **SyncLane= 0b0000000000000000000000000000001**，优先级最高,然后往下优先级逐渐降低

lane模型之前react内部采用的是ExpirationTime来标识任务的优先级。但是ExpirationTime 在表示批量任务时一些问题，最后选择了lane模型，具体可以看看下面这些文章

[React 为什么使用 Lane 技术方案](https://juejin.cn/post/6951206227418284063#heading-5)

[What is "Lane" in React?](https://dev.to/okmttdhr/what-is-lane-in-react-4np7)

我们来看看内部使用lane的一些操作

```javascript
//合并2个lane,这个方法在markUpdateLaneFromFiberToRoot中有调用，主要是把lane进行合并（比如把lane合并到parent.childLanes上）
export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}

//二进制按位或 则为1的位会保留在结果中 比如：
mergeLanes(
  NoLane /*0b0000000000000000000000000000000*/,
  OffscreenLane /*0b1000000000000000000000000000000*/
)
// => 0b1000000000000000000000000000000
//这样合并后的结果就同时具有了2个lane的信息
//sourceFiber.lanes mergeLanes 后，对应的位上就会有lane 的 1
sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane)


//判断是否是子集，是否包含
export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane) {
  return (set & subset) === subset;// a & b  二进制 按位与 等于自身 说明 b 中 对应为 1的 位 在 a中 同样也为 1 
}

//比如：

isSubsetOfLanes(
  NonIdleLanes, /*0b0001111111111111111111111111111*/
  SyncLane /*0b0000000000000000000000000000001*/
)
// => true. SyncLane is not Idle task

//把某个lane从lanes中去除
export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}
//比如:
//SyncLane     								0b0000000000000000000000000000001
//~SyncLane    								0b1111111111111111111111111111110
//NonIdleLanes 								0b0001111111111111111111111111111
//NonIdleLanes & ~SyncLane  	0b0001111111111111111111111111110  // 可以看到最后一位1去掉了


// ~subset 按位取反


//判断是否包含
export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane) {
  return (a & b) !== NoLanes;
}

// NoLanes 0b0000000000000000000000000000000

// (a & b) != NoLanes 说明 a 和 b 存在相同位上的同时为 1
// 类似 还有 (executionContext & RenderContext) !== NoLanes 这种，只有在executionContext  = RenderContext 时成立
// context 只有预定义的几种

```

[按位非 (~)](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Bitwise_NOT) 按位非运算时，任何数字`x的运算结果都是``-(x + 1)`。例如，`〜-5`运算结果为`4`

通过32位2进制数，通过merge,达到 多个对应位的数字为1，就可以很好的表达批的概念 

例如 Lanes 为 17 时，表示将批量更新 SyncLane（值为 1）和 DefaultLane（值为 16）的任务









