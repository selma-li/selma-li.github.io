---
layout: post
title: 前端构建工具简史
date: 2023-08-30 16:57:44
tags: [Webpack, Vite]
categories: build-tool
---

本文旨在解答：前端项目为什么需要构建打包？构建工具为什么需要更新迭代，又是如何演变成现在这样的？
![构建工具发展简史](/assets/img/2023/08/module-history.png)
<!--more-->
# 无模块化时代
在浏览器支持 ES 模块之前，JavaScript 并没有提供原生机制让开发者以模块化的方式进行开发。

在无模块化时代，大量的全局变量充斥在我们的项目之中，代码引用时需要依赖特定顺序，这些都使得代码的维护变得更加复杂。
``` html
<html>
  <head>
    <title>JQuery</title>
  </head>
  <body>
    <div id="root"/>
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script type="text/javascript">
      $(document).ready(function(){
        $('#root')[0].innerText = 'Hello World'
      })
    </script>
  </body>
</html>
```
# AMD/CMD——异步加载模块
为了解决浏览器端 JS 模块化的问题，出现了通过引入相关工具库的方式来解决这一问题。出现了两种应用比较广的规范及其相关库：AMD(RequireJs) 和 CMD(Sea.js)。

AMD 推崇依赖前置、提前执行，CMD 推崇依赖就近、延迟执行，都采用异步的方法加载模块。从此不需要手动维护代码引用顺序，也解决了全局变量污染问题。

``` js
// AMD
// 加载完jquery后，将执行结果 $ 作为参数传入了回调函数
define(["jquery"], function ($) {
  $(document).ready(function(){
    $('#root')[0].innerText = 'Hello World';
  })
  return $
})

// CMD
// 预加载jquery
define(function(require, exports, module) {
  // 执行jquery模块，并得到结果赋值给 $
  var $ = require('jquery');
  // 调用jquery.js模块提供的方法
  $('#header').hide();
});
```

# 前端工程化时代
2009年，Node.js出现，JS终于挣脱了浏览器的束缚，开始拥有了文件操作的能力。

Nodejs 不仅在服务器领域占据一席之地，也将前端工程化带进了正轨。此时出现了Grunt、Gulp等第一批基于Node.js的自动化构建工具，能够帮忙我们处理代码压缩、编译、单元测试等工作。
Node.js遵循的CommonJS模块化规范也成为了主流。
``` js
// utils.js
var utils = {
  request() {
    console.log(window.utils);
  }
};
module.exports = utils;

// main.js中引入
var utils = require('./utils');
utils.request();
```

CommonJS 是服务器端模块规范， 加载模块是同步的。`browserify`致力于打包产出在浏览器端可以运行的 CommonJS 规范的 JS 代码。

``` js
// gulpfile.js
var browserify = require('browserify');
var gulp = require('gulp');
// vinyl-source-stream可以作为两者粘合剂
// 使用它将browserify生成的文件转换成gulp支持的stream
var source = require('vinyl-source-stream');
 
gulp.task('browserify', function() {
  return browserify('./src/javascript/app.js')
    .bundle()
    // Pass desired output filename to vinyl-source-stream
    .pipe(source('bundle.js'))
    // Start piping stream to tasks!
    .pipe(gulp.dest('./build/'));
});
```

# ESM规范出现，webpack成为主流
在 2015 年，JavaScript 官方的模块化终于出现了，被称为ESM规范。使用相当简单，取代了 CommonJS 和 AMD 规范，成为浏览器和服务器通用的模块解决方案。但是官方只阐述如何实现该规范，浏览器少有支持。
``` js
// config.js
export const hello = 'hello world!';
export const api = `${prefix}/api`;

// main.js中引用
import { api, hello } from './config.js';
```

伴随着 MVC 框架以及 ESM 的出现与兴起，webpack2 顺势发布，宣布支持 AMD\CommonJS\ESM、css/less/sass/stylus、babel、TypeScript、JSX、Angular组件，Vue组件。从来没有一个如此大而全的工具支持如此多的功能，几乎能够解决目前所有构建相关的问题。

rollup 基于 esm，实现了强大的 Tree-Shaking 功能，使得构建产物足够的简洁、体积足够的小，被许多开源库开发者青睐。
Webpack后来也支持Tree-shaking，在中大型项目中使用得非常广泛。。


# ESM 规范原生支持，esbuild和vite
esbuild 支持 ES6/CommonJS 规范、Tree Shaking、TypeScript、JSX 等功能特性。
![esbuild](/assets/img/2023/08/esbuild.png)

可以看到，对比webpack，可以看到性能足有百倍的提升。为什么这么快呢？
  - 语言优势。esBuild 是选择 Go 语言编写的，而在 esBuild 之前，前端构建工具都是基于 Node，使用 JS 进行编写。JavaScript 是一门解释性脚本语言，即使 V8 引擎做了大量优化（JWT 及时编译），本质上还是无法打破性能的瓶颈。而 Go 是一种编译型语言，在编译阶段就已经将源码转译为机器码，启动时只需要直接执行这些机器码即可。
  - Go 天生具有多线程运行能力，而 JavaScript 本质上是一门单线程语言。esBuild 经过精心的设计，将代码 parse、代码生成等过程实现完全并行处理。
  - esBuild 只提供现代 Web 应用最小的功能集合。

Vite是下一代前端开发与构建工具。在开发环境和生产环境分别做了不同的处理，在开发环境中底层基于 esBuild 进行提速，在生产环境中使用 rollup 进行打包。
为什么在生产环境中构建使用 rollup？

 - 由于浏览器的兼容性问题以及实际网络中使用 ESM 可能会造成 RTT 时间过长，所以仍然需要打包构建。
![caniuse](/assets/img/2023/08/caniuse.png)
 - esbuild 虽然快，但是它还没有发布 1.0 稳定版本，另外 esbuild 对代码分割和 css 处理等支持较弱，所以生产环境仍然使用 rollup。

# 参考
 - [前端构建工具进化历程](https://juejin.cn/post/7205766006253011004#heading-10)