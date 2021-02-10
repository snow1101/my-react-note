### key 值的作用

1. react源码中会使用key 和 type 标识当前元素的唯一性
2. diff 算法中 进行比较

### ref 相关
*  创建ref 尽量不要使用内联，因为内联函数在更新时会被执行两次.官网上这么说**如果 ref 回调函数是以内联函数的方式定义的，在更新过程中它会被执行两次，第一次传入参数 null，然后第二次会传入参数 DOM 元素。这是因为在每次渲染时会创建一个新的函数实例，所以 React 清空旧的 ref 并且设置新的。通过将 ref 的回调函数定义成 class 的绑定函数的方式可以避免上述问题，但是大多数情况下它是无关紧要的。**

```javascript
// 推荐使用
this.setInputInstance = ele => this.inputInstance = ele;
<input ref={this.setInputInstance}/>

//不建议使用, 因为内联函数在更新对比的时候 永远是false，所以会先执行卸载，然后在执行挂载
<input ref={ele => this.inputInstance = ele}/>
```


* 默认情况下，你不能在函数组件上使用 ref 属性，因为它们没有实例。如果要在函数组件中使用 ref，你可以使用 forwardRef（可与 useImperativeHandle 结合使用），或者可以将该组件转化为 class 组件。