---
title: 用electron实现播放flash小游戏
date: 2021-08-07 13:27:13
tags: [electron,flash]
categories:
 - app
---
# 0 背景
从小学开始，我就很喜欢玩4399上的[宠物连连看flash小游戏](https://www.4399.com/flash/17801_4.htm)。

然而，因为flash的安全性等问题，很多浏览器都不再支持flash。比如chrome，自 2021 年起，[在任何版本的 Chrome 中，Flash 内容（包括音频和视频）都将无法再正常播放](https://www.blog.google/products/chrome/saying-goodbye-flash-chrome/)，并且从v88起完全移除Flash插件的相关代码。
<!-- more -->
![不能玩了](/assets/img/2021/08/game.png)


# 1 解决方案
因此，我想到了，利用 Electron ，指定一个低于v88的版本的chromium，使用 Pepper Flash 插件加载钟爱的flash小游戏，以供随时娱乐^ ^。

## 1.1 electron原理
[Electron](https://www.electronjs.org/docs)是一个使用 JavaScript、HTML 和 CSS 构建桌面应用程序的框架。它嵌入了 Chromium 和 Node.js，允许您使用 JavaScript、HTML 和 CSS  代码创建 在Windows、 macOS和Linux上运行的跨平台应用。

按我的理解，就是利用Node.js创建一个服务，把开发者写好的网页部署到Node.js服务中，并在Chromium浏览器中打开这个网址。

## 1.2 主要代码
版本：项目中的`electron`版本为`^11.4.7`，chromium版本为`87.0.4280.141`。

在`index.html`，就只放一个宠物连连看flash小游戏。
``` html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <!-- https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP -->
    <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
    <meta http-equiv="X-Content-Security-Policy" content="default-src 'self'; script-src 'self'">
    <title>宠物连连看</title>
    <link rel="stylesheet" type="text/css" href="./electron/main.css" />
  </head>
  <body style="margin: 0;">
    <embed class="game" src="./electron/assets/game.swf"> </embed>
  </body>
</html>
```
在`window.js`中，实现创建窗口方法，打开`index.html`。
``` js
  
const { BrowserWindow } = require('electron');
const path = require('path');

function createWindow () {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    useContentSize: true,
    webPreferences: {
      plugins: true,
    }
  });

  win.loadFile('index.html');
  const contents = win.webContents;
  contents.on("did-finish-load", () => {
    contents.insertCSS("html, body { height: 100vh; width: 100vw; }");
  });
}

module.exports = createWindow;
```
在`plugins.js`中，根据不同的操作系统引入不同的Pepper Flash 插件。
``` js
const getFlashPlugin = () => {
  if (process.platform === "win32") {
    if (process.arch === "x64") {
      return {
        pluginPath: "pepflashplayer64_34_0_0_164.dll",
        version: "34.0.0.164",
      };
    }

    return {
      pluginPath: "pepflashplayer32_32_0_0_363.dll",
      version: "32.0.0.363",
    };
  }

  if (process.platform === "darwin") {
    return {
      pluginPath: "PepperFlashPlayer.plugin",
      version: "30.0.0.127",
    };
  }

  return null;
};

module.exports = {
  getFlashPlugin
};
```
在`main.js`中引入`plugins.js`和`window.js`并调用。
``` js
const { app, BrowserWindow } = require('electron');
const path = require('path');
const { getFlashPlugin } = require('./plugins'); 
const createWindow = require('./window'); 

// 获得系统里面flash插件的位置
const flashPlugin = getFlashPlugin();
let flashFlag = false;
if (flashPlugin) {
  const { pluginPath, version } = flashPlugin;
  app.commandLine.appendSwitch("ppapi-flash-path", path.join(__dirname, 'assets', pluginPath));
  app.commandLine.appendSwitch("ppapi-flash-version", version);
  flashFlag = true;
}

app.whenReady().then(() => {
  if (!flashFlag) {
    console.error('get flash plugin error');
  }
  console.log('chrome version: ', process.versions.chrome); //  Chromium v88 以上版本（包含 v88）内核的浏览器不再支持 Flash
  createWindow();
});

// 如果没有窗口打开则打开一个窗口 (macOS)
app.on('activate', function () {
  if (BrowserWindow.getAllWindows().length === 0) createWindow()
});

// 关闭所有窗口时退出应用-windows--linux
app.on('window-all-closed', function () {
  if (process.platform !== 'darwin') app.quit()
});
```
详细代码可看下面的源码。

# 2 源码
github仓库：https://github.com/seminelee/electron-flash-linklink
只需运行简单的命令，你就能得到一个桌面版的flash小游戏应用！希望你能收获你想要的快乐！

# 3 最终效果
![最终效果](/assets/img/2021/08/game-electron.png)

# 参考
 - [使用 Pepper Flash 插件](https://www.bookstack.cn/read/electronjs-8.0.0-zh/tutorial-using-pepper-flash-plugin.md)
 - [electron 文档](https://www.electronjs.org/docs)