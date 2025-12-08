---
layout: post
title: A Brief History of Front-End Build Tools
date: 2023-08-30 16:57:44
tags: [Webpack, Vite]
categories: build-tool
---

This article aims to answer: Why do front-end projects need to be built and packaged? Why do build tools need to be updated and iterated, and how did they evolve into what they are now?
![构建工具发展简史](/assets/img/2023/08/module-history.png)
<!--more-->
# No modular era
Before browsers supported ES modules, JavaScript did not provide a native mechanism for developers to develop in a modular manner.


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
# AMD/CMD - Asynchronous loading of modules
In order to solve the problem of browser-side JS modularization, there has been a way to solve this problem by introducing related tool libraries. Two widely used specifications and related libraries have emerged: AMD (RequireJs) and CMD (Sea.js).

AMD advocates dependency pre-positioning and early execution, while CMD advocates dependence nearby and delayed execution. They both use asynchronous methods to load modules. From now on, there is no need to manually maintain the code reference order, and the problem of global variable pollution is also solved.

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

# The era of front-end engineering
In 2009, Node.js appeared, and JS finally broke free from the constraints of the browser and began to have the ability to operate files.

Nodejs not only occupies a place in the server field, but also brings front-end engineering on the right track.此时出现了Grunt、Gulp等第一批基于Node.js的自动化构建工具，能够帮忙我们处理代码压缩、编译、单元测试等工作。
The CommonJS modular specification followed by Node.js has also become mainstream.
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

CommonJS is a server-side module specification, and loading modules is synchronous.`browserify`Committed to packaging and outputting CommonJS standard JS code that can run on the browser side.

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

# ESM specification emerges and webpack becomes mainstream
In 2015, official JavaScript modularization finally appeared, called the ESM specification. It is quite simple to use and replaces CommonJS and AMD specifications as a universal module solution for browsers and servers. However, the official only explains how to implement the specification, and browsers rarely support it.
``` js
// config.js
export const hello = 'hello world!';
export const api = `${prefix}/api`;

// main.js中引用
import { api, hello } from './config.js';
```

With the emergence and rise of the MVC framework and ESM, webpack2 was released and announced support for AMD\CommonJS\ESM, css/less/sass/stylus, babel, TypeScript, JSX, Angular components, and Vue components. Never before has such a large and comprehensive tool supported so many features and been able to solve almost all current build-related problems.

Rollup is based on esm and implements the powerful Tree-Shaking function, making the built product simple and small enough, and is favored by many open source library developers.
Webpack later also supported Tree-shaking, which is widely used in medium and large projects. .


# ESM specification native support, esbuild and vite
esbuild supports ES6/CommonJS specifications, Tree Shaking, TypeScript, JSX and other features.
![esbuild](/assets/img/2023/08/esbuild.png)

You can see that compared to webpack, you can see that the performance is improved by a hundred times. Why so fast?
  - Language advantage. esBuild is written in the Go language. Before esBuild, front-end build tools were based on Node and written in JS. JavaScript is an interpreted scripting language. Even if the V8 engine has done a lot of optimizations (JWT just-in-time compilation), it is still essentially unable to break the performance bottleneck. Go is a compiled language. The source code is translated into machine code during the compilation stage. You only need to directly execute these machine codes when starting.
  - Go is inherently multi-threaded, while JavaScript is essentially a single-threaded language. esBuild is carefully designed to achieve completely parallel processing of code parse, code generation and other processes.
  - esBuild only provides a minimal set of features for modern web applications.

Vite is the next generation of front-end development and building tools. Different processing is done in the development environment and production environment. In the development environment, the bottom layer is based on esBuild for speed up, and in the production environment, rollup is used for packaging.
Why use rollup for production builds?

 - Due to browser compatibility issues and the use of ESM in the actual network may cause the RTT time to be too long, packaging and building are still required.
![caniuse](/assets/img/2023/08/caniuse.png)
 - Although esbuild is fast, it has not yet released the 1.0 stable version. In addition, esbuild has weak support for code splitting and css processing, so rollup is still used in production environments.

# reference
 - [前端构建工具进化历程](https://juejin.cn/post/7205766006253011004#heading-10)