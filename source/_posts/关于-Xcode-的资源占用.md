---
title: 关于 Xcode 的资源占用
date: 2021-03-30 17:39:12
tags:
---

> macOS: Big Sur 11.2.1 
> Xcode: 12.4 (12D4e)

## TL；DR
为了安全起见，除了 `~/Libray/Developer/Xcode/iOS DeviceSupport` 文件夹不建议删除外，`~/Libray/Developer/` 内的其他内容，都可以删除。

## 前言：
由于笔者一直在使用着 256G 的 macBook Pro，因此一直对文件存储表示相当的关注。最近发现，即便工作机没有存放任何私人内容，在 iOS debug 过程中依然会经常出现磁盘容量不足的提示，甚至会影响 Module Cache 文件的写入，因此导致构建失败。而出现这种情况后，笔者习惯进入 `Developer` 文件夹，进行一通操作，释放大量磁盘空间，然后再次进入漫长的等待构建过程。
虽然操作了很多次，可是一致没搞清楚 Xcode 对于缓存文件的管理，因此趁着这个机会，深入了解一下这个烦困依旧的问题。

## Developer 文件夹初探
在笔者所用的开发环境中，Xcode 生成的临时文件绝大部分都集中在 `~/Library/Developer` 。当然，如果你直接删除这个文件夹，不过可能会有一些难以预料的问题会出现。
Developer 文件夹中包含了 CoreSimulator, Xcode, 以及 XCTestDevices 文件夹，不同的文件夹中存放的数据，都是各有作用的。以笔者开发环境为例，Developer 文件夹中的目录结构如下：

```
.
├── CoreSimulator
│   ├── Caches
│   │   └── dyld
│   ├── Devices
│   │   ├── 34EF2726-B835-4ADA-B589-A7E71620D7B6
│   │   └── device_set.plist
│   └── Temp
│       └── BackgroundDelete
├── XCTestDevices
└── Xcode
    ├── DerivedData
    │   ├── ModuleCache.noindex
    │   └── xxxxxx-hgprucnfscdzudfulchfczkhsmjy
    ├── DocumentationCache
    │   └── v178
    ├── GPUToolsAgent.sock
    ├── UserData
    │   ├── IB\ Support
    │   ├── IDEEditorInteractivityHistory
    │   ├── IDEPreferencesController.xcuserstate
    │   └── KeyBindings
    ├── iOS\ Device\ Logs
    │   ├── iOS\ Device\ Logs\ 12.4.db
    │   ├── iOS\ Device\ Logs\ 12.4.db-shm
    │   └── iOS\ Device\ Logs\ 12.4.db-wal
    └── iOS\ DeviceSupport
        └── 14.4\ (18D52)\ arm64e
```

## Developer 文件夹细看
### CoreSimulator
在 macOS 环境的模拟器环境，其中 `Caches/dyld` 文件夹会生成开发调试相关的工具文件，详细可以参考这篇文档 [dyld 详解](https://www.dllhook.com/post/238.html)。
而 `Devices` 文件夹则是针对每一个模拟器的 iOS 系统生成一个用户层的沙盒环境，在 iPhoneX 跟 iPhoneXs 的两个模拟器中，其用户环境是不一致的，Xcode 会在这个目录分别为两个模拟器生成不同的文件夹。这个文件夹包含了开发者在对应模拟器环境中的 Debug 应用，以及应用在 iOS 系统中的沙盒环境，如果在很长时间没有清理的话，很有可能会产生非常可观的应用垃圾。如果我们手动清理这个目录，那么 Simulator.app 在下次启动的时候会重新将其生成，但这个时候之前应用会以全新安装的形式进行启动。

### XCTestDevices
Xcode 自动化测试以及单元测试相关的目录，如果经常进行对应操作的话，这个目录也会产生相当可观的应用垃圾，建议经常清理。

### Xcode
这个目录下保存了开发过程中可能产生的缓存文件。其中 `DerivedData` 目录保存了项目构建过程中的中间文件。

#### DerivedData

![ModuleCache.onindex](https://cdn.iguan7u.cn/image/ModuleCache.png)
`DerivedData/ModuleCache.noindex` 是保存应用模块的预编译文件 Precompile module files (`.pcm`)，用于解决 Xcode 项目中可能出现数以万计的 `#import` 语句的构建问题，模块化拆分后，Xcode 能增量构建变更模块，极大的提升了开发效率。如果你删除了这个文件夹，下次 Xcode 将可能会出现全量构建的问题。

![ProjectFolder](https://cdn.iguan7u.cn/image/Xcode-ProjectModule.png)
`DerivedData/[项目名][乱码后缀]` 文件夹，记录的是对应项目产生的中间文件。Xcode 会根据不同的项目分别产生对应的文件夹。如果项目比较庞大，这里生成的中间文件会非常多。笔者正在参与的项目，会固定产生 20GB+ 的中间文件。

**index**文件夹
存放的是 Xcode 对于项目索引的信息，用于方法跳转，文件定位等地方，删除后 Xcode 将会重新生成。据说 Xcode9 以前的版本 index 是以可肉眼查看的格式存放的，后面改为用 [LMDB](https://en.wikipedia.org/wiki/Lightning_Memory-Mapped_Database) 格式存放了，估计是为了解决内存占用问题。

**logs**文件夹
Xcode 相关日志存放位置，内部将应用开发过程中每个过程都单独存放在不同的文件夹中，查看可以了解 Xcode 的不同阶段执行的动作。更多信息可以查看这里  [Test logs in Xcode](https://michele.io/test-logs-in-Xcode/) .

**SourcePackages**文件夹
暂时未知具体作用，可能跟 [Swift Package Manager](https://swift.org/package-manager/) 有关。

**TextIndex**文件夹
具体作用未知，名字可能跟字符索引相关？

**Build/Products**文件夹
存放 debug 或者 release 模式下构建的应用包以及对应的资源库。这里笔者好奇为什么不能在 Devices 文件夹中软链接过来这里的资源包，节省磁盘空间占用，也避免内容拷贝（笔者实测这种方式能正常打开应用）。。。当然，具体原因我们也不可而知了。

**Build/Intermediates.noindex**文件夹
这里存放项目代码构建的中间文件，非常庞大，需要定时清理。清理后可能会导致应用全量构建。

#### DocumentationCache
这个文件夹记录 Xcode 获取对应文档的缓存。

#### iOS Device Logs
这个文件夹记录每个模拟器设备产生的日志，看文件内容应该是 CoreData 方式保存的数据。

#### iOS DeviceSupport
这个文件夹**尤为重要**，其中保存了 Xcode 系统版本支持信息。如果是从低版本 Xcode 一直升级上来的，这里可能会保存了很多低版本的系统支持，我们可以根据自身实际的需求，删减其中的内容。如果错误删除了，或者后期需要，可以在 Xcode Preference 中的 Components 面板中重新下载。

#### UserData
故名思义，用户数据文件夹。具体存放的内容与 Devices 文件夹中存放的有所重复，具体逻辑也不得而知了。