---
title: OpenGL-ES Check List
tags: []
---

## 前言

- 概括性的记述OpenGL ES 的渲染流程，简明性的列出《OpenGL ES 3.0编程指南》中提到需要注意与可优化的点。
- 从iOS平台切入开展，内容会不断更进。
- 目的是为了便于脱离书，快速回顾要点与书中指出的优化点。

## 渲染管线以及优化点

### 渲染管线图

<center>
<img src ="/assets/article/OpenGL-ES-pipelining.png" width="424" height="300.0" />
</center>

### 上下文管理

```
创建上下文
EAGLContext *eglContext = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES3];

切换上下文
[EAGLContext setCurrentContext:eglContext];

删除当前上下文
if ([EAGLContext currentContext] == self.context) {
		 [EAGLContext setCurrentContext:nil];
}
self.context = nil;
```

### 渲染管线内容要点

1. 创建顶点、片段shader

   - Shander 内部注意使用次数较多的同一个数字，需要定义为常数变量。（例如：数字1.0被使用N次，将会被统计为N个统一变量消耗N个存储单元）。

2. 根据顶点、片段着色器创建Program。

   - 顶点着色器进行 MVP 坐标变换。
   - 顶点着色器也可以结合变换反馈做GPU动画。
   - 顶点着色器也可以进行逐顶点光照，然后插值。
   - 插值有两种：平滑、线性。
   - 大部分图像处理（逐片段处理）在片段着色器内进行。
   - 片段着色器进行逐片段照明更好一些，但是计算量级比较大。

3. 顶点属性（客户端数组、VBO、VAO）

   - 常量顶点属性。

   - VBO。（相比较于客户端数组：避免每次调用绘制程序从客户端拷贝进GPU）
   - VAO，VAO对象绑定后，后续执行顶点属性的操作状态将被保存在VAO对象内。（可以在初始化/绘制前某个阶段 将 VBO 状态 配置好，在绘制的时候仅需绑定VAO即可。）

4. VBO 装填数据的 3 种可选方式

   - 使用 glBufferData 直接更新。
   - 使用 glMapBufferRange 映射缓冲区对象。（优点：映射缓冲区可以减少程序内存占用；共享内存架构下，映射缓冲区将直接返回GPU存储缓冲区地址空间的直接指针。）
   - glCopyBufferSubData 将数据从一个缓冲区对象拷贝到另一个缓冲区对象。

5. 绘制图元

   - glDrawArrays
   - glDrawElements（尽量用此替代DrawArrays，例如顶点重用机制将会占用更少的内存。书中提到的“诸多原因”后续待调研。
   - glDrawElementsInstanced（高效的几何形状实例化，调用一次渲染多个具有不同属性（变换矩阵/颜色/大小）的对象，实例化API降低了想 OpenGL ES 引擎发送 许多绘图API 的CPU开销。

6. 光栅化之前需要指定要提出的面（默认情况下，剔除是被禁用的。）

   - 剔除尽量要开启，避免GPU浪费时间去光栅化不可见的三角形。
   - 启用剔除可以改善 ES 的性能。

7. 指定统一变量。

8. 纹理

   - glTexImageXXX 上传纹理时候 会根据glPixelStorei设置的解包对齐方式。
   - mip 贴图 可以采用自定义链（需要客户端设置生成），相对灵活；或采用系统glGenerateMipmap ，将会由GPU 来生成，（可以避免从客户端来回传递，节省带宽和内存。）
   - 使用mip 贴图，过滤方式则可以选择三线性过滤。
   - 纹理格式有规范化格式（从整型转为0-1.0 / -1.0 - 1.0），浮点纹理格式，整数纹理格式，sRGB纹理格式
   - sRGB 是非线性颜色空间。（需要使用 sRGB 纹理格式接收存储，在读入着色器时会转换到线性空间计算，然后被渲染到 sRGB 空间将会被转换到非线性空间。）

9. 加载纹理的方式

   - glTexImageXXX : 从客户端加载纹理，或者设定为NULL。
   - gReadBuffer ： 从颜色缓冲区/附着 复制数据到一个纹理，后续需要调用glCopyTexImageXXX装填texture 。（此种方法从颜色缓冲区复制到纹理，相对不高效）
   - 位块传输。相对高校
   - glTextStoreXX ：ES 3.0 推荐使用不可变纹理。（原因在于OpenGL ES 驱动程序在绘图之前无法确定纹理是否已经完全指定，会检查每个mip 贴图级别或者子图像是否相符，每个级别的大小是否正确以及是否有足够的内存；不可变纹理可以避免这些校验。）

10. 采样器对象。（纹理需要设置采样与过滤模式等）

    - glGenSamplers 

    - 应用程序经常在大量纹理上使用相同的设置。在这种情况下，用glTexParameter 为每个纹理对象设置采样器状态将会带来大量额外的开销。

11. 片段

    - 片段着色器必须指定数据精度 highp/mediump/lowp
    - Alpha 测试使用 discard 字段抛弃片段。

12. 缓冲区深度是指的该缓冲区单个元素所占的位数（例如3个颜色分量，每个分量占用8位，深度是3 * 8 = 24）。

13. 深度、模版测试 （生产mask 的效果）。

14. 混合。（片段颜色 与 缓冲区内颜色的混合公式）

15. 抖动。

16. 多重采样。

17. 使用 glReadPixels 帧缓冲区颜色缓冲区中读取与写入像素 

    - 将从当前绑定的帧缓冲区对象的颜色缓冲区 读取到客户端 或者VBO（下面）

    - 最好依赖返回的像素数据类型 （通过 glGetIntegerv ( GL_IMPLEMENTATION_COLOR_READ_TYPE, &readType ) ）
    - 当使用glBindBuffer将有效的缓冲区对象绑定到 GL_PIXEL_PACK_BUFFER后，调用glReadPixels 将会启动DMA传输并像素写入该缓冲区对象。

18. 多重渲染目标

    - 需要调用glDrawBuffers 指定渲染的颜色附着。

### 帧缓冲区对象

1. 渲染到纹理方式的对比，
   - 上文提到的 gReadBuffer。此方法执行从帧缓冲区 copy 到纹理缓冲区，往往对性能有不利影响，且纹理尺寸需要 <= 缓冲区尺寸。
   - 通过使用连接到纹理的PbBuffer。此方法在于窗口系统提供的绘制表面进行上下文的切换，可能会带来气泡效应。
   - 帧缓冲区对象作为绘制目标。帧缓冲区对象的可以将纹理作为颜色/深度模版缓冲区 附着；且多个帧缓冲区之间可以共享颜色/深度模版附着。
2. 渲染缓冲区与纹理 （在作为附着方面的对比）
   - 渲染缓冲区支持多重采样。
   - 如果图像没有被当作纹理使用，则使用渲染缓冲区可能带来性能上的好处（因为 ES 将会以更高效的格式存储渲染缓冲区）
   - ps：由此可见如果我们仅仅想作为纹理使用的时候需要选择 纹理作为附着。
3. 帧缓冲区对象与 系统窗口表面的对比
   - 窗口支持像素归属测试。
   - 窗口支持双缓冲区。
   - 帧缓冲区的优势在于，它可以与其他帧缓冲区共享深度/模版缓冲区。
4. 帧缓冲区完整性检查 （glCheckFrameBufferStatus）
   - 确保帧缓冲区至少有一个有效的附着。
   - 相关有效附着必须具有相同的宽高。
   - 如果存在深度和模版附着，必须是相同的对象。
   - 所有渲染缓冲区附着的 GL_RENDERBUFFER_SAMPLES 必须相同。如果附着是渲染缓冲区和纹理的组合，则GL_RENDERBUFFER_SAMPLES 为0。
5. 位块传输（glBlitFrameBuffer）高效的将一个帧缓冲区矩形区域像素复制到另一个帧缓冲区。
6. 帧缓冲区对象属于重型对象，可进行享元。

### 同步与栅栏

1. glFenceSync : 发送同步信号。
2. glClientWaitSync : 阻塞客户端的信号等待。
3. glWaitSync : GPU信号阻塞等待。（ps : 不阻塞客户端）
