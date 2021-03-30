---
title: Electron 进程间通讯详解
date: 2019-06-30 12:46:09
tags:
---
## Electron 应用架构
Electron 不同于其他应用的架构，它存在两种进程类型，分别为**主进程**、**渲染进程**，这两种进程类型在应用的生命周期中承担了截然不同的角色，深入了解它们各自的角色尤为重要。

![ElectronProcesses.png](https://cdn.iguan7u.cn/image/ElectronProcesses.png)

### 主进程
Electron 运行 `package.json` 中的 `main` 脚本中的进程被称为主进程，该进程在应用整个生命周期只会存在唯一一个，负责界面窗口的创建、控制、销毁等管理行为，同时也能控制整个应用的生命周期。
通过阅读文档以及源码我们能够发现，主进程本质上是一个 Node.js 进程，该进程充分利用了 Node.js 的跨平台特性，在其 API 的底层，抹掉了不同操作系统中的差异，为 Electron 中跨平台开发中的文件管理提供了基础。
主进程同时提供一套跨平台的接口，能控制各个操作系统中的原生资源，如：对话框、菜单、文件拖拽等，使得 Electron 的用户体验极大的接近了原生应用。

### 渲染进程
Electron 使用 Chromium 来展示 UI 界面，在应用程序中被称为 `BrowserWindow` 。当主进程每创建一个独立 `BrowserWindow` 实例，Electron 都会初始化一个独立的渲染进程，隔离了不同窗口之间的环境，使得每个窗口都只需要关心自己内部的 web 页面。

## Electron 进程间通讯方式
在计算机系统设计中，不同的进程间内存资源都是相互隔离的，因此进程间的数据交换，会使用**进程间通讯**方式达成。而不同于一般的原生应用开发，Electron 的渲染进程与主进程分别属于独立的进程中，而且进程间会存在频繁的数据交换，这时选择一个合理的进程间通讯方式显得尤为重要。下面是 Electron 中官方提供的进程间通讯方式：

- **LocalStorage, window.postMessage**
在前端开发中，鉴于浏览器对本地数据有严格的访问限制，所以一般通过该两种方式进行窗口间的数据通讯，该方式同样适用于 Electron 开发中。然而因为 API 设计目的仅仅是为了前端窗口间简单的数据传输，大量以及频繁的数据通讯会导致应用结构松散，同时传输效率也值得怀疑。

- **IPC (Inner-Process Communication)**
Electron 中提供了 `ipcRender` 、`ipcMain` 作为主进程以及渲染进程间通讯的桥梁，该方式属于 Electron 特有传输方式，不适用于其他前端开发场景。Electron 沿用 Chromium 中的 IPC 方式，不同于 socket、http 等通讯方式，Chromium 使用的是命名管道 [IPC](https://www.chromium.org/developers/design-documents/inter-process-communication) ，能够提供更高的效率以及安全性。

主进程代码：
```javascript
const ipc = require('electron').ipcMain
ipc.on('getMsg', (event, msg) => {
  console.log(msg)     // You sended a message to main thread.
})
```

渲染进程代码：
```javascript
const ipc = require('electron').ipcRenderer
ipc.send('getMsg', 'You sended a message to main thread.')
```

可是 IPC 的使用方式类似于通知监听，在多个模块中使用容易造成项目代码的松散，加大后期维护成本。

- **Remote**
Remote 模块为渲染进程和主进程通信提供了一种简单方式，该方式类似于 Java 中的 [RMI](https://en.wikipedia.org/wiki/Java_remote_method_invocation)，在渲染进程中能显式调用主进程中的模块代码，封装了底层中进程间通讯的细节，让开发者能轻松上手使用。

渲染进程调用主进程模块：
```javascript
const { BrowserWindow } = require('electron').remote
let win = new BrowserWindow({ width: 800, height: 600 })
win.loadURL('https://github.com')
```

天下没有免费的午餐。上手易用的方式往往伴随着其可能带来的风险。在官方文档中提到，通过 Remote 模块从主进程返回到渲染进程的对象，在主进程和渲染进程中会同时存在两份实例，其生命周期是一致的。渲染进程中该对象被一直持有，主进程中对应的对象就一直不会被销毁。因此使用不当，很可能造成严重的内存泄漏。
Remote 底层仍然时候 IPC 作为进程间通讯的方式，尽管在调用过程中对使用者是无感的，可是在使用过程中有必要明白其基本原理。同时 Electron 通过 hook 了渲染进程中被返回对象的 setter, getter 方法，将两个进程中对应的对象实例进行了属性绑定，在渲染进程对该对象的属性进行修改，会同时修改主进程的对应对象的属性。在这种便捷的机制下，也容易引起其他问题。我们在下文中在详细描述。

## 进程间通讯问题
![WeDriveStruct.png](https://cdn.iguan7u.cn/image/WeDriveStruct.png)

在企业微信云盘项目初期，我们普遍使用 Remote 作为进程间通讯的方式，该模块正如其设计初衷一样，几乎抹掉了 Electron 应用存在两个独立进程的特点，在开发过程中几乎与单进程应用别无二致。得益于该模块，项目的开发进度一马平川。可是随着项目架构以及调试数据的进一步发展，严重的问题开始逐渐浮现：

### 1. 应用程序为何会出现卡顿？

随着主进程逻辑增多，数据库读取次数增加，页面开始逐渐出现了卡顿的现象，在 macOS 中甚至偶尔会出现沙滩排球的鼠标图案。这个情况逐渐引起了我们的重视，怀疑项目中的架构是否存在不合理之处。
我们在应用中增加了 `console.time` 以及 `console.timeEnd` 统计逻辑执行时间，发现相关数据解析、以及数据库保存读取操作并没有存在耗时很长的情况，我们很快就将目光放在了 remote 模块中。众所周知，进程间通讯往往存在不可忽视的性能损耗，而在 Electron 提供的如此便利的 rpc 方式下，是否也存在性能损耗呢？我们做了一个简单的 demo：

> 在实验中，我们从渲染进程中通过点击按钮，触发一个调用主线程的逻辑，返回 500 个各自拥有 50 个属性的 JSON 对象，然后在渲染进程中调用 `JSON.stringify` 方法，访问从主线程中返回的 JSON 对象，记录程序运行的时间。同时在渲染进程中增加一个定时器，每隔特定的时间更新页面中的数字，检查渲染进程的运行状况。  
> 附上 [测试 demo 代码](https://github.com/iGuan7u/ElectronRemoteBlockDemo)  

![ElectronBlock](https://cdn.iguan7u.cn/image/ElectronTest.gif)

通过这个 demo 我们可以发现，在点击按钮后，渲染进程的数字出现卡顿的情况，等到 `JSON.stringify` 方法打印完成后，页面的数字才正常的继续更新。
由此可见，使用 remote 模块调用主进程的逻辑是，是会存在导致渲染进程卡顿的情况的发生。

### 原因

可是究竟是 remote 模块中的哪些逻辑导致了渲染进程的卡顿呢？在 [官方文档中](https://electronjs.org/docs/api/remote) 中，并没有详细的提到 remote 模块的详细实现，同时也没有提到使用 remote 模块会导致渲染进程卡顿的问题。然后文档中略微提到了一点：

> remote 模块返回的每个对象 (包括函数) 表示主进程中的一个对象 (我们称它为远程对象或远程函数)。 当调用远程对象的方法时, 调用远程函数, 或者使用远程构造函数 (函数) 创建新对象时, 实际上是在发送**同步**进程消息。  

那么，渲染进程的卡顿，是否与该同步消息有关呢？阅读 Electron 的 remote [源码](https://github.com/electron/electron/blob/78411db4b5c796570ee0e5eec1ef00f586e0a01d/lib/renderer/api/remote.js) 我们发现，remote 模块的内部，实际是也通过调用 ipc 进行进程间通讯，以 `remote.getGlobal` 方法为例：

remote.js
```javascript
/* Get a global object in browser. */
exports.getGlobal = (name) => {
  const command = 'ELECTRON_BROWSER_GLOBAL'
  const meta = ipcRendererInternal.sendSync(command, contextId, name)
  return metaToValue(meta)
}
```

rpc-server.js
```javascript
handleRemoteCommand('ELECTRON_BROWSER_GLOBAL', function (event, contextId, globalName) {
  const customEvent = eventBinding.createWithSender(event.sender)
  event.sender.emit('remote-get-global', customEvent, globalName)

  if (customEvent.returnValue === undefined) {
    if (customEvent.defaultPrevented) {
      throw new Error(`Blocked remote.getGlobal('${globalName}')`)
    } else {
      customEvent.returnValue = global[globalName]
    }
  }

  return valueToMeta(event.sender, contextId, customEvent.returnValue)
})
```

remote 使用一个内部的 ipc 通道 `ipcRendererInternal` 进行通讯，在调用 getGlobal 时，发送一个 `ELECTRON_BROWSER_GLOBAL` 的指令，主线程在接收到对应指令时，再返回 global 中的对应对象。
在源码中我们发现，remote 模块是通过调用 `ipcRendererInternal` 中的 `sendSync` 方法，同步获取主进程中的对象。而其中的 `sendSync` 方法，我们可以从官方文档中查询到：

> **注意:** 发送同步消息将会阻塞整个渲染进程，你应该避免使用这种方式 - 除非你知道你在做什么。  

![RemoteCommunicate.png](https://cdn.iguan7u.cn/image/RemoteCommunicate.png)

到这里，我们可以确定 `sendSync` 方法正是导致渲染进程卡顿的罪魁祸首。

其实细心一点我们可以发现，在 demo 中，主进程返回数据以及渲染进程中接受回调，程序都打印了其时间戳。由此可见，在主进程返回数据时，渲染进程在 330ms 后才收到了具体的返回数据。因此可以得知，一次 remote 模块的跨进程调用，渲染进程会阻塞接近 1/3 秒的时间。

### 2. 应用程序为何会卡顿这么长时间？

由上述原因我们可以得知，在一次 remote 模块的跨进程通讯，渲染进程会卡顿 1/3 秒，这在操作逻辑不频繁的情况下，仍是在可接受范围。可是在我们的测试 demo 中，应用程序的测试 demo 中，我们可以看到窗口阻塞了可以明显感知的时间长度，这里是否还存在其他的问题仍待进一步探究？

我们回归一下测试 demo 的现象：在经过 330ms 后，渲染进程打出了返回数据后的时间戳，因此可以确定，在经过 330ms 后，渲染进程已经完成了主进程方法的调用，并且得到了该方法的返回数据。而在此之后，渲染进程依然处于阻塞的状态，控制台一直在遍历打印返回值的信息，直到信息打印完成后，也就是经过 4508ms 后，渲染进程解除了阻塞的状态，恢复更新计时器的数字。

我们再细看官方文档中的 remote 模块介绍，其中提到：

> Electron 确保只要渲染进程中的远程对象存在（换句话说，没有被垃圾收集），主进程中的相应对象将不会被释放。 当远程对象被垃圾回收后，主进程中的相应对象将被解除引用。  

在主进程返回对象后，为什么依然需要保留对象呢？因此，我们有合理的理由怀疑：在遍历 remote 模块的返回数据时候，仍然存在跨进程通讯的情况。

### 原因

我们细看一下 remote 中 `metaToValue `方法代码，发现渲染进程在接收到 remote 模块返回的对象时并不是解析成真实的对象，而是更接近于主进程中对象的**镜像对象**，所有真正的 value 都是通过发送 rpc 消息到主进程中，主进程再将当前真正的值返回到渲染进程中。

```javascript
// Convert meta data from browser into real value.
function metaToValue (meta) {
	// diffent types of value should use different methods
  const types = {
    value: () => meta.value,
    array: () => meta.members.map((member) => metaToValue(member)),
    buffer: () => bufferUtils.metaToBuffer(meta.value),
    promise: () => resolvePromise({ then: metaToValue(meta.then) }),
    error: () => metaToPlainObject(meta),
    date: () => new Date(meta.value),
    exception: () => { throw errorUtils.deserialize(meta.value) }
  }

  if (meta.type in types) {
    return types[meta.type]()
  } else {
    let ret
    // get remote object from the cache
    ...

    // A shadow class to represent the remote function object.
    if (meta.type === 'function') {
      const remoteFunction = function (...args) {
        let command
        ...
        const obj = ipcRendererInternal.sendSync(command, contextId, meta.id, wrapArgs(args))
        return metaToValue(obj)
      }
      ret = remoteFunction
    } else {
      ret = {}
    }

    setObjectMembers(ret, ret, meta.id, meta.members)
    setObjectPrototype(ret, ret, meta.id, meta.proto)
    Object.defineProperty(ret.constructor, 'name', { value: meta.name })

    // Track delegate obj's lifetime & tell browser to clean up when object is GCed.
    ...

    // set remote object in cache
    remoteObjectCache.set(meta.id, ret)
    return ret
  }
}

```

![RemoteObject.png](https://cdn.iguan7u.cn/image/RemoteObject.png)

我们可以通过图理解一下，左侧是指在渲染进程中取得的 remoteObject、remoteFunction，右侧是指在主进程中对应的真实 obejct 或者 function。当渲染进程中访问其 remoteObject 中的方法或者属性时，内部都会通过 `ipcRendererInternal.sendSync` 同步通知主进程中，根据其中的隐藏字段 `atomId` 在主进程中找到对应的真实对象，获取属性值或者执行其对应方法，然后在通过 `event.returnValue` 返回数据。其内部实现与 remote 执行远程方法基本一致。

所以，我们确定了渲染进程长时间卡顿，是因为我们在渲染进程中遍历 remote 对象中的属性，内部执行大量 `sendSync` 方法导致渲染进程一致处于等待状态，从而阻塞了渲染进程其他任务的执行。

### 3. 那我是否不使用 Remote 就好？

上面所遇到的问题，都是因为业务代码中直接使用 Remote 模块导致性能收到了严重影响，那我们是否只需要改用 IPC 方式调用主进程中的逻辑就好了呢？Chromium IPC 在设计上是使用完全异步的方式，理论上不会出现上述所提及的问题，下面我们来实践一下。
在 demo 中，我们增加一个按钮，使用 IPC 调用主进程中 `blockTheMainProcess` 的方法，模拟主进程在进行 CPU 密集型操作，具体代码如下：
```javascript
function blockTheMainProcess() {
  console.log('now start block the main process')
  while(1) {
    // block main process
  }
}
```

（由于录屏无法正常记录鼠标状态，请读者自行测试）

我们发现，在点击 IPCAction 按钮后，渲染进程的计时器并没有像问题 2 中停止，可是鼠标却变成了 macOS 中应用程序无响应的 Beach Ball，因此即便在不使用 Remote 模块下，主进程的卡顿仍可能导致渲染进程无法响应用户的点击。

### 原因

在操作系统设计中，不同的进程理论上是互不影响的，它们各自获取到的资源都是独立的，那么按照正常理解中，Electron 的主进程阻塞是不应该影响到渲染进程的运行的。可是渲染进程跟主进程并非真正独立的进程，它们同属于一个 Electron 应用，那么我们有合理的怀疑，在 Electron 内部，必定存在内部的进程间通讯，而且这些通讯正是导致渲染进程无法响应的罪魁祸首。

![ElectronSendSync.png](https://cdn.iguan7u.cn/image/electron-sendSync.png)

通过搜索 Electron 源码可以发现，Electron 内部也存在的很多的模块使用了 IPC `sendSync` 的方法，而这些方法每一个都可能导致渲染进程无法响应。

## 解决方案
Electron 的渲染进程以及主进程在内部都使用了 chromium 中的 V8 Javascript 引擎，其执行 JS 效率已经在 node.js 中被充分地证明，因此作为原生应用， JS 执行效率应该不会成为其性能瓶颈。因此我们可以认为，访问 remoteObject 导致渲染进程卡顿执行 Electron 架构设计所限，并非不可绕过的问题。

- **在频繁访问的方法建议使用 ipcRenderer/ipcMain 进行通讯**
由于 Electron 中 ipcRenderer 以及 ipcMain 使用的是 chroumium 中的文件句柄的方式进行跨进程数据传递，该方式在设计上采用的是完全异步的方式，同时性能对比普通的 socket 有明显的优势，因此项目中在主进程以及渲染进程中的大量使用也不会对性能有明显影响。

- **不要在主进程中进行 CPU 密集型操作**
主进程是负责控制应用窗口以及整个应用程序的生命周期，并非处理业务逻辑的。在项目初期我们曾错误认为主进程是作为程序后台去使用的，因此我们将业务代码统一放到主进程中运行，一旦业务逻辑出现需要 CPU 长时间处理的，甚至会导致程序处于无响应状态。因此我们将业务逻辑都转移到了主进程 fork 出来的子进程进行处理，避免了可能出现卡顿的情况。

- **Remote 调用的主进程方法尽量设计为 aync 方法**
上述提到，因为 remote 底层使用的是 `sendSync` 方法，该方法会让渲染进程一直处于阻塞状态直到主进程方法执行完成并返回数据，因此如果主进程中的方法是 IO 操作或者是 CPU 密集型的，则会导致渲染进程一直处于阻塞状态无法处理用户点击事件。而声明为 async 方法，则会让主进程方法立即返回一个 Promise 对象，无需等待方法执行完成，极大地减少了渲染进程的等待时间。

- **需要调用 remote 的主进程方法，不要返回大量数据**
上述第二条问题我们描述，在使用 remote 获取主进程的数据，在渲染进程该对象会处于一种**镜像**的状态，所有的属性获取、变更同样需要跨进程的访问。这里并不仅限于调用主进程同步的方法，同时包括主进程 async 的方法。
在前一个注意点我们提到，remote 调用 async 方法虽然能立即返回 Promise 对象，无需渲染进程等待主进程方法执行完成，可是其执行完的数据仍然会通过 Promise 对象中的 `then` 方法传递执行后的数据。在该数据中，属性值仍然是使用 remote 的机制。
因此大量数据的获取我们推荐使用 ipcRenderer/ipcMain 进行获取。

## 进一步的解决方案 —— “架空”主进程
在我们上面讨论的解决方案中，其实并不能彻底的解决进程阻塞导致的卡顿问题，随着应用规模的增长，主进程最终会承担更多的业务逻辑，而不可避免的会导致在某个场景下出现 CPU 密集型的操作。这个时候，探讨将 CPU 密集性任务分散开，是不太现实的。我们开始设想，能否 “架空” 主进程，让它闲下来呢？
在一般的客户端开发中，如果进程任务过于繁重，都会通过多开线程的方式减少进程阻塞的场景，而 node.js 也意识到单线程所带来的局限性，在后面的版本引入了 cluster 的功能模块，为 node.js 带来的负载均衡的子进程特性。而其后甚至在最近的 10.5.0 版本引入 worker_threads 的实验性模块，为 node.js 带来的真正的多线程。

### worker_threads

其实作为客户端开发，worker_threads 应该是这个问题的完美解，多线程的支持让 Electron 能更加接近原生应用。可是 worker_threads 模块目前仍然在实验性阶段，官方并不推荐在生产环境中使用，甚至后期不排除将其移除。而 Electron 官方文档中提及到的多线程，也是较为模糊的态度：

> 在 Web Workers 里可以直接加载任何原生 Node.js 模块，但不推荐这样做。 大多数现存的原生模块是在假设单线程环境的情况下编写的，如果把它们用在Web Workers 里会导致崩溃和内存损坏。  

### cluster

cluster 为 node.js 提供了子进程的功能。使用 cluster，可以让多个子进程使用同一个端口，并且为其提供了负载均衡的特性，使得 node.js 能充分利用多核 CPU 的特性。可是在我们的目标中，我们并不是像替主进程减负，而是想彻底地让主进程闲下来。

### child_process

我们最终使用了 child_process 模块。child_process 模块为 node.js 提供了原始的子进程方式。通过测试我们发现，在子进程中，即使长时间运行 CPU 密集型的操作，渲染进程以及主进程都不会受到应用。经过一系列的重构，我们将绝大部分业务逻辑转移到子进程中，真正彻底地解决了 CPU 密集型运算导致的渲染进程卡顿的问题。


## 总结
Electron 的应用架构不同于普通的原生应用，其多进程的设计在某些方面可能还优于一般的架构设计，可是在许多的技术细节中， Electron 的官方文档并没有详细说明其中的原理以及副作用。正如 remote 通讯方式虽然给 Electron 提供了极其便利的 rpc 通讯方式，可是使用者仍然需要深入了解 remote 的运行原理以及机制，需要清楚意识到，该通讯方式在某些情况下可能会对程序体验造成毁灭性的打击。因此我们需要慎重选择使用跨进程通讯。

#blog
