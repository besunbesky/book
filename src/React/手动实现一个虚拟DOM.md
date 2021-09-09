手动实现一个简单的虚拟 `DOM` (类似 `Vue2.x` ， `React` 中的 `Fiber` 太过于复杂)

# 创建

我们来尝试简单渲染一个 `DOM` 结构，

```html
<div id='app'>
  <p>节点1</p>
</div>
```

我们有一个最简陋的 `createElement` 函数来返回一个虚拟 `DOM`

```js
const vnodeType = {
  HTML: 'HTML',
  TEXT: 'TEXT',

  COMPONENT: 'COMPONENT',
  CLASS_COMPONENT: 'CLASS_COMPONENT',
}

const childType = {
  EMPTY: 'EMPTY',
  SINGLE: 'SINGLE',
  MULTIPLE: 'MULTIPLE',
}


// 新建虚拟DOM
// 名字，属性，子元素
function createElement(tag, data, children = null) {

  let flag;
  if (typeof tag === 'string') {
    // 普通html标签
    flag = vnodeType.HTML
  } else if (typeof tag === 'function') {
    flag = vnodeType.COMPONENT
  } else {
    flag = vnodeType.TEXT
  }

  // 0， 1， n
  let childrenFlag;
  if (children == null) {
    childrenFlag = childType.EMPTY
  } else if (Array.isArray(children)) {
    let length = children.length
    if (length === 0) {
      childrenFlag = childType.EMPTY
    } else {
      childrenFlag = childType.MULTIPLE
    }
  } else {
    childrenFlag = childType.SINGLE
    children = createTextVnode(children + '')
  }


  // 返回vnode
  return {
    flag, //vnode类型
    tag, // 标签，div文本没有tag，组件就是函数
    data,
    children,
    childrenFlag
  }
}

//渲染
function render() {

}

// 创建文本类型 vnode
function createTextVnode(text) {
  return {
    flag: vnodeType.TEXT,
    tag: null,
    data: null,
    children: text,
    childrenFlag: childType.EMPTY
  }
}
```

页面上

```js
let div = createElement('div', { id: 'app' }, [
    createElement('p', {}, '节点1'),
  ]);
console.log(JSON.stringify(div, null, 2))
```

out

```json
{
  "flag": "HTML",
  "tag": "div",
  "data": {
    "id": "app"
  },
  "children": [
    {
      "flag": "HTML",
      "tag": "p",
      "data": {},
      "children": {
        "flag": "TEXT",
        "tag": null,
        "data": null,
        "children": "节点1",
        "childrenFlag": "EMPTY"
      },
      "childrenFlag": "SINGLE"
    }
  ],
  "childrenFlag": "MULTIPLE"
}
```

接下来我们将它渲染到页面上。

# 渲染

我们搞多一些 `p` 元素在页面上，并且调用 `render` 函数来渲染。

```html
<html lang="en">

<head>
  <meta charset="UTF-8" />
  <meta http-equiv="X-UA-Compatible" content="IE=edge" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Document</title>
  <style>
    .item-header {
      font-size: 30px;
      color: green;
    }
  </style>
</head>

<body>
  <div id='app'></div>
  <script src='./index.js'></script>
  <script>
    let vnode = createElement('div', { id: 'app' }, [
      createElement('p', { key: 'a', style: { color: 'blue' } }, '节点1'),
      createElement('p', { key: 'b', '@click': () => alert('xxx') }, '节点2'),
      createElement('p', { key: 'c', 'class': 'item-header' }, '节点3'),
      createElement('p', { key: 'd' }, '节点4'),
    ]);
    render(vnode, document.getElementById('app'))
    // console.log(JSON.stringify(div, null, 2))
  </script>
</body>

</html>
```

还有一些额外的工作（简易版本），需要注意以下的一些问题。

- 属性的处理。
- 首次渲染和 `diff`
- `el` 属性
- 递归渲染子元素

简易版的代码如下:

```js
const vnodeType = {
  HTML: 'HTML',
  TEXT: 'TEXT',

  COMPONENT: 'COMPONENT',
  CLASS_COMPONENT: 'CLASS_COMPONENT',
};

const childType = {
  // 节点没有子元素或者是空数组
  EMPTY: 'EMPTY',
  // 节点是文本元素
  SINGLE: 'SINGLE',
  // 节点有1或多个子元素
  MULTIPLE: 'MULTIPLE',
};

// 新建虚拟DOM
// 名字，属性，子元素
function createElement(tag, data, children = null) {
  let flag;
  if (typeof tag === 'string') {
    // 普通html标签
    flag = vnodeType.HTML;
  } else if (typeof tag === 'function') {
    flag = vnodeType.COMPONENT;
  } else {
    flag = vnodeType.TEXT;
  }

  // 0， 1， n
  let childrenFlag;
  if (children == null) {
    childrenFlag = childType.EMPTY;
  } else if (Array.isArray(children)) {
    let length = children.length;
    if (length === 0) {
      childrenFlag = childType.EMPTY;
    } else {
      childrenFlag = childType.MULTIPLE;
    }
  } else {
    childrenFlag = childType.SINGLE;
    // 文本元素直接给textVnode节点
    children = createTextVnode(children + '');
  }

  // 返回vnode
  return {
    flag, //vnode类型
    tag, // 标签，div文本没有tag，组件就是函数
    data,
    children,
    childrenFlag,
    el: null,
  };
}

//渲染
function render(vnode, container) {
  // 区分首次渲染和再次渲染
  // 首次渲染直接mount，再次渲染需要diff
  mount(vnode, container);
}

// 首次挂载元素
function mount(vnode, container) {
  let { flag } = vnode;
  // 区别对待HTML节点和Text节点
  if (flag === vnodeType.HTML) {
    mountElement(vnode, container);
  } else if (flag === vnodeType.TEXT) {
    mountText(vnode, container);
  }
}

function mountElement(vnode, container) {
  let dom = document.createElement(vnode.tag);
  // 存一下，以后都能拿到真实dom
  vnode.el = dom;
  let { data, children, childrenFlag } = vnode;

  // 挂载data属性
  if (data) {
    for (let key in data) {
      // 节点，名字，老值，新值
      patchData(dom, key, null, data[key]);
    }
  }

  // 根据子元素的不同类型来渲染子元素
  if (childrenFlag !== childType.EMPTY) {
    if (childrenFlag === childType.SINGLE) {
      mount(children, dom);
    } else if (childrenFlag == childType.MULTIPLE) {
      for (let i = 0; i < children.length; i++) {
        mount(children[i], dom);
      }
    }
  }

  // 挂载
  container.appendChild(dom);
}
function mountText(vnode, container) {
  let dom = document.createTextNode(vnode.children);
  vnode.el = dom;

  // 挂载
  container.appendChild(dom);
}

function patchData(el, key, pre, next) {
  switch (key) {
    // 处理style属性
    case 'style':
      for (let k in next) {
        el.style[k] = next[k];
      }
      break;
    // 处理class属性
    case 'class':
      el.className = next;
      break;
    // 其他
    default:
      // 处理事件绑定函数，以vue的@为例
      if (key[0] === '@') {
        if (next) {
          el.addEventListener(key.slice(1), next);
        }
      } else {
        el.setAttribute(key, next)
      }
      break;
  }
}

// 创建文本类型 vnode
function createTextVnode(text) {
  return {
    flag: vnodeType.TEXT,
    tag: null,
    data: null,
    children: text,
    childrenFlag: childType.EMPTY,
    el: null,
  };
}
```

至此，虚拟 `DOM` 已经首次渲染到页面上了。

然后我们再来看看如何简单实现 `DOM diff`。

# patch

假设我们要将

```js
let vnode = createElement('div', { id: 'app' }, [
  createElement('p', { key: 'a', style: { color: 'blue' } }, '节点1'),
  createElement('p', { key: 'b', '@click': () => alert('xxx') }, '节点2'),
  createElement('p', { key: 'c', 'class': 'item-header' }, '节点3'),
  createElement('p', { key: 'd' }, '节点4'),
]);
```

里渲染的 `DOM` ，变更为如下的 `DOM` 结构，然后在一秒后重新渲染：

```js
let vnode1 = createElement("div", { id: "app" }, [
  createElement("p", { key: 'd' }, "节点4"),
  createElement("p", { key: 'a', style: { color: 'blue' } }, "节点1"),
  createElement("p", { key: 'b', }, "节点2"),
  createElement("p", { key: 'e' }, "节点5"),
  createElement("p", { key: 'f', style: { color: '#eee' } }, "节点4"),
]);

setTimeout(() => {
  render(vnode1, document.getElementById('app'))
})
```

然后我们需要在 `render` 函数里区分首次渲染和再次渲染：

```js
//渲染
function render(vnode, container) {
  // 区分首次渲染和再次渲染
  // 首次渲染直接mount，再次渲染需要diff
  if (container.vnode) {
    // 更新
    patch(container.vnode, vnode, container);
  } else {
    mount(vnode, container);
  }

  container.vnode = vnode;
}
```

然后我们看一下 `patch` 函数：

```js
function patch(pre, next, container) {
  let nextFlag = next.flag;
  let preFlag = pre.flag;

  // 如果flag不同直接替换。
  if (nextFlag !== preFlag) {
    // 直接替换
    repaceVnode(pre, next, container);
  } else if (nextFlag == vnodeType.HTML) {
    patchElement(pre, next, container);
  } else if (nextFlag == vnodeType.TEXT) {
    // 文本节点只需要更新文字内容即可
    patchText(pre, next);
  }
}

// 替换节点，先移除再mount
function replaceVnode(pre, next) {
  container.removeChild(pre.el);
  mount(next, container);
}

// 文本节点直接替换文字即可
function patchText(pre, next) {
  let el = (next.el = pre.el);
  if (next.children !== pre.children) {
    el.nodeValue = next.children;
  }
}
```

以上两种最简单的对比都是比较好理解的。接下来来看一下 `flag` 不同的 `HTML` 节点的替换 `patchElement` 。

```js
function patchElement(pre, next, container) {
  // 如果tag不同就直接替换掉
  if (pre.tag !== next.tag) {
    repaceVnode(pre, next, container);
    return;
  }

  // 更新一下 el， 然后更新data
  let el = (next.el = pre.el);
  let preData = pre.data;
  let nextData = next.data;
  // 如果有新值，则全部更新到el上
  if (nextData) {
    for (let key in nextData) {
      let preVal = preData[key];
      let nextVal = nextData[key];
      patchData(el, key, preVal, nextVal);
    }
  }
  // 对旧值进行处理，
  // 旧的有，新的没有，就要置为空
  // 旧的有，新的有的已经在上面一个循环里被覆盖掉了。
  if (preData) {
    for (let key in preData) {
      let preVal = preData[key];
      if (preVal && !nextData.hasOwnProperty(key)) {
        patchData(el, key, preVal, null);
      }
    }
  }

  // data更新完毕 下面更新子元素
  patchChildren(
    pre.childrenFlag,
    next.childrenFlag,
    pre.children,
    next.children,
    el
  );
}

// 更新子元素的方法
function patchChildren(
  preChildFlag,
  nextChildFlag,
  preChildren,
  nextChildren,
  container
) {
  // 新老元素都有三种情况，用switch case做嵌套处理
  // 更新子元素
  // 老的是 1 ， 0， n
  // 新的是 1， 0 ， n
  switch (preChildFlag) {
    // 老的是一个
    case childType.SINGLE:
      switch (nextChildFlag) {
        // 新的也是一个，直接patch
        case childType.SINGLE:
          patch(preChildren, nextChildren, container)
          break;
        // 新的是空的，直接移除老的
        case childType.EMPTY:
          container.removeChild(preChildren.el)
          break;
        // 新的是多个的，先移除老的，再循环mount新的
        case childType.MULTIPLE:
          container.removeChild(preChildren.el)
          for (let i = 0; i < nextChildren.length; i++) {
            mount(nextChildren[i], container)
          }
          break;
      }
      break;
    // 老的是空的
    case childType.EMPTY:
      switch (nextChildFlag) {
        // 新的是一个，直接mount
        case childType.SINGLE:
          mount(nextChildren, container)
          break;
        // 新的也是空的，不做处理
        case childType.EMPTY:
          break;
        // 新的是多个，直接循环mount新的
        case childType.MULTIPLE:
          for (let i = 0; i < nextChildren.length; i++) {
            mount(nextChildren[i], container)
          }
          break;
      }
      break;
    // 老的是多个
    case childType.MULTIPLE:
      switch (nextChildFlag) {
        // 新的是一个，循环移除老的，再把新的mount上去
        case childType.SINGLE:
          for (let i = 0; i < preChildren.length; i++) {
            container.removeChild(preChildren[i]);
          }
          mount(nextChildren, container)
          break;
        // 新的是空的，循环移除老的，接下来无操作
        case childType.EMPTY:
          for (let i = 0; i < preChildren.length; i++) {
            container.removeChild(preChildren[i]);
          }
          break;
        // 新的是多个的情况，比较复杂，React和Vue的实现不同。这里简单实现一下。
        // 这个算法网上都有讲解，就不赘述了。
        default:
          let lastIndex = 0
          for (let i = 0; i < nextChildren.length; i++) {
            const nextVNode = nextChildren[i]
            let j = 0,
              find = false
            for (j; j < preChildren.length; j++) {
              const prevVNode = preChildren[j]
              if (nextVNode.key === prevVNode.key) {
                find = true
                patch(prevVNode, nextVNode, container)
                if (j < lastIndex) {
                  // 需要移动
                  const refNode = nextChildren[i - 1].el.nextSibling
                  container.insertBefore(prevVNode.el, refNode)
                  break
                } else {
                  // 更新 lastIndex
                  lastIndex = j
                }
              }
            }
            if (!find) {
              // 挂载新节点
              const refNode =
                i - 1 < 0
                  ? preChildren[0].el
                  : nextChildren[i - 1].el.nextSibling

              mount(nextVNode, container, refNode)
            }
          }
          // 移除已经不存在的节点
          for (let i = 0; i < preChildren.length; i++) {
            const prevVNode = preChildren[i]
            const has = nextChildren.find(
              nextVNode => nextVNode.key === prevVNode.key
            )
            if (!has) {
              // 移除
              container.removeChild(prevVNode.el)
            }
          }
          break;
      }
      break;
  }
}
```

至此，一次 `DOM` 更新就实现了。
