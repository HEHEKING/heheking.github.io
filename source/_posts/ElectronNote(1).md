---
title: Electron 基础（一）
date: 2020/11/22 22:45:51
categories:
- 教程
tags:
- electron
- ipc
---

2020.10

"私たちの征途は星の海です"
<!-- more -->

## 一. 入门

### 1. 下载和安装

​	对于 Electron 而言，你需要先安装 Node.js 环境。

​	Node.js：https://nodejs.org/zh-cn/

​	Electron：https://www.electronjs.org/

​	第一步：创建一个新的文件夹作为 electron 的工作区、并进入她

```shell
mkdir electrondemo
cd electrondemo
```

​	第二步：初始化 npm 、安装 Electron

```shell
npm init -y
npm i --save-dev electron
# 也可以使用 cnpm
cnpm i --save-dev electron
```

​	安装 electron 时，我们需要一些时间。安装结束后，可以再次进行 npm 的初始化，目的是为了不手动创建 package.json

### 2. package.json

​	我们来看看一个现成的  package.json

```json
{
  "name": "fatfat_launcher",
  "version": "1.0.0",
  "description": "",
  "main": "main.js",
  "dependencies": {
    "electron": "^10.1.3",
    "electron-packager": "^15.1.0"
  },
  "devDependencies": {
    "electron": "^10.1.3",
    "electron-packager": "^15.1.0"
  },
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "electron .",
    "buildpack": "electron-packager ./  electrondemo --platform=win32 --arch=x64 --out=./app --electron-version=10.1.3 --electron-zip-dir=./electron-zip --overwrite"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

​	我们可以看到在 main 的值为 main.js，这是 electron 默认的主程序文件入口

​	这里的依赖中，除了 electron，还有一个 electron-packager，这是一个 electron 的打包程序。

​	在 scripts 中，我们定义了一段脚本命令 "start": "electron ."，这是调试当前的 electron 程序的命令，执行之后我们会运行它。

​	对于其他属性不明白的，可以查看以下文章

​	https://www.jianshu.com/p/b3d86ddfd555

### 3. main.js

这是一个简单例子

```js
// 引入依赖
const electron = require('electron');

// 创建主线程
const app = electron.app;
// BrowserWindow 是一个浏览器内核的窗口，这正是electron的核心
const BrowserWindow = electron.BrowserWindow;
const Menu = electron.Menu;

//声明要打开的窗口
let main = null;

//给主线程 app 绑定事件
app.on('ready', () => {
    //隐藏菜单栏
    Menu.setApplicationMenu(null);

    main = new BrowserWindow({
        width: 800,
        height: 600,
        resizable: false,
        webPreferences: {
            nodeIntegration: true,
            enableRemoteModule: true
        },
        frame: true,
        title: '肥宅启动器'
    });
    main.loadFile('./container/index.html');
    main.on('closed', () => {
        main = null;
    });

    console.log("app is ready.");
});
```

我们对 app 使用 on 方法，来监听它的生命周期，当它就绪的时候，我们开始对应用程式进行初始化，并按顺序做了接下来的事情：

1. 消除了 Menu （菜单栏），这是常见的 GUI 美化手段
2. 初始化 main ，创建了一个 BrowserWindow 对象
3. 使用 loadFile 来加载了一个 html 页面，并显示在了之前定义的 main 窗口中
4. 在之后的 on 函数里，我们为 closed 事件绑定了一个函数，用于销毁之前的 main 窗口
5. 在完成了这些之后，我们用 console.log() 打印了一段信息，表示初始化完成。

现在我们使用 npm start 来运行它

```
npm start
```

我们所加载的页面会被展示在窗口内。

## 二. 窗口通讯

当需要在 electron 中向子窗口传值，这是一种进程间通讯（**IPC**，*Inter-Process Communication*）。我们会使用到 ipcMain 模块。

### 示例

这里有一个简单的例子：

```js
app.on('ready', () => {
    //其他的代码
    ipcMain.on('setting-message', function (event, arg) {
        if(arg == 'get') {
            event.sender.send('setting-reply', setting);
        }
	});
}
```

这意味着，我们向 ipcMain 注册了一个名为 'setting-message' 的频道/事件，这里的 function 作为一个监听器存在，当受到 'setting-message' 这一标识的信息时，调用这个监听器去处理它。

### 事件对象

这里的 event 对象有以下方法：

`event.returnValue`

将此设置为在一个同步消息中返回的值.

`event.sender`

返回发送消息的 `webContents` ，你可以调用 `event.sender.send` 来回复异步消息.

arg，则是该信息的内容.

### 监听事件

官方文档：https://www.electronjs.org/docs/api/ipc-main

除了 ipcMain.on 之外，有以下几种方法：

`ipcMain.on(channel, listener)`

- `channel` String
- `listener` Function 

监听 `channel`, 当新消息到达，将通过 `listener(event, args...)` 调用 `listener`.

`ipcMain.once(channel, listener)`

- `channel` String
- `listener` Function

为事件添加一个一次性用的`listener` 函数.这个 `listener` 只有在下次的消息到达 `channel` 时被请求调用，之后就被删除了.

`ipcMain.removeListener(channel, listener)`

- `channel` String
- `listener` Function

为特定的 `channel` 从监听队列中删除特定的 `listener` 监听者.

`ipcMain.removeAllListeners([channel])`

- `channel` String (可选)

删除所有监听者，或特指的 `channel` 的所有监听者.

### 发送事件

在这里，我们先创建一个 render.js 的文件来绑定到目标页面上，在这个 js 文件中，我们可以处理许多 electron / javascript 上的操作。

一个例子：

**render.js**

```javascript
const $ = require('../module/jquery-3.5.1.min')
const ipcRenderer = require('electron').ipcRenderer;

$(document).ready(async function(){
    window.otaku = [];

    ipcRenderer.on("figureList-reply", function (event, arg) {
        window.otaku.figureList = arg;
    });
    ipcRenderer.on("setting-reply", function (event, arg) {
        let json = $.parseJSON(arg);
        window.otaku.setting = json;
    });
    ipcRenderer.on("saveFigureData-reply", function(event, arg) {
        if(arg == true) {
            console.log("发送成功-re");
        }
    });

    window.otaku.saveFigureData = function(obj) {
        ipcRenderer.send("saveFigureData-message", obj);
        console.log("发送成功");
        location.reload();
    }
});

window.onload = function () {
    ipcRenderer.send("setting-message", "get");
    ipcRenderer.send("figureList-message", "get");
}
```

当使用子窗口向父窗口发送通讯时，我们需要使用 ipcRenderer，他可以进行从渲染器进程到主进程的异步通信。

官方文档：https://www.electronjs.org/docs/api/ipc-renderer

ipcRenderer 的用法与 ipcMain 的用法非常相似。

将这个文件 用 script 标签引入即可。

```
<script src="../render/index.js"></script>
```

