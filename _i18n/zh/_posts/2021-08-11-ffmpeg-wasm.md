---
layout: post
title: 前端webassembly+web worker视频截帧
date: 2021-08-11 16:35:20
tags: [ffmpeg,wasm]
categories:
 - media
---
# 0 背景
目前在业务场景中，用户上传视频，等待视频上传成功后，后台会跑截帧服务，最后返回图片作为推荐封面，展示给用户。这个方案需要等待视频上传后，后台读取视频，再跑截帧任务，用户等待时间比较长。

因此，考虑前端来做截帧，在开始上传视频的同时生成推荐封面，提升用户体验。
<!-- more -->

# 1 方案对比

## 1.1 `canvas`截帧
利用`<video>`标签播放视频，再利用`videoObject.currentTime=seconds`设置到指定时刻播放，最后在`<canvas>`中进行绘制图片。有一个相关的[开源库](https://github.com/zzarcon/video-snapshot)，可以体验下它的[demo](https://zzarcon.github.io/video-snapshot/)。
但是，`<video>`支持视频封装格式有限，[只支持MP4、WebM和Ogg](https://developer.mozilla.org/zh-CN/docs/Web/Media/Formats/Video_codecs)。这与业务现网的逻辑不一致，mov、flv等格式不能上传，不能达到上线标准。
![canvas-video](/assets/img/2021/08/canvas.png)

## 1.2 `Webassembly`截帧
使用功能强大的C/C++编写的[ffmpeg](https://ffmpeg.org/about.html)，通过emscripten编译器打包成wasm + js的形式，再使用js实现视频截帧功能。
兼容性方面，`Webassembly`已得到了来自各大主要浏览器但支持，只有[部分浏览器仍不支持](https://caniuse.com/?search=webassembly)，对于不支持的浏览器采用旧方案。
该方案在b站等平台已有相关实践，有相关的实现可以参考。最后决定使用该方案。

## 1.3 `Webassembly`截帧的实现对比
### 1.3.1 ffmpeg.wasm
目前，已有开源库[ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm)。该库包括：
 - [@ffmpeg/core](https://github.com/ffmpegwasm/ffmpeg.wasm-core)：编译ffmpeg生成ffmpeg-core.wasm + js胶水代码。
 - [@ffmpeg/ffmpeg](https://github.com/ffmpegwasm/ffmpeg.wasm)：实现了调用上一步生成的胶水代码的部分，提供了load, run等API。同时，如果开发者对@ffmpeg/core不满意，也可以构建自定义的ffmpeg-core.wasm。

![@ffmpeg/ffmpeg](/assets/img/2021/08/ffmpeg-wasm.png)
那么，能直接用吗？有这些问题还待解决：
 - 浏览器兼容性：我们知道，浏览器的js线程是单线程的，并且与渲染线程互斥。为了不阻塞页面的渲染和js主线程，`@ffmpeg/core`在编译ffmpeg时，配置了[多线程](https://emscripten.org/docs/porting/pthreads.html)，导致产生的js胶水代码中使用了`sharedarraybuffer`。[`sharedarraybuffer`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)能满足主线程和worker之间的数据共享，也可以满足多个worker之间的数据共享，用于此场景中是很理想的。但是，因为[安全问题](https://meltdownattack.com/)，所有主流浏览器均默认禁用，需要另外配置一些返回头部字段，而且[支持度不太理想](https://caniuse.com/?search=sharedarraybuffer)，不能达到上线标准。
 ![SharedArrayBuffer](/assets/img/2021/08/sharedarraybuffer.png)
 - wasm冗余：`@ffmpeg/core`编译出来的`ffmpeg-core.wasm`几乎包括了ffmpeg的所有功能，[文件](https://unpkg.com/@ffmpeg/core@0.10.0/dist/ffmpeg-core.wasm)大小是24MB（gzip后是8.5MB），其中很多是截帧不需要的。

### 1.3.2 其他平台的实现
根据业务的要求（支持的格式比较少），通过自定义编译ffmpeg，最后生成的wasm文件大小可以减少到4.7MB（gzip后可以更小）。
但是，自己维护一份c语言的入口文件，用FFmpeg提供的内部库，实现截帧功能，然后再编译ffmpeg。
这种方式比较考验对FFmpeg的理解，而且与ffmpeg的特定版本绑定，而随着FFmpeg的版本升级，ffmpeg的API、目录可能会有变更。再加上，随着业务发展，我们可能会用到ffmpeg更多的功能时，还需要修改这份c代码，可维护性比较低。

## 1.4 小结
因此，最终采用的方案是，使用Webassembly截帧，具体实现：
 1. 自定义编译ffmpeg，优化wasm文件大小。
 2. 使用ffmpeg（v4.3.1）提供的fftools/ffmpeg.c入口文件，无需自己写c代码。
 3. 编译出不带`sharedarraybuffer`的ffmpeg-core.wasm+js，最后使用web worker运行截帧相关的业务代码，以防阻塞主线程。
 4. 调用编译生成的ffmpeg的js胶水代码，实现截帧功能，这部分可以使用`@ffmpeg/ffmpeg`。


# 2 自定义编译ffmpeg
## 2.1 运行docker使用官方的emscripten环境
emscripten是一个WebAssembly编译器工具链。

下载[Docker Desktop](https://www.docker.com/get-started)，通过[运行docker](https://emscripten.org/docs/getting_started/downloads.html#using-the-docker-image)的方式使用已经搭建好的Emscripten环境，避免本地开发环境的坑。
mac的Docker Desktop老是连接不上，实测windows的ubuntu跑docker命令更稳定。

在ffmpeg源码目录中，编写运行docker的脚本如下：
``` bash
#!/bin/bash
set -euo pipefail

EM_VERSION=2.0.8

docker pull emscripten/emsdk:$EM_VERSION
docker run \
  --rm \
  -v $PWD:/src \ # 绑定挂载
  -v $PWD/wasm/cache:/emsdk_portable/.data/cache/wasm \
  emscripten/emsdk:$EM_VERSION \
  sh -c 'bash ./build.sh'

```

### 2.1.1 了解下emscripten原理
具体来说，就是C/C++等语言，经过 clang 前端变成 LLVM 中间代码（IR），再从LLVM IR到wasm。然后浏览器把 WebAssembly 下载下来，然后先经过 WebAssembly 模块，再到目标机器的汇编代码，再到机器码（x86/ARM等）。

![img](/assets/img/2021/08/emscripten.jpg)

那，LLVM和Clang是什么呢？
 - LLVM，就是不同的前端后端使用统一的中间代码LLVM Intermediate Representation (LLVM IR)。
 - Clang是LLVM的一个子项目，基于LLVM架构的C/C++/Objective-C编译器前端。

![img](/assets/img/2021/08/llvm.jpg)
>  - Frontend前端：词法分析、语法分析、语义分析、生成中间代码
>  - Optimizer优化器：中间代码优化（循环优化、删除无用代码等等）
>  - Backend后端：生成目标代码。如目标代码是绝对指令代码（机器码），则这种目标代码可立即执行。如果目标代码是汇编指令代码，则需汇编器汇编之后（生成机器码）才能运行。

接下来是编写编译的脚本`build.sh`。

## 2.2 配置ffmpeg编译参数，去掉冗余
ffmpeg是优秀的C/C++音视频处理库，可以实现视频截图。

首先，我们要知道实现截图会涉及的库和组件。

涉及到的库：
 - libavcodec：音视频的编码和解码。
 - libavformat：音视频的封装和解封装。
 - libavutil：包含一些公共的工具函数的使用库，包括算数运算，字符操作等。
 - libswscale：图像伸缩和像素格式转化。
涉及的组件：
 - demuxer：对视频解封装
 - decoder：对视频解码
 - encoder：得到解码后的帧之后，输出图片编码
 - muxer：图片封装

使用[`emconfigure`](https://emscripten.org/docs/compiling/Building-Projects.html#integrating-with-a-build-system)设置合适的环境参数，和配置FFmpeg编译参数。
关于配置的说明文档：
 - 运行`emconfigure ./configure --help` 查看所有可以用的配置。
 - 关于FFMPEG 配置的详细说明可以点击[这里](https://cloud.tencent.com/developer/article/1393972)查看。
 
``` bash
# configure FFMpeg with Emscripten
emconfigure ./configure 
  --target-os=none        # use none to prevent any os specific configurations
  --arch=x86_32           # use x86_32 to achieve minimal architectural optimization
  --enable-cross-compile  # enable cross compile
  --disable-x86asm        # disable x86 asm
  --disable-inline-asm    # disable inline asm
  --disable-stripping     # disable stripping
  --disable-programs      # disable programs build (incl. ffplay, ffprobe & ffmpeg)
  --disable-doc           # disable doc
  --nm="llvm-nm"
  --ar=emar
  --ranlib=emranlib
  --cc=emcc
  --cxx=em++
  --objcc=emcc
  --dep-cc=emcc
  # 去掉不需要的库
  --disable-avdevice
  --disable-swresample
  --disable-postproc
  --disable-network
  --disable-pthreads
  --disable-w32threads
  --disable-os2threads
  # 配置需要的解封装，编解码器等
  --disable-everything # 减少wasm体积的关键，除了以下的组件外的个别组件都disable
  --enable-filters
  --enable-muxer=image2
  --enable-demuxer=mov # mov,mp4,m4a,3gp,3g2,mj2
  --enable-demuxer=flv
  --enable-demuxer=h264
  --enable-demuxer=asf
  --enable-encoder=mjpeg
  --enable-decoder=hevc
  --enable-decoder=h264
  --enable-decoder=mpeg4
  --enable-protocol=file

# build dependencies
emmake make -j4
```

## 2.4 生成js+wasm
使用`emcc`将上一步 make 生成的链接代码编译为 JavaScript + WebAssembly。这里使用`fftools/ffmpeg.c`作为入口文件，不需要自己维护一份c语言入口文件。
可通过`emcc --help`查看[emcc参数选项](https://emscripten.org/docs/tools_reference/emcc.html#command-line-syntax)，以及通过`clang --help`查看[clang参数选项](https://linux.die.net/man/1/clang)。
``` bash
emcc
  -I. -I./fftools # Add directory to include search path
  -Llibavcodec -Llibavdevice -Llibavfilter -Llibavformat -Llibavresample -Llibavutil -Llibpostproc -Llibswscale -Llibswresample # Add directory to library search path
  -Qunused-arguments # Don't emit warning for unused driver arguments.
  -o wasm/dist/ffmpeg-core.js # output
  fftools/ffmpeg_opt.c fftools/ffmpeg_filter.c fftools/ffmpeg_hw.c fftools/cmdutils.c fftools/ffmpeg.c # input
  -lavdevice -lavfilter -lavformat -lavcodec -lswresample -lswscale -lavutil -lm # library
  -s USE_SDL=2 # use SDL2
  -s MODULARIZE=1 # use modularized version to be more flexible
  -s EXPORT_NAME="createFFmpegCore" # assign export name for browser
  -s EXPORTED_FUNCTIONS="[_main]" # export main and proxy_main funcs
  -s EXTRA_EXPORTED_RUNTIME_METHODS="[FS, cwrap, ccall, setValue, writeAsciiToMemory]" # export extra runtime methods
  -s INITIAL_MEMORY=33554432 # 33554432 bytes = 32MB
  -s ALLOW_MEMORY_GROWTH=1 # allows the total amount of memory used to change depending on the demands of the application
  -s ASSERTIONS=1 # for debug
  --post-js wasm/post-js.js # emits a file after the emitted code. use to expose exit function
  -O3 # optimize code and reduce code size
```

最后构建的ffmpeg-core.wasm大小为5MB，gzip后会更小。
源码：[build.sh](https://github.com/seminelee/ffmpeg.wasm-core/blob/n4.3.1-wasm/build1.sh)

到这里，编译ffmpeg完成了！接下来回到我们熟悉的前端领域。

# 3 实现截帧功能
## 3.1 调用js胶水代码
关于调用js胶水代码的这部分，在开源库`@ffmpeg/ffmpeg`已经实现了，我们可以简单地使用它的[API](https://github.com/ffmpegwasm/ffmpeg.wasm/blob/master/docs/api.md)。
``` js
const { createFFmpeg } = require('@ffmpeg/ffmpeg');
const ffmpeg = createFFmpeg({ log: true });

(async () => {
  await ffmpeg.load();
  // ... 省略获取时长duration部分
  const frameNum = 8;
  const per = duration / (frameNum - 1);
  for (let i = 0; i < frameNum; i++) {
    await ffmpeg.run('-ss', `${Math.floor(per * i)}`, '-i', 'example.mp4', '-s', '960x540', '-f', 'image2', '-frames', '1', `frame-${i + 1}.jpeg`);
  }
})();
```

期间，还发现了`-ss`放在`-i`前，可以截取指定时间的帧，而不用等待逐帧读取，可以提升截图速度。可查看相关[API文档](https://ffmpeg.org/ffmpeg.html)。
![ffmpeg文档](/assets/img/2021/08/ffmpeg-api.png)

P.S.`@ffmpeg/ffmpeg`目前还不支持加载去掉`pthreads`的ffmpeg-core.wasm+js，也给该库提了[pr](https://github.com/ffmpegwasm/ffmpeg.wasm/pull/235)。

### 3.1.1 JavaScript与C交换数据
那`load`和`run`方法具体是怎么实现的呢？
这里要先了解的是，[JavaScript与C交换数据](https://www.cntofu.com/book/150/zh/ch2-c-js/ch2-04-data-exchange.md)时，只能使用Number作为参数。因为从语言角度来说，JavaScript与C/C++有完全不同的数据体系，Number是二者唯一的交集，因此本质上二者相互调用时，都是在交换Number。

因此如果参数是字符串、数组等非Number类型，则需要拆分为以下步骤：
 - 使用`Module._malloc()`在`Module`堆中分配内存，获取地址ptr；
 - 将字符串/数组等数据拷入内存的ptr处；
 - 将ptr作为参数，调用C/C++函数进行处理；
 - 使用`Module._free()`释放ptr。

以下是[`@ffmpeg/ffmpeg`](https://github.com/ffmpegwasm/ffmpeg.wasm)的部分源码：
``` js
const createFFmpegCore = require('path/to/ffmpeg-core.js');
let ffmpeg;

// 加载
const load = async () => {
  Core = await createFFmpegCore({
    print: (message) => {},
  });
  ffmpeg = Core.cwrap('_main', 'number', ['number', 'number']); // cwrap调用导出的主函数
};

const parseArgs  = (Core, args) => {
  const argsPtr = Core._malloc(args.length * Uint32Array.BYTES_PER_ELEMENT);
  args.forEach((s, idx) => {
    const buf = Core._malloc(s.length + 1);
    Core.writeAsciiToMemory(s, buf);
    Core.setValue(argsPtr + (Uint32Array.BYTES_PER_ELEMENT * idx), buf, 'i32');
  });
  return [args.length, argsPtr]; // [数组的长度, 数组的ptr]
};

// 执行ffmpeg命令
const run = (..._args) => {
  return new Promise((resolve) => {
    ffmpeg(...parseArgs(Core, _args)); // 传入命令参数
  });
};

module.exports = {
  load,
  run,
};
```

# 4 web worker
因为在构建中没有配置`-s USE_PTHREADS=1 `，上面调用`ffmpeg`的方法会阻塞js主线程和页面的渲染。比如，在生成推荐封面的同时，无法更新上传视频的进度状态，用户点击页面上的其他按钮也无法响应等。因此，需要增加一个web worker来运行。
[`Web Worker`](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Using_web_workers)是在与浏览器页面线程分开的线程上运行的脚本，可以用于从页面线程分流几乎所有繁重的处理。主线程和worker可以通过`postMessage()`方法和`onmessage`事件进行通信。

但使用`postMessage()`方法和`onmessage`事件进行编写通信过程会使代码显得繁琐。这里推荐使用[`Comlink`](https://github.com/GoogleChromeLabs/comlink) (1.1kB)，使代码变得更友好，让通信变得无感知。

比如截帧的通信：
main.js
``` js
import * as Comlink from 'Comlink';
async function onFileUpload(file) {
  const ffmpegWorker = Comlink.wrap(new Worker('./worker.js'));
  const frameU8Arrs = await ffmpegWorker.getFrames(file);
}
```
worker.js
``` js
import * as Comlink from 'Comlink';
async function getFrames(file) {
  // ...
  // 先获取时长duration等
  const frameNum = 8;
  const per = duration / (frameNum - 1);
  let frameU8Arrs = [];
  for (let i = 0; i < frameNum; i++) {
    await ffmpeg.run('-ss', `${Math.floor(per * i)}`, '-i', 'example.mp4', '-s', '960x540', '-f', 'image2', '-frames', '1', `frame-${i + 1}.jpeg`);
  }
  // 从MEMFS获取图片二进制数据Uint8Array
  for (let i = 0; i < frameNum; i++) {
    const u8arr = await ffmpeg.FS('readFile', `frame-${i + 1}.jpeg`); 
    frameU8Arrs.push(u8arr);
    ffmpeg.FS('unlink', fileName);
  }
  return frameU8Arrs;
}

Comlink.expose({
  getFrames,
});
```
`Comlink`是基于Es6 `Proxy`和`postMessage()`的RPC实现。例子中，`ffmpegWorker`是位于worker.js中的对象，main.js里拿到的只是`ffmpegWorker`的本体的句柄，实际上`ffmpegWorker.getFrames`等方法的执行也是在worker.js上运行的。
唯一一个坑点是这个库的[产出是es6代码](https://github.com/GoogleChromeLabs/comlink/issues/477)，还需要通过构建配置转为es5代码。
## 4.1 webpack配置
另外，如果你使用webpack，可能还会遇到无法加载正确的`worker.js`路径的问题。可以这样配置[`worker-plugin`](https://www.npmjs.com/package/worker-plugin)。
``` js
const WorkerPlugin = require('worker-plugin');
const isPub = true; // 是否生产环境
{
  // ...
  plugins: [
    new WorkerPlugin({
      globalObject: 'this',
      filename: isPub ? '[name].[chunkhash:9].worker.js' : '[name].worker.js',
    }),
  ],
}
```

# 5 上线效果
上线后，对于支持该方案的浏览器，用户无需等待视频上传完成，即可选择、编辑视频封面。

而且，比起在后台读取视频后截帧，前端截帧的耗时也大大缩小了。这在视频大小越大的视频越明显。

# 6 后续优化点
## 6.1 提高浏览器支持率
在部分浏览器报错，之后持续优化，提高浏览器支持率。（如Safari某版本fetch wasm报错）。
## 6.2 减少wasm文件大小
wasm体积还有减少的空间。（如在编译配置中配置了`enable-filters`使用了所有的filters）。
## 6.3 读取视频文件优化
因为默认使用MEMFS，会将视频文件整个存入内存中，然后处理。大的视频文件如800MB+的视频文件，在Firefox 90版本，运行任务时内存占用会接近3G，还会出现浏览器崩溃的情况。
``` js
const getVideoInfo = async (file) => {
  // ...先实现fileToUint8Array方法
  const bufferArr = await fileToUint8Array(file);
  ffmpeg.FS('writeFile', 'example.mp4', bufferArr); // 先保存到MEMFS
  await ffmpeg.run('-i', 'example.mp4', '-loglevel', 'info');
}
```
 ![firefox](/assets/img/2021/08/firefox.png)

目前想到的方案是，使用[WORKERFS](https://emscripten.org/docs/api_reference/Filesystem-API.html#filesystem-api-workerfs)。WORKERFS运行在 Web Worker 中，提供对 woker 内部的 File 和 Blob 对象的只读访问，而无需将整个数据复制到内存中，符合我们的需求。
 ![workerfs](/assets/img/2021/08/workerfs.png)



# 参考
 - [Build FFmpeg WebAssembly version (= ffmpeg.wasm): Part.2 Compile with Emscripten](https://jeromewu.github.io/build-ffmpeg-webassembly-version-part-2-compile-with-emscripten/)
 - [前端视频帧提取 ffmpeg + Webassembly](https://juejin.cn/post/6854573219454844935)