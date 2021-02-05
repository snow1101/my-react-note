# react 性能优化相关

## 常见的组件优化技术

### class 组件优化
*  shouldComponentUpdate
*  PureComponent - 组件内部有自己的浅比较逻辑

##### 源码如下 (ReactFiberClassComponent.old.js 文件)
```javascript

function checkShouldComponentUpdate(
  workInProgress,
  ctor,
  oldProps,
  newProps,
  oldState,
  newState,
  nextContext,
) {
  const instance = workInProgress.stateNode;
  // 如果组件有自己的shouldComponentUpdate 方法就执行
  if (typeof instance.shouldComponentUpdate === 'function') {
    if (__DEV__) {
      if (
        debugRenderPhaseSideEffectsForStrictMode &&
        workInProgress.mode & StrictMode
      ) {
        disableLogs();
        try {
          // Invoke the function an extra time to help detect side-effects.
          instance.shouldComponentUpdate(newProps, newState, nextContext);
        } finally {
          reenableLogs();
        }
      }
    }
    const shouldUpdate = instance.shouldComponentUpdate(
      newProps,
      newState,
      nextContext,
    );

    if (__DEV__) {
      if (shouldUpdate === undefined) {
        console.error(
          '%s.shouldComponentUpdate(): Returned undefined instead of a ' +
            'boolean value. Make sure to return true or false.',
          getComponentName(ctor) || 'Component',
        );
      }
    }

    return shouldUpdate;
  }
// 如果是PureComponent 就进行浅比较
  if (ctor.prototype && ctor.prototype.isPureReactComponent) {
    return (
      !shallowEqual(oldProps, newProps) || !shallowEqual(oldState, newState)
    );
  }
// 默认返回true
  return true;
}

```

### 函数组件优化

* memo - 常用于优化组件

```javascript
const MemoCounter = memo(
  props => {
    console.log("MemoCounter");
    return <div>MemoCounter-
{props.count.count}</div>;
  },
  (prevProps, nextProps) => {
    return return preProps.count.count ===
nextProps.count.count;
} );
```
* useMemo - 常用于组件中避免不必要的运算

```javascript
 
// 它仅会在某个依赖项(count)改变时才重新计算 memoized 值。这 种优化有助于避免在每次渲染时都进⾏高开销的计算。import React, { useState, useMemo } from "react";
export default function UseMemoPage(props)
{
  const [count, setCount] = useState(0);
  const expensive = useMemo(() => {
    console.log("compute");
    let sum = 0;
    for (let i = 0; i < count; i++) {
	sum += i; }
	return sum;
	//只有count变化，这⾥里里才重新执⾏行行
  }, [count]);
const [value, setValue] = useState(""); 
return (
    <div>
      <h3>UseMemoPage</h3>
       <p>expensive:{expensive}</p>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>add</button>
      <input value={value} onChange={event => setValue(event.target.value)} />
    </div>
); }
```
* useCallback 

##### 源码如下 (ReactFiberHooks.old.js)

```Javascript
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    // Assume these are defined. If they're not, areHookInputsEqual will warn.
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  hook.memoizedState = [callback, nextDeps];
  return callback;
}

```
