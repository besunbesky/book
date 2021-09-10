# Vue2.x 响应式原理

众所周知，Vue2.x是基于Object.defineProperty()来实现响应式的。

那么，defineProperty是啥呢。

先附上MDN的连接[Object.defineProperty()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)

## Object.defineProperty

引用MDN的原话：

> `Object.defineProperty()`方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回此对象。

那么重点有两个：

1. 对象
2. 定义新属性或修改现有属性

那么什么叫`定义新属性或修改现有属性`呢？

我们来看看另外一个概念`属性描述符`

## 属性描述符

### 描述符的介绍

对象里目前存在的属性描述符有两种主要形式：

- 数据描述符：具有值的属性，该值可以是可写的，也可以是不可写的。
- 存取描述符：由getter和setter函数描述的属性。

一个描述符只可能是二者之一，不可能同时是两者。即二选一。

上述两种描述符都是对象，共享以下可选键值。（默认值是指在使用 `Object.defineProperty()` 定义属性时的默认值）

| 键值           | 描述                                                                    | 默认值       |
| ------------ | --------------------------------------------------------------------- | --------- |
| configurable | 当且仅当该属性的 `configurable` 键值为 `true` 时，该属性的描述符才能够被改变，同时该属性也能从对应的对象上被删除。 | false     |
| enumerable   | 当且仅当该属性的 `enumerable` 键值为 `true` 时，该属性才会出现在对象的枚举属性中。                  | false     |
| value        | 该属性对应的值。可以是任何有效的 JavaScript 值（数值，对象，函数等）。                             | undefined |
| writable     | 当且仅当该属性的 `writable` 键值为 `true` 时，属性的值，也就是上面的 `value`，才能被赋值。           | false     |

> 因为`enumerable`属性默认是false，所以通过`Object.defineProperty()`定义的属性都是不会被`for...in`和`Object.keys`访问到。

以下是存取描述符独有的：

| 键值  | 描述                                                                                                                                 | 默认值       |
| --- | ---------------------------------------------------------------------------------------------------------------------------------- | --------- |
| get | 属性的 getter 函数，如果没有 getter，则为 `undefined`。当访问该属性时，会调用此函数。执行时不传入任何参数，但是会传入 `this` 对象（由于继承关系，这里的`this`并不一定是定义该属性的对象）。该函数的返回值会被用作属性的值。 | undefined |
| get | 属性的 setter 函数，如果没有 setter，则为 `undefined`。当属性值被修改时，会调用此函数。该方法接受一个参数（也就是被赋予的新值），会传入赋值时的 `this` 对象。                                   | undefined |

可以看到，跟值相关的，默认值都是undefined。跟`able`相关的，默认值都是false。

### 怎么区分描述符呢？

|       | configurable | enumerable | value | writable | get | set |
| :---: | :----------: | :--------: | :---: | :------: | :-: | :-: |
| 数据描述符 |      √       |     √      |   √   |    √     |  ×  |  ×  |
| 存取描述符 |      √       |     √      |   ×   |    ×     |  √  |  √  |

如果一个描述符不具有 `value`、`writable`、`get` 和 `set` 中的任意一个键，那么它将被认为是一个数据描述符。如果一个描述符同时拥有 `value` 或 `writable`
和 `get` 或 `set` 键，则会产生一个异常。

了解到这里已经够我们去理解Vue2.x的响应式原理了。

我们再填一下上面的坑：

如果对象中不存在指定的属性，`Object.defineProperty()` 会创建这个属性。当描述符中省略某些字段时，这些字段将使用它们的默认值。 搬一下MDN的代码：

```javascript
var o = {}; // 创建一个新对象

// 在对象中添加一个属性与数据描述符的示例
Object.defineProperty(o, "a", {
  value : 37,
  writable : true,
  enumerable : true,
  configurable : true
});

// 对象 o 拥有了属性 a，值为 37

// 在对象中添加一个设置了存取描述符属性的示例
var bValue = 38;
Object.defineProperty(o, "b", {
  // 使用了方法名称缩写（ES2015 特性）
  // 下面两个缩写等价于：
  // get : function() { return bValue; },
  // set : function(newValue) { bValue = newValue; },
  get() { return bValue; },
  set(newValue) { bValue = newValue; },
  enumerable : true,
  configurable : true
});

o.b; // 38
// 对象 o 拥有了属性 b，值为 38
// 现在，除非重新定义 o.b，o.b 的值总是与 bValue 相同

// 数据描述符和存取描述符不能混合使用
Object.defineProperty(o, "conflict", {
  value: 0x9f91102,
  get() { return 0xdeadbeef; }
});
// 抛出错误 TypeError: value appears only in data descriptors, get appears only in accessor descriptors
```

## 响应式原理

所谓响应式，即，一个值变化的时候要根据这个变化，产生相应的行为。

在浏览器上，dom渲染完成之后，那么如果你不主动通过dom修改操作来重新渲染，那么这个dom就永远不变了。
那如果dom上渲染的是X的值，X变了怎么办呢。难不成你每次给X赋值的时候，都手动document.balabala吗？

看过上面的set属性之后你可能就想明白了，重点再复习一下：

**当属性值被修改时，会调用此函数。**

锵锵锵，简单来说就是用`get`和`set`来实现的啦。

### 简单实现

话不多说，直接上代码 。

```javascript
let x;

let f = function (v) {
  return v * 100;
}

// 变化之后的处理函数
let active;

const onXChange = (cb) => {
  active = cb;
  // 初始化的时候就需要执行一次
  active()
}

// 工厂
const ref = (value) => {
  let initValue = value;
  return Object.defineProperty({}, "value", {
    get() {
      return initValue;
    },
    set(newVal) {
      initValue = newVal;
      active();
    }
  })
}

// 初始化
x = ref(1);

// 添加回调
onXChange(() => {
  console.log(f(x.value));
})

x.value = 2;
x.value = 3;
```

上述的写法有点问题。如果，不光是一个callback，有多个callback如何处理？

一个active不够用。所以我们需要一个地方进行依赖收集。（即，多个组件依赖这个数据，每个组件都需要进行变化。）

### 带依赖收集

变化后的代码如下：

```javascript
/* eslint-disable no-debugger */
let x;

let f = function (v) {
  return v * 100;
}

let active;

const onXChange = (cb) => {
  active = cb;
  active();
  active = null;
}
class Dep {
  deps = new Set();
  depend() {
    if (active) {
      this.deps.add(active);
    }
  }
  notify() {
    this.deps.forEach(dep => dep());
  }
}

const ref = (value) => {
  let initValue = value;
  let deps = new Dep();

  return Object.defineProperty({}, "value", {
    get() {
      deps.depend();
      return initValue;
    },
    set(newVal) {
      initValue = newVal;
      deps.notify();
    }
  })
}

x = ref(1);

onXChange(() => {
  console.log(f(x.value));
})

onXChange(() => {
  console.log(x.value + 1);
})

x.value = 2;
x.value = 3;
```
