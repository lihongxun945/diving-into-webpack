# 写一个自己的babel-loader

# 写一个简单的 babel-loader

这里所有的代码都在github上，地址 [https://github.com/lihongxun945/my-babel-loader](https://github.com/lihongxun945/my-babel-loader)。

这里以 `babel-loader` 为例，看我们如何写一个自己的loader。首先，我们参考这篇[官方教程](https://webpack.github.io/docs/how-to-write-a-loader.html)，虽然写的很粗略，但是我们可以学会写一个简单的loader。

最简单的loader是一个什么都不做，原样返回JS代码的 loader，像这样：

```js
module.exports = function (source) {
  return source
}
```
但是我们的 `babel-loader` 显然需要调用 `babel` 来编译代码，我们查一下 [babel-core](https://babeljs.io/docs/core-packages/) 文档，可以调用 `babel.transform` API来编译代码。再加上一些 `presets` 的设置，我们可以把上面的代码做一下改造如下：

```js
var babel = require("babel-core")
module.exports = function (source) {
  var babelOptions = {
    presets: ['env']
  }
  var result = babel.transform(source, babelOptions)
  return result.code
}
```

关于 Babel 的API用法不在这里详细解释，有兴趣的可以直接去看官方的文档。我们这里做了一个很简单的转换，就是把 接收到的 `source` 源码，用 `babel` 编译一下，然后返回编译后的代码。

那么问题来了，我们要如何指定使用自己的loader呢，可以参考下面这种写法，直接在 `webpack` config 文件里面加一个 `resolveLoader` 的配置即可，我们这里把 `bable-loader` 指定为自己写的。

```js
  resolveLoader: {
    alias: {
      "babel-loader": resolve('./build/babel-loader.js')
    }
  },
```

然后我们就可以运行 `npm run dev` 编译自己的JS代码，比如我写了这么几行代码：

```js
class People {
  constructor (name) {
    this.name = name
  }

  sayName () {
    console.log(`Hello there, I'm ${this.name}`)
  }
}

const lily = new People('Lily')
lily.sayName()
```

经过我们的 `babel-loader` 编译后，最终输出的代码是这样的：

```js
var People = function () {
  function People(name) {
    _classCallCheck(this, People);

    this.name = name;
  }

  _createClass(People, [{
    key: 'sayName',
    value: function sayName() {
      console.log('Hello there, I\'m ' + this.name);
    }
  }]);

  return People;
}();

var lily = new People('Lily');
lily.sayName();
```

可以看到其中 `class` , 字符串模板，`const` 等都被 babel 编译过。

# 添加sourcemap

上面的简单代码并没有实现 sourcemap 功能，如果需要支持 sourcemap，显然需要把 `babel-core` 产生的 `sourcemap` 传给 `webpack`。之前因为只返回编译后的代码，所以我们直接返回了字符串，如果需要同时返回编译的代码和sourcemap，我们需要这个接口 [this.callback](https://webpack.js.org/api/loaders/#this-callback)，官方文档上是这么说的：

```js
this.callback(
  err: Error | null,
  content: string | Buffer,
  sourceMap?: SourceMap,
  meta?: any
);
```
那么我们查一下babel的文档之后，很简单就能获取 `sourcemap` 了，其实就是 `result.map`，改动后的代码如下：

```js
var babel = require("babel-core")

module.exports = function (source, inputSourceMap) {
  var babelOptions = {
    presets: ['env'],
    inputSourceMap: inputSourceMap,
    sourceMaps: true
  }
  var result = babel.transform(source, babelOptions)
  this.callback(null, result.code, result.map)
}
```

这里 `sourceMaps: true` 是告诉 `babel` 要生成 `sourcemap`，因为默认情况下它是不会生成的。然后刷新页面应该就能看到sourcemap了。

然而，根据你的webpack配置不同，很可能并看不到~~~这是因为 sourcemap 其实是要交给 webpack 来管理的，我们只是把 `sourcemap` 传给了 webpack，而它在最终编译出的代码中到底要不要显示是 webpack 自己的配置决定的。那么我们在 `webpack.config.js` 中添加一行代码打开sourcemap功能即可：

```js
devtool: 'eval-source-map'
```
这样再刷新页面就能看到sourcemap了，确实可以，然而 sourcemap 的文件名怎么是 `unknown`？ 我们在创建 sourcemap的时候需要指定一下文件名，不然确实会出现 unknown 的问题。

那么问题又来了，怎么获取文件名呢？

可以通过 [this.request](https://webpack.js.org/api/loaders/#this-request) 来获取当前的文件名，比如这个示例中的  `this.request` 是：

```
~/github/my-webpack-loader/build/babel-loader.js!~/github/my-webpack-loader/src/main.js
```
**其中 `~` 是被我省略的绝对路径**

可以看出 `this.request` 就是一个加载文件的请求，包含了两部分，通过 `!` 分割，前一部分是 对应的 `loader`，后一部分是文件的路径。我们一行代码就可以把文件名提取出来：

```js
this.request.split('!')[1].split('/').pop()
```

然后在 `babel.transform` 的配置中增加一行：

```js
filename: this.request.split('!')[1].split('/').pop()
```

再刷新页面就可以看到正常的 sourcemap 了。

# 模块

我们现在已经支持 编译代码 和 sourcemap，下面我们要支持 modules，我们把 `main.js` 代码拆成两部分：

*people.js*

```js
export default class People {
  constructor (name) {
    this.name = name
  }

  sayName () {
    console.log(`Hello there, I'm ${this.name}`)
  }
}
```

*main.js*

```js
import People from './people'

const lily = new People('Lily')
lily.sayName()
```

那么我们需要怎么做处理呢？ 对JS文件来说，我们最好的方式是不作任何特殊处理。上面的代码其实已经可以正常打包模块了。那么是怎么做到的呢？

因为JS在`webpack`中是一等公民，webpack把所有的资源都当做JS来加载。webpack默认支持常见的AMD,CMD,ES6,nodejs 等常见的模块加载方式，并且会自动做 bundle（打包）和 `tree-shaking`。反而，babel 只是把 ES6 模块编译成了 nodejs 模块，它并不会做bundle。

比如我们的 `people.js`，经过 `babel-loader` 编译出来的代码是这样的：

```js
// 省略几个工具方法。。。
var People = function () {
  function People(name) {
    _classCallCheck(this, People);

    this.name = name;
  }

  _createClass(People, [{
    key: "sayName",
    value: function sayName() {
      console.log("Hello there, I'm " + this.name);
    }
  }]);

  return People;
}();

exports.default = People;
```

webpack 会识别 `exports.default` 语法，并且最终把两个文件合并一个大的文件。反而，如果我们在 `babel-loader` 中做了打包，会导致 `webpack` 无法做 `tree-shaking` 优化。

这也是webpack 的各种 JS loader，如 `vue-loader`, `jsx-loader` 等的共同做法，即把 modules bundle 交给 webpack 处理，让webpack做最大程度的优化。

当然，这种做法仅仅对 JS 有效，因为JS是webpack的一等公民，webpack内置了对JS 模块的完整支持，而其他的文件，比如 `css`, `html` 等，我们都需要把他们转成JS然后交给webpack。这也是为什么 webpack 中有 `style-loader`, `url-loader` 却没有一个 `js-loader` 的原因。

我们的 `babel-loader` 代码就已经写完了。其实总共就10行代码，最终完整代码如下所示：

```js
var babel = require("babel-core")

module.exports = function (source, inputSourceMap) {
  var babelOptions = {
    presets: ['env'],
    inputSourceMap: inputSourceMap,
    filename: this.request.split('!')[1].split('/').pop(),
    sourceMaps: true
  }
  var result = babel.transform(source, babelOptions)
  this.callback(null, result.code, result.map)
}
```

那么我们去看一下官方的 [babel-loader](https://github.com/babel/babel-loader/blob/master/src/index.js) 是如何实现的。显然他的代码比我们的代码多很多，他写了那么多，其实主要是增加了 Cache，以及增加了对异常的处理。有兴趣的可以自己去研究一下。

关于webpack的模块加载机制，我们后续再详细解读。

# 如何编译 JSX

如果我们是使用React写的组件，那么同样可以通过 `babel-loader` 来编译。关于如何编译JSX，babel官网这里做了很详细的文档 [transform-react-jsx](https://babeljs.io/docs/plugins/transform-react-jsx/)

简单来说，就是 `babel-core` 本身虽然不支持 `jsx`，会报语法错误，但是我们可以通过加载一个插件就能支持 jsx用法如下：

```js
require("babel-core").transform("code", {
  plugins: ["transform-react-jsx"]
});
```

那么我们只要在 上面我们写的 `bable-loader` 的代码中加入一行 `plugins: ["transform-react-jsx"]` 就可以支持 jsx 的编译了，是不是很简单。

react 因为用的JSX，而jsx 因为全部编译成了JS 所以它的loader很简单。但是 Vue 的组件并不是可以直接就全部编译成JS，而是包括了 html,JS,CSS三部分，所以 `vue-loader` 相对来说就复杂很多了。
