# webpack 源码解析 四：file-loader 和 url-loader

file-loader 和 url-loader 相对简单一些，如果没有看过代码可能一下想不到 `file-loader` 是如何工作的。其实他们都依靠 webpack 提供的强大的API，自己本身并没有做多少工作，完全不用担心读写文件的问题，因为这些webpack已经帮你封装好了。

# file-loader

`file-loader` 并不会对文件内容进行任何转换，只是复制一份文件内容，并根据配置为他生成一个唯一的文件名。

`file-loader` 的工作流程如下：

1. 通过 `loaderUtils.interpolateName` 方法可以根据 `options.name` 以及文件内容生成一个唯一的文件名 url（一般配置都会带上hash，否则很可能由于文件重名而冲突）
2. 通过 `this.emitFile(url, content)` 告诉 webpack 我需要创建一个文件，webpack会根据参数创建对应的文件，放在 public path 目录下。
3. 返回 `'module.exports = __webpack_public_path__ + '+ JSON.stringify(url) + ‘;’` ，这样就会把原来的文件路径替换为编译后的路径

对`file-loader` 中最重要的几行代码解释如下（我们自己来实现一个 `file-loader` 就只需要这几行代码就行了，完全可以正常运行且支持 `name` 配置)：

```js
var loaderUtils = require('loader-utils')

module.exports = function (content) {

  // 获取options，就是 webpack 中对 file-loader 的配置，比如这里我们配置的是 `name=[name]_[hash].[ext]`
  // 获取到的就是这样一个k-v 对象 { name: "[name]_[hash].[ext]" }
  const options = loaderUtils.getOptions(this) || {};

  // 这是 loaderUtils 的一个方法，可以根据 name 配置和 content 内容 生成一个文件名。为什么需要 文件内容呢？这是为了保证当文件内容没有发生变化的时候，名字中的 [hash] 字段也不会变。可以理解为用文件的内容作了一个hash
  let url = loaderUtils.interpolateName(this, options.name, {
    content
  })

  this.emitFile(url, content) // 告诉webpack，我要创建一个文件，文件名和内容，这样webpack就会帮你在 dist 目录下创建一个对应的文件

  // 这里要用到一个变量，就是 __webpack_public_path__ ，这是一个由webpack提供的全局变量，是public的根路径
  // 参见：https://webpack.js.org/guides/public-path/#on-the-fly
  // 这里要注意一点：这个返回的字符串是一段JS，显然，他是在浏览器中运行的
  // 举个栗子：
  // css源码这样写： background-image: url('a.png')
  // 编译后变成: background-image: require('xxxxxx.png')
  // 这里的 require 语句返回的结果，就是下面的 exports 的字符串，也就是图片的路径
  return 'module.exports = __webpack_public_path__ + '+ JSON.stringify(url)
}

// 一定别忘了这个，因为默认情况下 webpack 会把文件内容当做UTF8字符串处理，而我们的文件是二进制的，当做UTF8会导致图片格式错误。
// 因此我们需要指定webpack用 raw-loader 来加载文件的内容，而不是当做 UTF8 字符串传给我们
// 参见： https://webpack.github.io/docs/loaders.html#raw-loader
module.exports.raw = true
```

如果我们不支持 `name` 参数，甚至只需要两行代码就能实现。

# url-loader

`url-loader` 并不是复制文件，而是把文件做base64编码，直接嵌入到CSS/JS/HTML代码中。

`url-loader` 的工作流程如下：

1. 获取 limit 参数
2. 如果 文件大小在 limit 之类，则直接返回文件的 base64 编码后内容
3. 如果超过了 limit ，则调用 `file-loader

因为逻辑比较简单，这里直接放上源码以及我添加的注释：

```js
module.exports = function(content) {

    // 获取 options 配置，上面已经讲过了就不在重复
  var options =  loaderUtils.getOptions(this) || {};
  // Options `dataUrlLimit` is backward compatibility with first loader versions
    // limit 参数，只有文件大小小于这个数值的时候我们才进行base64编码，否则将直接调用 file-loader
  var limit = options.limit || (this.options && this.options.url && this.options.url.dataUrlLimit);

  if(limit) {
    limit = parseInt(limit, 10);
  }

  var mimetype = options.mimetype || options.minetype || mime.lookup(this.resourcePath);

  // No limits or limit more than content length
  if(!limit || content.length < limit) {
    if(typeof content === "string") {
      content = new Buffer(content);
    }

        // 直接返回 base64 编码的内容
    return "module.exports = " + JSON.stringify("data:" + (mimetype ? mimetype + ";" : "") + "base64," + content.toString("base64"));
  }

    // 超过了文件大小限制，那么我们将直接调用 file-loader 来加载
  var fallback = options.fallback || "file-loader";
  var fallbackLoader = require(fallback);

  return fallbackLoader.call(this, content);
}

// 一定别忘了这个，因为默认情况下 webpack 会把文件内容当做UTF8字符串处理，而我们的文件是二进制的，当做UTF8会导致图片格式错误。
// 因此我们需要指定webpack用 raw-loader 来加载文件的内容，而不是当做 UTF8 字符串传给我们
// 参见： https://webpack.github.io/docs/loaders.html#raw-loader
module.exports.raw = true
```
