---
title: 越狱手机Reveal的动态库加载流程
tags:
- Reveal
- LLDB
---

## 结论

Reveal动态库加载流程如下图：

<img src ="/assets/article/reveallibload.png" width="1000" height="551" />

1. 红色'dlopen'箭头代表：左边被加载的库内部会调用dlopen加载箭头右边的库。
2. 箭头1号相继dlopen了右边的两个库。
3. substitute-inserter.dylib、libsubstitute.dylib、substitute-loader.dylib属于越狱后系统所带的库，可以从Cydia中看到。
4. RHRevealLoader.dylib、libReveal.dylib属于Reveal的库。

## 背景

越狱手机配置Reveal后，可以查看手机所安装的App的界面结构如下图：

<center>
<img src ="/assets/article/revealcheck.png" width="1000" height="728" />
</center>
这里有一个困扰我的问题：Reveal能够查看越狱手机所安装的所有App运行起时界面结构，它是如何做到的呢？

**推测其运作流程**

1. 手机启动后加载Reveal的动态库。
2. 手机Reveal库相关代码绘制页面bitmap。
3. 从手机端向Mac传输渲染信息
4. Mac端Reveal显示。

其中第2步即是手机app图层中各个view的bitmap，大概率是draw，或着纯色不绘制等技术实现。

第3步，可以选择USB/WIFI。

第4步就是Mac端对传输过来的信息进行整合渲染。

**本篇文章主要探讨第1步**——reveal的动态库是如何在越狱手机上各个App手机上加在的？







## 探究越狱手机端Reveal的动态库加载流程
### 查看App的动态库列表

1. MAC连接手机，USB映射通信接口。
2. LLDB连接调试指定APP（极少数安全防护比较高的app会阻止ptrace，可以通过调试手段绕过)。
3. 查看APP运行时链接接的库。

<img src ="/assets/article/revealImageList.png" width="800" height="135" />

既然其中有三方动体库加载，那这些动态库能够被加载进来，这里推测出于以下2个原因：

1. 静态修改：Mach-o文件头文件被修改，添加了依赖动态库。
2. 动态修改：App时候设置了DYLD_INSERT_LIBRARY 环境变量。

### LLDB链接越狱记被调试应用
首先端口映射，然后使用LLDB与debugserver命令链接越狱手机的相应app，使得断点停在__dyld_start。具体如下图：

<img src ="/assets/article/RevealDebugserver.png" width="1000" height="114" />
<img src ="/assets/article/RevealdyldStart.png" width="500" height="186" />
### 查看DYLD_INSERT_LIBRARIES环境变量的值
* 根据dyld的加载流程，紧接目前程序暂停的位置，搜索符号“dyld`**ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&)**“。

<img src ="/assets/article/RevealProcess.png" width="1000" height="207" />

* 触发断点后，再次出查看内存中映射的module。下图同时可以看出，内存中加载的module绝大部分是DSC（共享缓存库）。

<img src ="/assets/article/RevealImageList1.png" width="1000" height="203" />
<img src ="/assets/article/RevealModules.png" width="1000" height="173" />

* 查找‘getenv‘符号并在此处设置断点，触发条件设置为第一个参数等于“DYLD_INSERT_LIBRARIES”，并打印第一个参数。触发断点后，使用‘finish’命令，然后查看‘getenv’的返回值DYLD_INSERT_LIBRARIES = /usr/lib/substitute-inserter.dylib，该库是越狱系统带的。

<img src ="/assets/article/RevealInsertLi.png" width="1000" height="541" />

### 设置dlopen 断点查看Reveal与越狱系统Substitute库加载过程
* 为dlopen设置断点，触发条件设置为当第一个参数中包含‘Reveal’或‘substitute’。

<img src ="/assets/article/RevealBdlopen.png" width="1000" height="220" />

* 第一次触发的是‘substitute-insert.dylib’触发open ‘libsubstitute.dylib’。

<img src ="/assets/article/Revealsubstitue.png" width="1000" height="532" />

* 第二次触发的是‘substitute-insert.dylib’触发open ‘substitute-loader.dylib’。

<img src ="/assets/article/Revealsubload.png" width="1000" height="531" />

* 第三次触发的是‘substitute-insert.dylib’触发open ‘substitute-loader.dylib’。

<img src ="/assets/article/RevealRloader.png" width="1000" height="606" />

* 第四次触发的是‘substitute-insert.dylib’触发open ‘substitute-loader.dylib’。

<img src ="/assets/article/ReveallibR.png" width="1000" height="568" />
## 越狱手机Reveal配置
1. [手机越狱工具un0ver](https://unc0ver.dev/)，截止写该文章时可以越狱iOS11.0-14.8.
2. Mac安装Reveal.
3. Mac下载ifunBox (用于查看越狱手机文件树，拖拽Mac/iphone文件交换）。
4. iphone越狱后Cydia搜索Reveal Loader，并在设置里面打开需要观察的App。
5. Cydia下载OpenSSH。
6. Cydia下载 "Apple File Conduit 2"。
7. 将Mac端的Reveal库（ Help-->Show Reveal Library in Finder-->iOS Library）framework包内RevealServer改名libReveal.dylib，迁移进iphone的/Library/RHRevealLoader/ 文件夹下（如果没有就创建文件夹）。
8. iphone设置内Reveal Loader，Enable需要观察的App。
