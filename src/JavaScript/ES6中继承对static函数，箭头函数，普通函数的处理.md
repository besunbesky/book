# 问题引入

最近一直在看原型继承相关的东西，翻到这么一篇文章：
[从ES6中的extends讲js原型链与继承](http://wulv.site/2017-05-29/从ES6中的extends讲js原型链与继承.html)

文中有一个点让我很感兴趣，箭头函数在继承过程中无法通过super关键字获取，这是为什么呢？

# 前置知识

## MDN上关于super的介绍

[The super keyword is used to access and call functions on an object's parent - in
MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super) 大概有这么几个关键点：

1. 子类中存在constructor方法的时候，需要调用super方法，并且需要在使用this关键字之前调用
2. super关键字可以用来调用父对象上的方法
3. 可以使用super来调用父对象上的静态方法
4. 不可以使用delete来删除super上的属性
5. 不可以复写super对象上的只读属性

## 子类中是否必须主动调用super方法？

我的看法是不需要。 网上有些文章（[比如这篇](https://segmentfault.com/a/1190000008165717)）写道：

> 因为若不执行super，则this无法初始化。

我的个人理解是，this是指代执行上下文环境的，不存在无法初始化的情况。更准确的说法是这样：如果不使用super方法，那么父类中的属性值无法进行初始化，如果这个时候子类通过this字段来访问了父类中的属性值，那么只能得到一个undefined。至于为什么这么写编译的时候会报错？我的理解是，这应该是一种语法错误，而且是一种规范要求，ES6语法的规范要求，这种要求并不是说会影响到代码的实际执行。举个栗子：

```
// typescript中一段简单的继承代码实现
class Parent {
    name = 'parent';
    func = function() {
        console.log('func in parent called.');
    }
}

class Child extends Parent {
    age = 3;
    func = function() {
        console.log('age is: ', this.age);    // 使用了this，不会报错
    }
}
```

这段代码非常简单，在子类中使用了this关键字，编译时不会报错，也可以正常执行。然后我们进行一点修改，在子类中引入constructor方法

```
class Child extends Parent {
    age = 3;
    // error TS2377: Constructors for derived classes must contain a 'super' call.
    constructor() {
    
    }
    func = function() {
        console.log('age is: ', this.age);
    }
}
```

可以看到，编译阶段已经开始报错了。在typescript的语法中，子类的constructor方法中不但需要调用super方法，而且必须在第一行代码就调用super，否则都是会报错的。看下面这段代码:

```
class Child extends Parent {
    age = 3;
    constructor() {
        console.log('First line in constructor without super method');
        super();    // error TS2376: A 'super' call must be the first statement in the constructor when a class contains initialized properties or has parameter properties.
    }
    func = function() {
        console.log('age is: ', this.age);
    }
}
```

来，我们接着改

```
class Parent {
    name = 'parent';
    func = function() {
        console.log('func in parent called.');
    }
}

class Child extends Parent {
    age = 3;
    constructor() {
        console.log('Show property of parent, name is: ', this.name);    // error TS17009: 'super' must be called before accessing 'this' in the constructor of a derived class.
        console.log('Show property of child, age is: ', this.age);        // error TS17009: 'super' must be called before accessing 'this' in the constructor of a derived class.
        super();    // error TS2376: A 'super' call must be the first statement in the constructor when a class contains initialized properties or has parameter properties.
        console.log('Show property of parent, name is: ', this.name);
        console.log('Show property of child, age is: ', this.age);
    }
    func = function() {
        console.log('age is: ', this.age);
    }
}
```

可以看到，编译期已经开始报各种错误了，不过这不重要，我们这里利用typescript的编译器（tsc）来进行编译，并查看编译后的代码内容：

```
var __extends = (this && this.__extends) || (function () {
    var extendStatics = Object.setPrototypeOf ||
        ({ __proto__: [] } instanceof Array && function (d, b) { d.__proto__ = b; }) ||
        function (d, b) { for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p]; };
    return function (d, b) {
        extendStatics(d, b);
        function __() { this.constructor = d; }
        d.prototype = b === null ? Object.create(b) : (__.prototype = b.prototype, new __());
    };
})();
var Parent = (function () {
    function Parent() {
        this.name = 'parent';
        this.func = function () {
            console.log('func in parent called.');
        };
    }
    return Parent;
}());
var Child = (function (_super) {
    __extends(Child, _super);
    function Child() {
        var _this = this;
        _this.age = 3;
        _this.func = function () {
            console.log('age is: ', this.age);
        };
        console.log('Show property of parent, name is: ', _this.name); // 输出undefined，因为此时子类的实例上还没有继承到父类的属性值
        console.log('Show property of child, age is: ', _this.age); // 输出3，子类实例自己的属性值可以访问
        _this = _super.call(this) || this; // 构造函数式的继承实现，这一步就是讲父类的属性值设置到子类实例上
        console.log('Show property of parent, name is: ', _this.name); // 输出parent，此时子类的实例上经过上一步的继承，得到了父类的属性值
        console.log('Show property of child, age is: ', _this.age);  // 输出3，子类实例自己的属性值可以访问
        return _this;
    }
    return Child;
}(Parent));
//# sourceMappingURL=demo.js.map
```

由此可以知道，在ES6中使用extends进行继承操作的过程中，

- 子类并非必须调用super方法，除非存在constructor方法
- 在constructor方法中应该首先调用super方法，这是语法要求，不过这不是必须的
- 在调用super方法之前，将无法通过this关键字来访问父类的属性（这里就可以解释其他文章中提到的
  ‘若不执行super，则this无法初始化’，更准确的说法应该是‘若不执行super，则无法将父类的属性值初始化到当前子类实例上’）

## 子类中使用super.prop和super[expr]的方式是如何访问父类的属性和方法？

我们直接来看代码吧,关键点都注释了的

```
class Parent {
    public name = 'parent';
    public static staticName = 'staticParent';
    public static staticFunc() {
        console.log('staticFunc called in parent.');
    }

    public arrowFunc = () => {
        console.log('arrowFunc called in parent.');
    }

    public normalFunc() {
        console.log('normalFunc called in parent.')
    }
}

class Child extends Parent {
    public static staticFunc() {
        super.staticFunc();
        console.log('staticFunc called in Child.');
    }

    arrowFunc = () => {
        super.arrowFunc();
        console.log('arrowFunc called in Child.');
    }

    normalFunc() {
        super.normalFunc();
        console.log('normalFunc called in Child.')
    }

    getName() {
        console.log('parent name is: ', super.name);
        console.log('parent staticName is: ', super.staticName);
        console.log('child name is: ', this.name);
    }
}

/** 编译后的代码 **/
var __extends = (this && this.__extends) || (function () {
    var extendStatics = Object.setPrototypeOf ||
        ({ __proto__: [] } instanceof Array && function (d, b) { d.__proto__ = b; }) ||
        function (d, b) { for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p]; };
    return function (d, b) {
        extendStatics(d, b);
        function __() { this.constructor = d; }
        d.prototype = b === null ? Object.create(b) : (__.prototype = b.prototype, new __());
    };
})();
var Parent = (function () {
    function Parent() {
        this.name = 'parent';
        this.arrowFunc = function () {
            console.log('arrowFunc called in parent.');
        };
    }
    // 编译后的静态方法可以存在于Parent类的内部
    Parent.staticFunc = function () {
        console.log('staticFunc called in parent.');
    };
    Parent.prototype.normalFunc = function () {
        console.log('normalFunc called in parent.');
    };
    return Parent;
}());
Parent.staticName = 'staticParent'; // 编译后的静态属性依然存在于Parent类外
var Child = (function (_super) {
    __extends(Child, _super);
    function Child() {
        var _this = _super !== null && _super.apply(this, arguments) || this;
        _this.arrowFunc = function () { // 子类实例调用arrowFunc的时候会报错，因为_super.prototype上是不存在arrowFunc方法的
            _super.prototype.arrowFunc.call(_this); // Uncaught TypeError: Cannot read property 'call' of undefined
            console.log('arrowFunc called in Child.');
        };
        return _this;
    }
    Child.staticFunc = function () {
        _super.staticFunc.call(this);   // super可以正常访问父类的静态方法
        console.log('staticFunc called in Child.');
    };
    Child.prototype.normalFunc = function () {
        _super.prototype.normalFunc.call(this);
        console.log('normalFunc called in Child.');
    };
    Child.prototype.getName = function () {
        console.log('parent name is: ', _super.prototype.name); // 输出undefined， 父类原型（_super.prototype）上不存在name属性
        console.log('parent staticName is: ', _super.prototype.staticName); // 输出undefined，super无法正常访问父类的静态属性
        console.log('child name is: ', this.name);  // 输出parent，这是子类实例上的属性，继承自父类
    };
    return Child;
}(Parent));
//# sourceMappingURL=demo.js.map
```

这里再顺嘴提一句，关于静态属性和静态方法的区别。为什么在子类中通过super关键字来获取父类的静态方法经过编译后是_super.staticFunc，而获取静态属性依然是_super.prototype.staticName，从原型上获取导致获取失败呢？这个问题目前我还没有找到答案，希望有知道的小伙伴可以不吝指教。
不过我倒是搜到一些其他相关内容。 [Class 的静态属性和实例属性](http://es6.ruanyifeng.com/#docs/class)

> 因为 ES6 明确规定，Class 内部只有静态方法，没有静态属性。

虽然这种规定从ES7开始得到了修正，我们目前已经可以将静态属性写在Class的内部，但是经过编译之后可以发现，静态属性依然存在于类的实现的外部。

```
var Parent = (function () {
    function Parent() {
        this.name = 'parent';
        this.arrowFunc = function () {
            console.log('arrowFunc called in parent.');
        };
    }
    // 编译后的静态方法可以存在于Parent类的内部
    Parent.staticFunc = function () {
        console.log('staticFunc called in parent.');
    };
    Parent.prototype.normalFunc = function () {
        console.log('normalFunc called in parent.');
    };
    return Parent;
}());
Parent.staticName = 'staticParent'; // 编译后的静态属性依然存在于Parent类外
```

# 回到问题本身

问：箭头函数在继承过程中无法通过super关键字获取，这是为什么呢？ 答：因为子类中使用super.prop和super[expr]的方式获取的是父类原型（prototype）上的方法，静态方法除外。

# 参考资料

[从ES6中的extends讲js原型链与继承](http://wulv.site/2017-05-29/从ES6中的extends讲js原型链与继承.html)
[React ES6 class constructor super()](https://segmentfault.com/a/1190000008165717)
[Class 的静态属性和实例属性](http://es6.ruanyifeng.com/#docs/class)

[备份自](https://blog.csdn.net/qq_32442967/article/details/100086200?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-2.not_use_machine_learn_pai&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-2.not_use_machine_learn_pai)
