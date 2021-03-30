---
title: Debug electron main process on VSCode
date: 2019-01-15 20:03:48
tags: Electron
---

普通的后台开发，在调试阶段都以添加日志为首要的开发手段，根据不同的日志信息进行程序运行状态的推断。[Electron](https://electronjs.org/) 以整合 node.js 以及 chromium 提供跨平台应用开发的方案，在进行 electron 开发时，笔者很长一段时间都用 node.js 的后台开发方式进行桌面应用开发，以至于开发体验非常的差。本着“磨刀不误砍柴工”的原理，开始了寻找高效开发 electron 应用的方式。

### 官方指导

在 Electron 官网中，可以找到文档 [使用 VSCode 进行主进程调试 | Electron](https://electronjs.org/docs/tutorial/debugging-main-process-vscode) 指引如何进行调试开发。

### 发现问题

Electron 主进程使用的是 node.js，由于 node.js 中的 V8 引擎对 ES6 的支持并不完善，导致根据官网文档方式进行调试开发 ES6 语法的 js 文件，会报出异常，所以官方指导方法只适用于非 ES6 语法开发的应用。
而使用 ES6 开发应用最好的方式就是接入 webpack+babel 的配置。

### 解决方案

electron+webpack+babel 的开发方式，github 上已经存在不少的模板，本文以 [Electron-Vue](https://github.com/SimulatedGREG/electron-vue) 的模版为例。

##### 1. 修改 webpack dev 配置：

在 webpack 默认的配置中，启动 webpack-dev-server 后，程序默认会将构建好的资源装载在内存中，以提高执行访问速度。这里需要对默认配置进行修改，将构建成功后的资源写入到磁盘中，以便 webpack 能指定程序启动入口。
在 debug 配置选项中，增加下述配置，即可开启存储到磁盘的功能。
```
{
...
  devServer: {
    writeToDisk: true,   // 开启磁盘存储功能
    inline: false        // 关闭内联模式，修复可能 webpack 运行失败问题
  },
  devtool: '#source-map' // 保存 sourceMap 文件，断点调试需要
}
```

##### 2. 启动 webpack-dev-server，分别对主进程以及渲染进程的 js 资源进行构建打包：

因为在 electron-vue 的模板中，主进程跟渲染进程的 webpack 配置是独立存储的，所以在启动 webpack-dev-server 的时候，需要同时启动两个进程的构建任务，方可正常调试。
除此以外，需要注意的是，在模板中渲染进程占用的是 9080 端口，那么主进程的 webpack-dev-server 注意不要使用同一个端口，下面例子主进程使用的端口为 9081。

假设构建输出目录为 app/dist，新建一个 terminal，执行下面的命令：
```
# 启动主进程构建任务
$: webpack-dev-server --colors --config ./.electron-vue/webpack.main.config.js --port 9081 --content-base app/dist
```

同时，在另一个 terminal 中，执行下面的命令：
```
# 启动渲染进程构建任务
$: webpack-dev-server --colors --config ./.electron-vue/webpack.renderer.config.js --port 9080 --content-base app/dist
```

##### 3. 配置 VSCode debug：

1. 启动 VSCode，打开对应项目，点击左侧调试按钮 (Command + Shift + D) 进入调试面板，选择 Add Configuration… VSCode 会创建 .vscode 目录，并新建一个 launch.json 的文件，该文件是 VSCode 调试面板的配置信息；
2. 在 launch.json 文件中，输入以下内容。(其中 program 字段为 webpack-dev-server 构建后的文件)
```
{
  // Use IntelliSense to learn about possible Node.js debug attributes.
  // Hover to view descriptions of existing attributes.
  // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
			"name": "Launch",
			"type": "node",
			"request": "launch",
			"program": "${workspaceRoot}/dist/electron/main.js",
			"stopOnEntry": false,
			"args": [],
			"cwd": "${workspaceRoot}",
                        // this points to the electron task runner
			"runtimeExecutable": "${workspaceRoot}/node_modules/.bin/electron",
			"runtimeArgs": [
				"--nolazy"
			],
			"console": "integratedTerminal",
			"sourceMaps": true,
			"outFiles": []
    }
	]
}
```
3. 选择调试面板中左上角的绿色箭头，即可开始调试。如果运行正常，VSCode 底部状态栏会由蓝色转变为橙色；
4. 在 VSCode 代码页面，点击代码行号左侧空白位置即可添加断点，添加成功后，断点为实心红点。如添加后为灰色空心圆点，则属于添加断点失败；

#### 热更新

- 主进程跟渲染进程的编辑都能够热更新
- 编辑主进程资源后，点击 VSCode 调试面板绿色刷新按钮，才能执行最新的代码
- 编辑渲染进程资源后，在主窗口 Command+R 刷新页面，即可执行最新的代码

Have Fun.
