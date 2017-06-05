# react
react &amp; redux

## 源码解析

###   createStore
```
//上代码
const initialState = {};
const store = createStore(combineReducers({home,common}), initialState, applyMiddleware(...middlewares));

/*
首先执行执行的就是combineReducers，home的示例代码如下
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

combineReducers 通过assertReducerSanity 方法，使用createStore里面暴露的ActionTypes = {INIT: '@@redux/INIT'}将所有reducer初始化了一遍，并返回函数combination
*/

/*
然后执行applyMiddleware()，此函数返回的是一个匿名函数，接收参数为createStore函数
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

*/

/*
  此处终于进入了createStore这个函数了
  createStore函数中
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, preloadedState)
  }
  因为我们传入了第三个参数enhancer，也就是我们的applyMiddleware返回的匿名函数，所以函数直接return了enhancer(createStore)(reducer, preloadedState)

  分析enhancer(createStore)(reducer, preloadedState)

  首先将createStore函数作为参数传入给applyMiddleware()返回的匿名函数，执行第一步enhancer(createStore)，返回的是另一个匿名函数，接受参数reducer, preloadedState, enhancer

  此时执行第二步enhancer(createStore)(reducer, preloadedState)，传入的参数是reducer和preloadedState
  注明：  这里的reducer 是我们最开始执行的combineReducers返回的函数combination， preloadedState就是我们的 initialState = {};

  在执行enhancer(createStore)(reducer, preloadedState)时，第一步就是代码  var store = createStore(reducer, preloadedState, enhancer)
  所以此时，代码再次进入createStore 里面，但是由于我们此次并没有传入enhancer，所以createStore顺利向下执行，初始化一些参数

  var currentReducer = reducer  -> reducer是函数combination
  var currentState = preloadedState -> currentState是initialState = {}
  var currentListeners = []
  var nextListeners = currentListeners
  var isDispatching = false

  代码执行到最后遇到关键的一句话，在createStore 函数return前 执行了一次 dispatch({ type: ActionTypes.INIT })

  代码进入了createStore 函数中的方法dispatch
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

在执行到try代码块里面的时候，开始执行combination函数，传入currentState即初始化的initialState和初始化的action即{ type: ActionTypes.INIT }
combination代码如下
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

*/
```
