# redux源码解析
---

###   createStore

一般项目开始的时候我们会这样写
```
const initialState = {};
// 这里是一些中间件，像log之类的
const middlewares = [
    thunkMiddleware,
    promiseMiddleWare,
    localstorageMiddware,
    logger
  ];
const store = createStore(combineReducers({home,common}), initialState, applyMiddleware(...middlewares));
```

首先执行执行的就是combineReducers，home的示例代码如下

```
export default function(state = initState, action) {
  switch (action.type) {
    case 'GET_AGENT_SUCCESS':
    {
      const agentInfo = action.payload;
      return Object.assign({}, state, { agentInfo });
    }
    case 'GET_MARQUEE_SUCCESS':
    {
      const marQuee = action.payload.bizRecords;
      return Object.assign({}, state, { marQuee });

    }
    default:
      return state;
  }
}
```
combineReducers 通过assertReducerSanity 方法，使用createStore里面暴露的ActionTypes = {INIT: '@@redux/INIT'}将所有reducer初始化了一遍，并返回函数combination

然后执行applyMiddleware()，此函数返回的是一个匿名函数，接收参数为createStore函数

```
export default function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState, enhancer) => {
    var store = createStore(reducer, preloadedState, enhancer)
    var dispatch = store.dispatch
    var chain = []

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```
此时终于进入了createStore这个函数了，createStore函数中有一段代码如下
```
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, preloadedState)
  }
```
  因为我们传入了第三个参数enhancer，也就是我们的applyMiddleware返回的匿名函数，所以函数直接return了enhancer(createStore)(reducer, preloadedState)

  分析enhancer(createStore)(reducer, preloadedState)

  首先将createStore函数作为参数传入给applyMiddleware()返回的匿名函数，执行第一步enhancer(createStore)，返回的是另一个匿名函数，接受参数reducer, preloadedState, enhancer

  此时执行第二步enhancer(createStore)(reducer, preloadedState)，传入的参数是reducer和preloadedState
  注明：  这里的reducer 是我们最开始执行的combineReducers返回的函数combination， preloadedState就是我们的 initialState = {};

  在执行enhancer(createStore)(reducer, preloadedState)时，第一步就是代码  var store = createStore(reducer, preloadedState, enhancer)
  所以此时，代码再次进入createStore 里面，但是由于我们此次并没有传入enhancer，所以createStore顺利向下执行，初始化一些参数
```
  var currentReducer = reducer  -> reducer是函数combination
  var currentState = preloadedState -> currentState是initialState = {}
  var currentListeners = []
  var nextListeners = currentListeners
  var isDispatching = false
```
  代码执行到最后遇到关键的一句话，在createStore 函数return前 执行了一次 dispatch({ type: ActionTypes.INIT })

  代码进入了createStore 函数中的方法dispatch
```
  function dispatch(action) {
    //如果不是对象报错
  if (!isPlainObject(action)) {
    throw new Error(
      'Actions must be plain objects. ' +
      'Use custom middleware for async actions.'
    )
  }
  //如果没有t对象中没有type属性报错
  if (typeof action.type === 'undefined') {
    throw new Error(
      'Actions may not have an undefined "type" property. ' +
      'Have you misspelled a constant?'
    )
  }
  //如果正在dispatch 报错
  if (isDispatching) {
    throw new Error('Reducers may not dispatch actions.')
  }

  try {
    isDispatching = true
    currentState = currentReducer(currentState, action)
  } finally {
    isDispatching = false
  }

  var listeners = currentListeners = nextListeners
  for (var i = 0; i < listeners.length; i++) {
    listeners[i]()
  }

  return action
}
```
在执行到try代码块里面的时候，开始执行combination函数，传入currentState即初始化的initialState和初始化的action即{ type: ActionTypes.INIT }
combination代码如下
```
function combination(state = {}, action) {
    if (sanityError) {
      throw sanityError
    }

    if (process.env.NODE_ENV !== 'production') {
      var warningMessage = getUnexpectedStateShapeWarningMessage(state, finalReducers, action, unexpectedKeyCache)
      if (warningMessage) {
        warning(warningMessage)
      }
    }

    var hasChanged = false
    var nextState = {}
    for (var i = 0; i < finalReducerKeys.length; i++) {
      var key = finalReducerKeys[i]
      var reducer = finalReducers[key]
      var previousStateForKey = state[key]
      var nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        var errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    return hasChanged ? nextState : state
  }
  ```
可以看到，在for循环里面，每个reducer又被执行了一遍，并将结果挂在了nextState上面，最后判断nextStateForKey 和 previousStateForKey 是否相同，一旦一个发生变化，
hasChanged 就会变成true 将nextState 返回，赋值给了createStore中的currentState 然后此时nextListeners 还是[]，所以直接返回了action {type: "@@redux/INIT"}

此时dispatch 结束， createStore 返回了对象，并赋值给了applyMiddleware中的store；
{
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
代码此时进入了middlewares.map处

```
export default function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState, enhancer) => {
    var store = createStore(reducer, preloadedState, enhancer)
    var dispatch = store.dispatch
    var chain = []

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```
middlewares.map 做的最重要的一件事情就是将createStore的getState和dispatch两个方法传入到每个middleware中
比如 thunkMiddleware
```
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```
此时middlewares.map 里面所有的此时middleware 都执行了一遍 ，传入了dispatch和getState，返回了一个匿名函数，接收一个函数作为参数， 将所有执行结果产生的集合
赋值给chain，接着执行compose(...chain)，compose代码如下
```
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  const last = funcs[funcs.length - 1]
  const rest = funcs.slice(0, -1)
  return (...args) => rest.reduceRight((composed, f) => f(composed), last(...args))
}
```
可以看到 compose 返回了一个匿名函数，此时代码回到compose(...chain)(store.dispatch)  将store.dispatch 作为参数传入到了 compose 返回的匿名函数中
此时reduceRight 会逐个 执行middleware   返回了最后一个middleware执行后的结果，即一个接收action作为参数的函数 ，并赋值给了applyMiddleware 中的dispatch

此时applyMiddleware 最终全部返回{...store,dispatch}，赋值给了 最开始的代码store
const store = createStore(combineReducers({home,common}), initialState, applyMiddleware(...middlewares));

所以此时的store，有createStore方法中的
{
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
  但是dispatch 方法已经是被此时applyMiddleware  改造后的dispatch 了，即接收action作为参数的函数
