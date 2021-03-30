---
title: Electron asar 格式详解
date: 2019-03-19 20:08:11
tags:
---

## What is asar?
[asar](https://github.com/electron/asar) —— 官方并没有明确给出简称的来源，笔者个人认为是 A Simple Archive，实际上，文档中也着重表达了这个格式只是属于简单的文件归档，因此笔者认为这个称呼也是挺合理的。:)  Electron 中提供了这个格式，在构建应用时将执行资源打包到一个 .asar 文件中，应用启动执行时直接访问 .asar 文件内部资源获取执行代码。在使用 [Electron Packager](https://github.com/electron-userland/electron-packager) 或者 [Electron Builder](https://github.com/electron-userland/electron-builder) 等构建工具时，都会默认开启 .asar 归档模式。

asar 属于一种将多个文件打包合并的文件，类似于 Linux 中的 tar 格式文件，Windows 中的 zip, rar 格式文件，然而不同于上面类比的格式，asar 属于无压缩类型的，也没有经过加密处理的，所有包含的文件的二进制数据都直接添加到 .asar 文件中，该文件头部包含一个 JSON 格式的字符串，记录其中包含的文件结构以及所有文件的起始位置以及文件长度：

```json
{
   "files": {				// 文件结构起始位置
      "tmp": {			// 其中包含了一个 tmp 文件夹
         "files": {}
      },
      "usr" : {			// 其中包含了一个 files 文件夹
         "files": {
           "bin": {		// files 文件夹中存在 bin 文件夹
             "files": {
               "ls": {	// bin 文件夹有一个 ls 文件
                 "offset": "0",	// 文件起始位置 为 0
                 "size": 100,		// 文件大小是 100
                 "executable": true	// 该文件可以被执行
               }
             }
           }
         }
      }
   }
}
```

通过这个 JSON 头部的字符串能够完全解析出 .asar 中包含的文件以及目录结构等信息。

## Why use it?
在 Windows 系统中，文件路径默认使用 256 位的字符串存储，因此在资源文件路径过深，或者资源父级文件夹名过长的情况下，就会出现资源访问失败的问题。当然这个问题能够通过修改系统注册列表去增加文件路径的长度，可是作为一个应用程序需要用户修改系统设置才能正常使用，这个是极其不合理的。因此 Electron 提出了将所有执行资源归档到一个 .asar 文件中的解决方案，.asar 文件会保留原来资源的层级结构，逻辑代码既能够无需额外修改，也能解决 Windows 中可能出现层级过深导致的执行失败问题。

除此之外，引入 .asar 格式能加快 require 访问的速度。原因是，如果使用的是独立的文件，在 require 中需要根据路径访问文件系统中对应的位置，然后在从磁盘中读取其中的内容，而 .asar 只需要根据文件路径获取到文件的偏移位置 offset 以及文件长度 size，就可以直接从其中获取文件的具体信息。一来只需要保持一个文件的读状态，无需同时读取多个文件，二来还能加快其访问速度。

最后一点，文档中也提到，引入 .asar 文件，能够隐藏项目开发的源代码。然而笔者认为，这个打包归档对于隐藏项目代码没有任何作用，正如前文所说，asar 属于无压缩无加密的归档类型，有意解包者能轻松导出获取其中的内容。对于隐藏项目代码，建议还是使用 [UglifyJS](http://lisperator.net/uglifyjs/) 等代码混淆工具。

## How do use it?
首先一点，建议通过 Electron-Builder 或者 Electron-Packager 等构建工具，开箱即用，相当方便。

全局安装 asar，或者通过 npx 命令执行：
```
# 全局安装 asar
$: npm install -g asar
```

使用 asar 打包资源文件：
```
$: asar pack your-app-resource-folder app.asar
```

将打包资源放到指定路径：
	macOS 中默认放入  `electron/Electron.app/Contents/Resources/app.asar`
	Windows 和 Linux 默认放入  `electron/resources/app.asar`

## Other Questions
Q: **Electron 是怎么读取 asar 文件的？**
A：Electron 中使用了其自定义的 `fs` `require` 等模块，在遇到 asar 文件时，会自动将其解析成一个虚拟的文件夹，然后直接获取其中的文件内容。

Q：**如何获取 asar 文件的属性？**
A：可能在运行过程中，会遇到需要获取 asar 文件属性需求（如计算 app.asar MD5 值，确保应用程序没有被他人篡改）时，需要忽略 asar 虚拟文件夹的功能，这个时候可以引入 `original-fs` 模块，该模块是 node.js 中原生的模块，对于 asar 没有提供额外的解析功能。

Q：**如何解包 asar 文件？**
A：解包 asar 可以分析 Electron 应用中的打包资源，可以查处应用包含了哪些不需要引入的资源，减少应用包的大小。解包操作同样是通过全局安装的 asar 工具即可：

```shell
$: asar extract app.asar dest_path
```

更多的 asar 工具命令，可以通过 `asar --help` 命令查看。

Enjoy it.
