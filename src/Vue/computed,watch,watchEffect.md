# computed,watch,watchEffect

今天来实现一下vue里的computed，watch和watchEffect函数。

先把前面的代码简单捞出来，做一些修改。

使得页面上有个按钮，点击按钮，count增加。（万物始于计数器）

```javascript
let active;

const watchEffect = (cb) => {
  active = cb;
  active();
  active = null;
}

let nextTick = cb => Promise.resolve().then(cb)

let queue = [];
let queueJob = job => {
  if (!queue.includes(job)) {
    queue.push(job);
    nextTick(flushJobs);
  }
}
let flushJobs = () => {
  let job;
  while ((job = queue.shift()) !== undefined) {
    job();
  }
}
class Dep {
  deps = new Set();
  depend() {
    if (active) {
      this.deps.add(active);
    }
  }
  notify() {
    this.deps.forEach(dep => queueJob(dep));
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

let count = ref(0);

document.getElementById('add').addEventListener('click', () => {
  count.value++;
})

watchEffect(() => {
  let str = `count is: ${count.value}`;
  document.getElementById('app').innerText = str;
})
```

## watchEffect

watchEffect是一个函数，接受一个函数。函数内部依赖的值变了，函数就会立即执行。就跟我们之前的watch一模一样。所以，直接给他改个名就行了。

但是还要返回一个stop，用来停止监听，这个最后再说。

## computed

computed接受一个函数，内部依赖某个值，并返回一个新的值。内部还有一个缓存。当依赖的值变化的时候才会重新求值。

### 简单实现

简单实现一个computed函数：

```javascript
let computed = (fn) => {
  // 使用闭包来保存一个value
  let value;

  return {
    get value() {
      // 在get中响应式监听，直接在get中执行一次fn即可获取到计算后的值。
      value = fn();
      return value;
    }
  }
}
```

上述代码已经可以简单的实现一个计算属性。

### 缓存功能

但是我们的computed是有缓存的，而且如果多处调用计算属性，那么将会进行多次计算。为了减少计算。我们也要使用缓存。

通过下面的代码可以发现问题：

```javascript
let active;

const watchEffect = (cb) => {
  active = cb;
  active();
  active = null;
}

let nextTick = cb => Promise.resolve().then(cb)

let queue = [];
let queueJob = job => {
  if (!queue.includes(job)) {
    queue.push(job);
    nextTick(flushJobs);
  }
}
let flushJobs = () => {
  let job;
  while ((job = queue.shift()) !== undefined) {
    job();
  }
}
class Dep {
  deps = new Set();
  depend() {
    if (active) {
      this.deps.add(active);
    }
  }
  notify() {
    this.deps.forEach(dep => queueJob(dep));
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

let computedCount = 0;
let computed = (fn) => {
  // 使用闭包来保存一个value
  let value;

  return {
    get value() {
      // 在get中响应式监听，直接在get中执行一次fn即可。
      value = fn();
      computedCount += 1;
      return value;
    }
  }
}

let count = ref(0);

let computeCount = computed(() => count.value + 1)

document.getElementById('add').addEventListener('click', () => {
  count.value++;
})

watchEffect(() => {
  let str = `count is: ${count.value} \n computeCount is: ${computeCount.value} \n computedCount: ${computedCount}`;
  document.getElementById('app1').innerText = str;
})

watchEffect(() => {
  document.getElementById('app2').innerText = `computeCount 2 is : ${computeCount.value} \n computedCount: ${computedCount}`
})
```

我们在computed中增加一个flag来控制是否进行计算。

```javascript
let computed = (fn) => {
  // 使用闭包来保存一个value
  let value;
  let dirty = true;

  return {
    get value() {
      // 如果值没有发生变化，返回缓存中的值。
      // 也可以保证多处依赖的时候不会重复计算。
      if (dirty) {
        // 在get中响应式监听，直接在get中执行一次fn即可。
        value = fn();
        computedCount += 1;
        // 置为false
        dirty = false;
      }
      return value;
    }
  }
}
```

那么问题来了，既然置为了false，那么何时何地再置为true呢。

显然，在我们触发了set操作。在赋值过后的广播行为里，将所有computed属性里的flag都置为true。

对此需要进行一些改造。

> 此刻之前的代码，ref中，`initialValue`和`value`的定义不符合语义化，已更改。

```javascript
let active;

let effect = (fn, options = {}) => {
  // 定义一个effectInner来对active进行赋值和执行的操作。
  let effectInner = (...args) => {
    try {
      // 常规执行，将自身赋值给active，以在收集依赖的时候调用。
      active = effectInner;
      // 直接执行fn，防止递归调用。然后返回。
      // return active(...args);
      return fn(...args);
    } finally {
      // 为了能最终将active值为空。使用try catch finally的特性。
      active = null;
    }
  }
  // 给effectInner上绑定一个options对象。以在广播的时候调用。
  effectInner.options = options;

  return effectInner;
}

const watchEffect = (cb) => {
  // 替换为effect
  let runner = effect(cb);
  runner();
}

let nextTick = cb => Promise.resolve().then(cb)

let queue = [];
let queueJob = job => {
  if (!queue.includes(job)) {
    queue.push(job);
    nextTick(flushJobs);
  }
}
let flushJobs = () => {
  let job;
  while ((job = queue.shift()) !== undefined) {
    job();
  }
}
class Dep {
  deps = new Set();
  depend() {
    if (active) {
      this.deps.add(active);
    }
  }
  notify() {
    this.deps.forEach(dep => {
      queueJob(dep);
      dep.options && dep.options.schedular && dep.options.schedular();
    });
    // 放上面好像也没啥事情，丢里面也能跑。不知道为啥课上写外面，也没说为啥。
    // this.deps.forEach(dep => {
    //   dep.options && dep.options.schedular && dep.options.schedular();
    // })
  }
}

const ref = (initValue) => {
  let value = initValue;
  let deps = new Dep();

  return Object.defineProperty({}, "value", {
    get() {
      deps.depend();
      return value;
    },
    set(newVal) {
      value = newVal;
      deps.notify();
    }
  })
}

let computedCount = 0;
let computed = (fn) => {
  // 使用闭包来保存一个value
  let value;
  let dirty = true;
  // 使用effect包裹fn，增加option，以保证在执行完之后。能在广播中调用options里的schedular。
  let runner = effect(fn, {
    // 
    schedular: () => {
      // 执行这步的时候说明所依赖的值已经更新过了。所以需要置为true保证下面能计算。
      !dirty && (dirty = true);
    }
  });

  return {
    get value() {
      // 如果值没有发生变化，返回缓存中的值。
      // 也可以保证多处依赖的时候不会重复计算。
      if (dirty) {
        // 在get中响应式监听，直接在get中执行一次fn即可。
        value = runner();
        computedCount += 1;
        dirty = false;
      }
      return value;
    }
  }
}

let count = ref(0);

let computeCount = computed(() => count.value + 1)

document.getElementById('add').addEventListener('click', () => {
  count.value++;
})

watchEffect(() => {
  let str = `count is: ${count.value} \n computeCount is: ${computeCount.value} \n computedCount: ${computedCount}`;
  document.getElementById('app1').innerText = str;
})

watchEffect(() => {
  document.getElementById('app2').innerText = `computeCount 2 is : ${computeCount.value} \n computedCount: ${computedCount}`
})
```

细心的小伙伴发现了。点击add的时候下面的`id=app2`的`dom`不刷新了。这是为啥呢。

我们知道`watchEffect`会立即执行一遍。所以一开始的时候。我们的值是`computeCount.value`是1，没毛病。并且在设置缓存之后，`computedCount`也是1了。并没有重复计算。因为在执行`app2`的`dom`的渲染的时候，`count`的值并没有刷新，所以直接从缓存中取的值。所以上述代码的缓存功能是实现了。

至于为啥不更新。因为`app2`并不依赖于count的值，所以在依赖收集的时候，并没有被增加到`deps`里去。在页面渲染完毕之后，`deps`里只有两个`dep`，都是`app1`在渲染的时候产生的。一个是直接调用`get cout.value`的时候产生的依赖，还有一个是调用`computed`里的`count.value + 1`的时候使用的`get`产生的依赖。所以`count`更新的时候只会执行这两个。并不会触发`app2`的更新。

没搞懂的。理清一下逻辑。在`notify`的时候打个断点。在`depend`的时候打一个断点。就跑懂啦。

## watch

监听变化，并在监听回调函数中返回数据变更前后的两个值，常用于在数据变化之后执行的异步操作或者开销交大的操作。

那么，`app2`的不渲染问题也可以在这里解决。（不直接依赖`count`，但是需要根据`count`的变化而变化）

那么，核心思想就是，触发`count`的`get`方法把执行的回调函数增加到`deps`里。

我们这里watch的第一个参数只给到`function`，其他类型的不讨论。

所以有一个getter：

```javascript
// source是一个函数，里面有所依赖（要监听）的值
let getter = () => {
  return source();
};
```

然后我们需要一个增加依赖的地方，所以用到了effect，然后还需要给回调函数准备`newVal`和`oldVal`：

```js
let oldVal;
const runner = effect(getter, {
  schedular: () => {
    // 重复触发依赖收集
    let newVal = runner();
    // 只有当newVal不等于oldVal的时候才触发，即有变化之后才触发
    if (newVal !== oldVal) {
      cb(newVal, oldVal);
      // 重新赋值oldVal
      oldVal = newVal;
    }
  }
})
// 初始化赋值oldVal
oldVal = runner();
```

然后我们去使用watch，并且在callback里不直接使用`cout.value`，完整代码如下：

```js
let active;

let effect = (fn, options = {}) => {
  let effectInner = (...args) => {
    try {
      active = effectInner;
      return fn(...args);
    } finally {
      active = null;
    }
  }
  effectInner.options = options;

  return effectInner;
}

const watchEffect = (cb) => {
  let runner = effect(cb);
  runner();
}

let nextTick = cb => Promise.resolve().then(cb)

let queue = [];
let queueJob = job => {
  if (!queue.includes(job)) {
    queue.push(job);
    nextTick(flushJobs);
  }
}
let flushJobs = () => {
  let job;
  while ((job = queue.shift()) !== undefined) {
    job();
  }
}
class Dep {
  deps = new Set();
  depend() {
    if (active) {
      this.deps.add(active);
    }
  }
  notify() {
    this.deps.forEach(dep => {
      queueJob(dep);
      dep.options && dep.options.schedular && dep.options.schedular();
    });
    // this.deps.forEach(dep => {
    //   dep.options && dep.options.schedular && dep.options.schedular();
    // })
  }
}

const ref = (initValue) => {
  let value = initValue;
  let deps = new Dep();

  return Object.defineProperty({}, "value", {
    get() {
      deps.depend();
      return value;
    },
    set(newVal) {
      value = newVal;
      deps.notify();
    }
  })
}

let computed = (fn) => {
  // 使用闭包来保存一个value
  let value;
  let dirty = true;
  let runner = effect(fn, {
    schedular: () => {
      !dirty && (dirty = true);
    }
  });

  return {
    get value() {
      // 如果值没有发生变化，返回缓存中的值。
      // 也可以保证多处依赖的时候不会重复计算。
      if (dirty) {
        // 在get中响应式监听，直接在get中执行一次fn即可。
        value = runner();
        dirty = false;
      }
      return value;
    }
  }
}

let watch = (source, cb) => {
  let getter = () => {
    return source();
  };
  let oldVal;
  const runner = effect(getter, {
    schedular: () => {
      // 重复触发依赖收集
      let newVal = runner();
      if (newVal !== oldVal) {
        cb(newVal, oldVal);
        oldVal = newVal;
      }
    }
  })

  oldVal = runner();
}

let count = ref(0);

let computeCount = computed(() => count.value + 1)

watch(
  () => count.value,
  (newVal, preVal) => {
    console.log(newVal, preVal);
    document.getElementById('app2').innerText = `watchCount is : ${newVal}`
  }
)

document.getElementById('add').addEventListener('click', () => {
  count.value++;
})

watchEffect(() => {
  let str = `count is: ${count.value} \n computeCount is: ${computeCount.value}`;
  document.getElementById('app1').innerText = str;
})

// watchEffect(() => {
//   document.getElementById('app2').innerText = `computeCount 2 is : ${computeCount.value}`
// })
```

可以看到`app2`已经被更新了。

但是首次渲染的时候没有显示。当然咯。首次渲染的时候`count.value`又没变化咯。

要是我就是要初始化的时候也渲染呢？

那么就到了watch的options里，有个叫immediate的参数。用来判断是否需要立即执行一次。

对watch进行一下小修改：

```js
let watch = (source, cb, options = { immediate: false }) => {
  const { immediate } = options;
  let getter = () => {
    return source();
  };
  let oldVal;

  const applyCb = () => {
    // 重复触发依赖收集
    let newVal = runner();
    if (newVal !== oldVal) {
      cb(newVal, oldVal);
      oldVal = newVal;
    }
  }
  const runner = effect(getter, {
    // schedular: () => applyCb()
    schedular: applyCb
  })

  if (immediate) {
    applyCb();
  } else {
    oldVal = runner();
  }
}
```

这样的话第一次进来就可以看到啦。
