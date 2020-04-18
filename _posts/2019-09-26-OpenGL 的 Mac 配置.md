---
title: OpenGL 的 Mac 配置
key: OpenGL 的 Mac 配置(2019-09-26)
tags:
- OpenGL
---

# 前言

- 记录自己在Mac上的配置的实践过程。（系统macOS Mojave 版本 10.14.6）
- OpenGL ES 或者说计算机图形学的学习，是离不开图形学基础的，而图形学经典书籍关联的都是OpenGL操作的，出于此，就需要在Mac上配置OpenGL环境。

# 配置流程

## First

- Mac 电脑。
- 安装 Xcode （后续开发OpenGL使用）。

## CMake

1. 检查是否配置了 cmake ——打开终端命令行输入 cmake 后回车，如果提示 “cmake: command not found”，则进行一下步骤安装，如果已安装则跳过此小段。
2. 前往 [cmake 官方下载](https://cmake.org/download/)下载 CMake dmg，要下载Platform 为 Mac 的。如 图-1：
3. 安装 CMake dmg。安装后，打开CMake，点击左上的工具栏 “Tool”可看到，选择“How to install for command line use”，出现如 图-2。
4. 为了配置为默认路径/usr/local/bin，按照上图在命令行中输入sudo "/Applications/CMake.app/Contents/bin/cmake-gui" --install
5. 等命令行 CMake 安装进度结束（可能需要等一段时间），再次在终端命令行中输入 cmake 回车后，如 图-3，则表示 cmake 配置完毕。

<center>
<div>图-1</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/1a8b8d56e92b231a1780cba6ed65ddd1.png" />
</center>

<center>
<div>图-2</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/01594f82c519368372d36984616667b1.png" />
</center>


<center>
<div>图-3</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/2510de9cf7ad9b95eade3f6df91f81b6.png" />
</center>


## GLFW

1. 下载官方[GLFW资源](https://www.glfw.org/download.html)。
2. 如图-4，切记一定不要漏了下载第二个标红资源。否则后续会造成找不到 libglfw.3.dylib 库资源。
3. 下载完图-4 中 第一个标红资源后解压，命令行cd 到解压文件夹下
   - 执行 "cmake ." （不包含引号，即cmake+空格+小点）
   - 执行 sudo make install 
4. 执行完毕后（耐心等待命令执行完成），这时头文件和库文件分别被安装到/usr/local/include和/usr/local/lib下。
5. 打开Finder，快捷键 Shift + Cmd + G，输入 /usr/local/lib/，可看到相应的文件。
6. 注意，Attention ！此时重要的一步骤，将图-4标红第二文件资源解压后，文件夹下的 libglfw.3.dylib 库移入 上面第5部文件中，完成后如图-6。

<center>
<div>图-4</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/3ddb02ed25e056f69c298ee606b8be25.png" />
</center>


<center>
<div>图-5</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/f0d10b5f102397f1f26680f3d07737e8.png" />
</center>


<center>
<div>图-6</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/8a0cc6ba54956a9ed042044a67b7fc83.png" />
</center>


## Xcode Project Config

1. 新建项目时，选择 Command Line Tool项目。（图-7）
2. 选择 C 或 C++ 语言。（图-8）
3. Buid Phases 下添加刚才生成的库 libglfw.3.dylib。（图-9 至 图-13）
4. Buid Settings 下配置文件路径：（如图-14）
   - Header Search Paths 添加 /usr/local/include
   - Library Search Paths 添加 /usr/local/lib


<center>
<div>图-7</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/b6c480b635861dd766fb93fd24255823.png" />
</center>


<center>
<div>图-8</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/0d1541988d2f60634981603a3a8a7fc8.png" />
</center>

<center>
<div>图-9</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/92d2c7d1684f704bd83a94ea2b58fae8.png" />
</center>


<center>
<div>图-10</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/9968bf673567053024e46ae586b75eb4.png" />
</center>

<center>
<div>图-11</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/1925ff23d07633c285e314c138a26544.png" />
</center>

<center>
<div>图-12</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/687cb4b17999a4fa9eb9dc9a4181d89e.png" />
</center>



<center>
<div>图-13</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/96c055c50dbe777b9169268b1b8d54ad.png" />
</center>



<center>
<div>图-14</div>
<img src="https://images.xiaozhuanlan.com/photo/2019/29f494975ae1856aa7468e2f4c775035.png" />
</center>


## Attention ! 

1. 在前面 GLFW 项的第 3 步时候，如遇到提示 Could NOT find Doxygen 。
   - 可以执行 brew install doxygen。（前提是安装了 brew ，这里不赘述了）。
   - 上面命令执行，等待安装完成，重新执行 GLFW 的第 3 步骤。
2. 如遇到 library not found for xxx 
   - 检查是否忘记执行 GLFW 中 的第 6 步，或者执行第 6 步时，放错了文件夹。
   - 按照上面 “Xcode 工程配置” 步骤检查 3、4 步。
3. 如果在下面Test 代码执行过程中遇到类似提示：“libglfw.3.dylib 不是Mac信任库”，可参照普通dmg软件安装后打不开的解决方式，去设置里面更改权限“允许从以下下位置下载的APP:”。

## Test

以上步骤完成后就算完成了，接下来就可以开发了。

将以下代码 Copy 下来 粘贴在 main.c 文件里，运行一下，出现一个命令行终端 红的纯色屏幕。Congratulation ！You get it !

```
#include <GLFW/glfw3.h>
#include <OpenGL/gl3.h>



static void error_callback(int error, const char* description)
{
    fprintf(stderr, "Error: %s\n", description);
}

static void key_callback(GLFWwindow* window, int key, int scancode, int action, int mods)
{
    if (key == GLFW_KEY_ESCAPE && action == GLFW_PRESS)
        glfwSetWindowShouldClose(window, GLFW_TRUE);
}

void init()
{
    //初始化操作
}

void display()
{
    glClear(GL_COLOR_BUFFER_BIT);
    
    //绘制代码
    
    glFlush();
}

int main(int argc, const char * argv[])
{
    GLFWwindow* window;
    
    if (!glfwInit())
        exit(1);
    
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 4);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 1);
    
    window = glfwCreateWindow(640, 480, "Demo", NULL, NULL);
    if (!window)
    {
        glfwTerminate();
        exit(1);
    }
    
    glfwSetErrorCallback(error_callback);
    
    glfwSetKeyCallback(window, key_callback);
    
    glfwMakeContextCurrent(window);
    glfwSwapInterval(1);
    
    glClearColor(1.f, 0.f, 0.f, 1.f);
    
    init();
    
    while (!glfwWindowShouldClose(window))
    {
        display();
        
        glfwSwapBuffers(window);
        glfwPollEvents();
    }
    
    glfwDestroyWindow(window);
    
    glfwTerminate();
    exit(0);
    
    return 0;
}

```

## Feature

感兴趣的话可以从[OpenGL 简化中文版网站](https://learnopengl-cn.readthedocs.io/zh/latest/)开始学习联系小demo。

# 参考链接
- [Mac下OpenGL环境的配置](https://blog.csdn.net/weixin_33972120/article/details/82085792)
- [搭建Mac OpenGL开发环境](https://www.jianshu.com/p/891d630e30af)
- [CMake](https://cmake.org/download/)
- [GLFW](https://www.glfw.org/download.html)
- [OpenGL 简化中文版网站](https://learnopengl-cn.readthedocs.io/zh/latest/)
