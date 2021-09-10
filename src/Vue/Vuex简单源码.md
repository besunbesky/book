先把之前写过的vue的响应式原理里的代码略加修改搬过来

```javascript
/* eslint-disable no-unused-vars */
// let x;
// let y;
// let f = n => n * 100 + 100;

let active;

let watch = function (cb) {
  active = cb;
  active();
  active = null;
};

let queue = [];
let nextTick = cb => Promise.resolve().then(cb);
let queueJob = job => {
  if (!queue.includes(job)) {
    queue.push(job);
    nextTick(flushJobs);
  }
};
let flushJobs = () => {
  let job;
  while ((job = queue.shift()) !== undefined) {
    job();
  }
};

class Dep {
  constructor() {
    this.deps = new Set();
  }
  depend() {
    if (active) {
      this.deps.add(active);
    }
  }
  notify() {
    this.deps.forEach(dep => queueJob(dep));
  }
}

let ref = initValue => {
  let value = initValue;
  let dep = new Dep();

  return Object.defineProperty({}, "value", {
    get() {
      dep.depend();
      return value;
    },
    set(newValue) {
      value = newValue;
      dep.notify();
    }
  });
};

let createReactive = (target, prop, value) => {
  let dep = new Dep();

  // return new Proxy(target, {
  //   get(target, prop) {
  //     dep.depend();
  //     return Reflect.get(target, prop);
  //   },
  //   set(target, prop, value) {
  //     Reflect.set(target, prop, value);
  //     dep.notify();
  //   },
  // });

  return Object.defineProperty(target, prop, {
    get() {
      dep.depend();
      return value;
    },
    set(newValue) {
      value = newValue;
      dep.notify();
    }
  });
};

export let reactive = obj => {
  let dep = new Dep();

  Object.keys(obj).forEach(key => {
    let value = obj[key];
    createReactive(obj, key, value);
  });

  return obj;
};

// let data = reacitve({
//   count: 0
// });

import { Store } from "./vuex";

let store = new Store({
  state: {
    count: 0
  },
  mutations: {
    addCount(state, payload) {
      state.count += payload || 1;
    }
  },
  plugins: [
    store =>
      store.subscribe((mutation, state) => {
        console.log(mutation);
      })
  ]
});

document.getElementById("add").addEventListener("click", function () {
  // data.count++;
  store.commit("addCount", 1);
});
let str;
watch(() => {
  str = `hello ${store.state.count}`;
  document.getElementById("app").innerText = str;
});
```

vuex其实也就是在构造过程中调用了vue的reactive来包裹自己的state来使得state变为响应式的。

```javascript
import { reactive } from './myVue';

export class Store {

  constructor(options = {}) {
    let { state, mutations, plugins } = options;
    this._vm = reactive(state);
    this._mutations = mutations;

    this._subscribe = [];
    plugins.forEach(plugin => plugin(this));
  }

  get state() {
    return this._vm;
  }

  commit(type, payload) {
    const entry = this._mutations[type];
    if (!entry) {
      return;
    }
    entry(this.state, payload);
    this._subscribe.forEach(fn => fn({ type, payload }, this.state));
  }

  subscribe(fn) {
    if (!this._subscribe.includes(fn)) {
      this._subscribe.push(fn);
    }
  }

}
```
