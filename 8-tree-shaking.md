# unused harmony exports

webpack 本身并不会删除任何多余的代码，删除无用代码的工作是 Uglify做的。webpack 做的事情是 `unused harmony exports`  的标记，也就是他会分析你的代码，把不用的 `exports` 删除，但是他不会删除全部的代码，只是把 `exports` 关键字删除了，变量的声明并没有删除。如果不信，可以自己尝试一下。

这里举个栗子：

```js

export const x = function () {
  console.log('x')
}

export function y () {
  console.log('y')
}
```

上面是一个模块，如果我们这样引用 `import { x } from './a.js'`，那么webpack会进行 `unused harmony exports` 分析，他会发现你的 `y` 函数根本没用到，于是把y的 `exports` 声明去掉了，留下了一个无用的 `y` 函数声明。于是输出这样的代码：

```js

/***/ "./src/a.js":
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
/* unused harmony export y */
const x = function () {
  console.log('x');
};
/* harmony export (immutable) */ __webpack_exports__["a"] = x;

function y() {
  console.log('y');
}

/***/ }),
```

可以看到， `y` 函数的 export 被去掉了，但是y函数本身并没有被删除。如果我们启用 `Uglify` 插件，由于y 函数没有被任何人使用，所以 Uglify 会把它直接删掉。


```js
function(e,t,n){"use strict";t.a=function(){console.log('x')}}])
```


这就是webpack 在`tree-shaking` 中扮演的角色：他会进行无用导出的分析，并把对应的 export 删除，但变量本身的声明并不会被删除。删除代码的操作是交给 uglify来做的。

但是，webpack 的这个特性仅限于 `ES6` 模块语法，如果你用了 nodejs 的模块，即 `module.exports`，那么webpack 将 不会做任何优化。所以如果你使用了 `babel`，请一定不要把 ES6 modules 编译掉，否则 tree-shaking 将没有效果。

为什么会这样呢，因为 `ES6 Modules` 是静态的，而 CMD 是动态的，这里借用youyuxi在知乎上的一段回答：

> 1. 只能作为模块顶层的语句出现，不能出现在 function 里面或是 if 里面。（ECMA-262 15.2)
> 2. import 的模块名只能是字符串常量。(ECMA-262 15.2.2)
> 3. 不管 import 的语句出现的位置在哪里，在模块初始化的时候所有的 import 都必须已经导入完成。换句话说，ES6 imports are hoisted。(ECMA-262 15.2.1.16.4 - 8.a)
> 4. import binding 是 immutable 的，类似 const。比如说你不能 import { a } from './a' 然后给 a 赋值个其他什么东西。
> 作者：尤雨溪
> 链接：https://www.zhihu.com/question/41922432/answer/93346223
> 来源：知乎
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

正因为这些特性，使得静态分析依赖变得很靠谱。否则如果无法静态分析，自然无法在编译阶段进行tree shaking。

# 函数副作用

这里涉及到另外一个问题，就是函数副作用。我们先定义一下函数副作用，就是一个函数引用了函数外部的变量（读或者写）。如果在 `x` 中引用了 `y` 怎么办？比如我们把代码改成：

```js
export const x = function () {
  y()
}

export function y () {
  console.log('y')
}
```

`x` 函数中调用了 `y` 函数，但是 `y` 函数本身不需要 `export`。所以webpack 依然会把 `export y` 中的 `export` 关键字去掉。变成这样：

```js

"use strict";
/* unused harmony export y */
const x = function () {
  console.log(y);
};
/* harmony export (immutable) */ __webpack_exports__["a"] = x;


function y() {
  console.log('y');
}
```

而 Uglify 进行压缩的时候，会发现 `y` 函数在 `x` 中被引用了，因此 `y` 函数并不会被删除：

```js
function(e,t,n){"use strict";function o(){console.log("y")}t.a=function(){console.log(o)}}])
```


***显而易见的是，webpack 通过静态语法分析，找出了不用的 `export` ，把他们改成 `free variable`，而 `Uglify`同样也通过静态语法分析，找出了不用的变量声明，直接把他们删了。***

但是函数副作用是非常复杂的，有没有可能被删除的 `y` 函数其实是有副作用的？肯定是存在的，比如我们在 `y` 中引用 一个对象的属性，然后设置 `getter` ，那么读取属性就会产生副作用：

```js
const obj = {z:"1"}

export const x = function () {
  console.log(obj.z)
}


Object.defineProperty(obj, "z", {
  get () {
    console.log('get')
  }
})

obj.z // 这一行代码会产生副作用
```

在 `uglify` 之前，上述代码会输出两次 `get` ，但是uglify之后，由于 `obj.z` 被删除了，所以只会输出一次 `get`。也就是 uglify 在处理getter的时候，可能会错误的删除代码导致逻辑出错。

这是 `uglify` 之后的代码：

```js
function(e,t,n){"use strict";const r={z:"1"};t.a=function(){console.log(r.z)},Object.defineProperty(r,"z",{get(){console.log("get")}})}]
```

可以看到，最后一行的 `obj.z` 直接被删除了。这显然是 `uglify` 的一个bug。

# webpack 中关于 tree-shaking 的代码

TODO
