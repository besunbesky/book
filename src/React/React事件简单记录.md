# React事件机制

## DOM事件

冒泡和捕获

![image-20210624173959717](./images/image-20210624173959717.png)

先从父元素向下传递捕获，直到子元素处理掉，然后再逐个冒泡。

所以有了一个事件委托的机制。

## React事件

React会将所有事件都绑定在document上。

统一使用事件监听。都是在冒泡阶段处理。

所以一般在组件挂载的时候增加监听事件。

组件卸载的时候删除监听事件。

事件触发的时候，组件会生成一个合成事件。然后发送到document上。

document会通过dispatch event回调函数依次执行dispatch listener中同类型事件的监听函数。

事件注册是在组件生成的时候，将VDOM中的所有的事件对应的原生事件都注册在Document中一个监听器中。所有的事件处理函数都存放在listenerbank中，并以key做为索引。（将可能要触发的事件分门别类）

1. 是合成事件，不是DOM原生事件
2. 在document监听所有支持事件
3. 使用统一的分发函数dispatchEvent来指定事件函数的执行

[拓展阅读1](https://juejin.cn/post/6844903502729183239)

[拓展阅读2](http://zhenhua-lee.github.io/react/react-event.html)
