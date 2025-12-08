---
layout: post
title: Front-end webassembly + web worker video frame capture
date: 2021-08-11 16:35:20
tags: [ffmpeg,wasm]
categories:
 - media
---
# 0 background
In the current business scenario, the user uploads a video. After the video is uploaded successfully, the frame capture service will be run in the background, and finally the picture will be returned as a recommended cover and displayed to the user. This solution requires waiting for the video to be uploaded, reading the video in the background, and then running the frame capture task. The user waits for a long time.

Therefore, consider using the front end to capture frames and generate recommended covers when uploading videos to improve user experience.
<!-- more -->

# 1 Comparison of plans

## 1.1 `canvas`Cut frame
use`<video>`Tag play video, reuse`videoObject.currentTime=seconds`Set to play at a specified time, and finally`<canvas>`to draw pictures. There is a related[开源库](https://github.com/zzarcon/video-snapshot), you can experience it[demo](https://zzarcon.github.io/video-snapshot/)。
but,`<video>`Supports limited video packaging formats,[只支持MP4、WebM和Ogg](https://developer.mozilla.org/zh-CN/docs/Web/Media/Formats/Video_codecs). This is inconsistent with the logic of the existing business network. Format such as mov and flv cannot be uploaded and cannot meet the online standards.
![canvas-video](/assets/img/2021/08/canvas.png)

## 1.2 `Webassembly`Cut frame
Written in powerful C/C++[ffmpeg](https://ffmpeg.org/about.html), packaged into the form of wasm + js through the emscripten compiler, and then use js to implement the video frame capture function.
In terms of compatibility,`Webassembly`Has support from all major browsers but only[部分浏览器仍不支持](https://caniuse.com/?search=webassembly), using the old scheme for browsers that don't support it.
This solution has been put into practice on platforms such as Station B, and there are relevant implementations for reference. Finally decided to use this solution.

## 1.3 `Webassembly`Implementation comparison of frame capture
### 1.3.1 ffmpeg.wasm
Currently, there are open source libraries[ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm). The library includes:
 - [@ffmpeg/core](https://github.com/ffmpegwasm/ffmpeg.wasm-core): Compile ffmpeg to generate ffmpeg-core.wasm + js glue code.
 - [@ffmpeg/ffmpeg](https://github.com/ffmpegwasm/ffmpeg.wasm): Implements the part of calling the glue code generated in the previous step, and provides load, run and other APIs. At the same time, if developers are not satisfied with @ffmpeg/core, they can also build a custom ffmpeg-core.wasm.

![@ffmpeg/ffmpeg](/assets/img/2021/08/ffmpeg-wasm.png)
So, can it be used directly? These issues remain to be resolved:
 - Browser compatibility: We know that the browser's js thread is single-threaded and mutually exclusive with the rendering thread. In order not to block the rendering of the page and the js main thread,`@ffmpeg/core`When compiling ffmpeg, configure[多线程](https://emscripten.org/docs/porting/pthreads.html), resulting in the generated js glue code being used`sharedarraybuffer`。[<<<CODE_119>>>](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)It can meet the data sharing between the main thread and workers, and can also meet the data sharing between multiple workers. It is ideal for use in this scenario. But, because[安全问题](https://meltdownattack.com/), all mainstream browsers are disabled by default, and some additional return header fields need to be configured, and[支持度不太理想](https://caniuse.com/?search=sharedarraybuffer), cannot meet the online standard.
 ![SharedArrayBuffer](/assets/img/2021/08/sharedarraybuffer.png)
 - wasm redundancy:`@ffmpeg/core`compiled`ffmpeg-core.wasm`Includes almost all functions of ffmpeg,[文件](https://unpkg.com/@ffmpeg/core@0.10.0/dist/ffmpeg-core.wasm)The size is 24MB (8.5MB after gzip), a lot of which is not needed for frame capture.

### 1.3.2 Implementation of other platforms
According to business requirements (there are relatively few supported formats), by customizing ffmpeg, the size of the final wasm file generated can be reduced to 4.7MB (can be smaller after gzip).
However, I maintain a C language entry file myself, use the internal library provided by FFmpeg to implement the frame capture function, and then compile ffmpeg.
This method tests the understanding of FFmpeg and is bound to a specific version of ffmpeg. As the version of FFmpeg is upgraded, the API and directory of ffmpeg may change. In addition, as the business develops, we may need to modify this C code when we use more functions of ffmpeg, which has low maintainability.

## 1.4 Summary
Therefore, the final solution adopted is to use Webassembly to capture frames, and the specific implementation is:
 1. Customize compilation of ffmpeg and optimize wasm file size.
 2. Use the fftools/ffmpeg.c entry file provided by ffmpeg (v4.3.1) without writing C code yourself.
 3. Compile without`sharedarraybuffer`ffmpeg-core.wasm+js, and finally use the web worker to run the business code related to frame capture to prevent blocking the main thread.
 4. Call the js glue code of ffmpeg generated by compilation to realize the frame capture function. This part can be used`@ffmpeg/ffmpeg`。


# 2 Customize compilation of ffmpeg
## 2.1 Run docker using the official emscripten environment
emscripten is a WebAssembly compiler toolchain.

download[Docker Desktop](https://www.docker.com/get-started),pass[运行docker](https://emscripten.org/docs/getting_started/downloads.html#using-the-docker-image)The method uses the already built Emscripten environment to avoid the pitfalls of the local development environment.
Mac's Docker Desktop always fails to connect. It has been tested that running docker commands on Windows' Ubuntu is more stable.

In the ffmpeg source directory, write the script to run docker as follows:
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

### 2.1.1 Understand the principle of emscripten
Specifically, languages ​​such as C/C++ are transformed into LLVM intermediate code (IR) through the clang front-end, and then from LLVM IR to wasm. Then the browser downloads WebAssembly, and then passes through the WebAssembly module first, then to the assembly code of the target machine, and then to the machine code (x86/ARM, etc.).

![img](/assets/img/2021/08/emscripten.jpg)

So, what are LLVM and Clang?
 - LLVM means that different front-end and back-end use unified intermediate code LLVM Intermediate Representation (LLVM IR).
 - Clang is a sub-project of LLVM, a C/C++/Objective-C compiler front-end based on the LLVM architecture.

![img](/assets/img/2021/08/llvm.jpg)
> - Frontend: lexical analysis, syntax analysis, semantic analysis, intermediate code generation
> - Optimizer: intermediate code optimization (loop optimization, deletion of useless code, etc.)
> - Backend: generate target code. If the object code is absolute instruction code (machine code), such object code can be executed immediately. If the target code is assembly instruction code, it needs to be compiled by an assembler (generating machine code) before it can be run.

The next step is to write the compiled script`build.sh`。

## 2.2 Configure ffmpeg compilation parameters and remove redundancy
ffmpeg is an excellent C/C++ audio and video processing library that can realize video screenshots.

First, we need to know the libraries and components involved in implementing screenshots.

Libraries involved:
 - libavcodec: audio and video encoding and decoding.
 - libavformat: audio and video encapsulation and decapsulation.
 - libavutil: A library that contains some public utility functions, including arithmetic operations, character operations, etc.
 - libswscale: Image scaling and pixel format conversion.
Components involved:
 - demuxer: Decapsulate video
 - decoder: Decode the video
 - encoder: After getting the decoded frame, output the picture encoding
 - muxer: image packaging

use[<<<CODE_125>>>](https://emscripten.org/docs/compiling/Building-Projects.html#integrating-with-a-build-system)Set appropriate environment parameters and configure FFmpeg compilation parameters.
Documentation about configuration:
 - run`emconfigure ./configure --help`View all available configurations.
 - For detailed instructions on FFMPEG configuration, please click[这里](https://cloud.tencent.com/developer/article/1393972)Check.
 
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

## 2.4 Generate js+wasm
use`emcc`Compile the link code generated by make in the previous step into JavaScript + WebAssembly. used here`fftools/ffmpeg.c`As an entry file, you do not need to maintain a C language entry file yourself.
Passable`emcc --help`Check[emcc参数选项](https://emscripten.org/docs/tools_reference/emcc.html#command-line-syntax), and through`clang --help`Check[clang参数选项](https://linux.die.net/man/1/clang)。
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

The size of the final built ffmpeg-core.wasm is 5MB, and it will be smaller after gzip.
Source code:[build.sh](https://github.com/seminelee/ffmpeg.wasm-core/blob/n4.3.1-wasm/build1.sh)

At this point, compiling ffmpeg is completed! Next, return to the front-end area we are familiar with.

# 3 Implement frame-cutting function
## 3.1 Call js glue code
Regarding this part of calling js glue code, in the open source library`@ffmpeg/ffmpeg`has been implemented, we can simply use its[API](https://github.com/ffmpegwasm/ffmpeg.wasm/blob/master/docs/api.md)。
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

During this period, we also discovered`-ss`put on`-i`Before, you can intercept frames at a specified time without waiting to read frame by frame, which can improve the screenshot speed. You can view related[API文档](https://ffmpeg.org/ffmpeg.html)。
![ffmpeg文档](/assets/img/2021/08/ffmpeg-api.png)

P.S.`@ffmpeg/ffmpeg`Loading and removing is not currently supported.`pthreads`ffmpeg-core.wasm+js has also been mentioned for this library.[pr](https://github.com/ffmpegwasm/ffmpeg.wasm/pull/235)。

### 3.1.1 Exchanging data between JavaScript and C
That`load`and`run`How is the method implemented?
The first thing to understand here is that,[JavaScript与C交换数据](https://www.cntofu.com/book/150/zh/ch2-c-js/ch2-04-data-exchange.md)When , you can only use Number as a parameter. Because from a language perspective, JavaScript and C/C++ have completely different data systems, and Number is the only intersection between the two, so essentially when they call each other, they are exchanging Number.

Therefore, if the parameter is a non-Number type such as a string or array, it needs to be split into the following steps:
 - use`Module._malloc()`exist`Module`Allocate memory in the heap and obtain the address ptr;
 - Copy string/array and other data into the ptr of memory;
 - Use ptr as a parameter to call the C/C++ function for processing;
 - use`Module._free()`Release ptr.

The following is[<<<CODE_141>>>](https://github.com/ffmpegwasm/ffmpeg.wasm)Part of the source code:
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
Because there is no configuration in the build`-s USE_PTHREADS=1 `, called above`ffmpeg`The method will block the js main thread and the rendering of the page. For example, while generating recommended covers, the progress status of uploaded videos cannot be updated, and users cannot respond when clicking other buttons on the page. Therefore, a web worker needs to be added to run it.
[<<<CODE_144>>>](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Using_web_workers)is a script that runs on a thread separate from the browser page thread and can be used to offload almost any heavy processing from the page thread. The main thread and workers can pass`postMessage()`method and`onmessage`Communicate events.

but use`postMessage()`method and`onmessage`Writing communication processes for events can make the code cumbersome. It is recommended to use here[<<<CODE_149>>>](https://github.com/GoogleChromeLabs/comlink)(1.1kB), making the code more friendly and making communication imperceptible.

For example, frame interception communication:
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
`Comlink`is based on Es6`Proxy`and`postMessage()`RPC implementation. In the example,`ffmpegWorker`is an object located in worker.js. What is obtained in main.js is just`ffmpegWorker`The handle of the ontology, in fact`ffmpegWorker.getFrames`The execution of other methods is also run on worker.js.
The only pitfall is this library[产出是es6代码](https://github.com/GoogleChromeLabs/comlink/issues/477), and also need to be converted to es5 code through build configuration.
## 4.1 webpack configuration
In addition, if you use webpack, you may also encounter failure to load the correct`worker.js`Path problem. It can be configured like this[<<<CODE_157>>>](https://www.npmjs.com/package/worker-plugin)。
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

# 5 online effect
After going online, for browsers that support this solution, users can select and edit video covers without waiting for the video to be uploaded.

Moreover, compared with capturing the frame after reading the video in the background, the time taken to capture the frame on the front end is also greatly reduced. This is more noticeable with larger videos.

# 6 subsequent optimization points
## 6.1 Improve browser support rate
After reporting errors in some browsers, we will continue to optimize and improve browser support. (For example, a certain version of Safari reports an error in fetch wasm).
## 6.2 Reduce wasm file size
There is still room for reduction in the size of wasm. (If configured in the compilation configuration`enable-filters`All filters used).
## 6.3 Optimization of reading video files
Because MEMFS is used by default, the entire video file will be stored in memory and then processed. For large video files, such as 800MB+ video files, in the Firefox 90 version, the memory usage will be close to 3G when running tasks, and the browser will crash.
``` js
const getVideoInfo = async (file) => {
  // ...先实现fileToUint8Array方法
  const bufferArr = await fileToUint8Array(file);
  ffmpeg.FS('writeFile', 'example.mp4', bufferArr); // 先保存到MEMFS
  await ffmpeg.run('-i', 'example.mp4', '-loglevel', 'info');
}
```
 ![firefox](/assets/img/2021/08/firefox.png)

The current solution that comes to mind is to use[WORKERFS](https://emscripten.org/docs/api_reference/Filesystem-API.html#filesystem-api-workerfs). WORKERFS runs in a Web Worker and provides read-only access to the File and Blob objects inside the worker without copying the entire data to memory, which meets our needs.
 ![workerfs](/assets/img/2021/08/workerfs.png)



# reference
 - [Build FFmpeg WebAssembly version (= ffmpeg.wasm): Part.2 Compile with Emscripten](https://jeromewu.github.io/build-ffmpeg-webassembly-version-part-2-compile-with-emscripten/)
 - [前端视频帧提取 ffmpeg + Webassembly](https://juejin.cn/post/6854573219454844935)