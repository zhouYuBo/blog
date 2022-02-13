---
title: LLDB——Symbolication
tags:
- LLDB
- LLDB官方文档笔记
---

## Crash logs符号化的方式

1. Xcode 自动化解析。
2. 命令行工具symbolicatecrash + 对应UUID的dSYM。
3. LLDB命令/脚本。

&#8195;&#8195;绝大多数情况下我们会使用第二种方式，因为公司每个app发布的历史版本都保留着对应的可执行文件与dSYM，出现线上闪退/常卡顿上报，都可以通过这种方式解决。那我们还需要LLDB命令/脚本进行符号化么？会，这里列举一些场景：

- 特殊情况下，某些测试企业包可能很难以找到对应的dSYM文件。这个时候我们可以查看Crash logs日志文件对应各个库的UUID，并从已有的dSYM文件中检索，分别找到各个库UUID对应的dSYM文件，逐个映射。
- 制作专属的解析工具。
- 挖掘更细的信息，比如闪退时候的堆栈汇编、寄存器信息（可以通过LLDB辅助查看）。

以下是对官方的LLDB符号化文档的转译笔记。

## LLDB 创建Target

&#8195;&#8195;LLDB 是一个包含调试以及command解释器功能的动态库。LLDB可以用来符号化crash logs，相比于其他符号化工具，能够提供更多的信息：

- Inlined functions （内联函数）
- 地址所在作用域相关的变量、变量所在的位置（所使用寄存器/栈区偏移量）

符号化最简单的加载可执行Mach-O方式：

```
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out
```

通过使用‘--no-dependents’ flag，可以不加载可执行文件所依赖的其他所有动态库。在不使用--no-dependents的情况下，可执行文件所依赖的/usr/lib路径下的动态库将会被加载进来，而此路径下的动态库可能与闪退日志中的版本号不同。

‘image list’ 命令会列出当前target所关联的动态库。

```
(lldb) image list
[  0] 73431214-6B76-3489-9557-5075F03E36B4 0x0000000100000000 /tmp/a.out
      /tmp/a.out.dSYM/Contents/Resources/DWARF/a.out
```

在target中查找某个地址相关的信息：

```
(lldb) image lookup --address 0x100000aa3
      Address: a.out[0x0000000100000aa3] (a.out.__TEXT.__text + 131)
      Summary: a.out`main + 67 at main.c:13
```

由于没有为指定偏移，也没有为sections指定偏移地址。此处所使用的地址是文件相对便宜地址。

在‘target create’不使用‘--no-dependents’ flag情况下，搜索某个地址将会匹配多个动态库的虚拟地址偏移量：

```
(lldb) image lookup -a 0x1000
      Address: a.out[0x0000000000001000] (a.out.__PAGEZERO + 4096)

      Address: libsystem_c.dylib[0x0000000000001000] (libsystem_c.dylib.__TEXT.__text + 928)
      Summary: libsystem_c.dylib`mcount + 9

      Address: libsystem_dnssd.dylib[0x0000000000001000] (libsystem_dnssd.dylib.__TEXT.__text + 456)
      Summary: libsystem_dnssd.dylib`ConvertHeaderBytes + 38

      Address: libsystem_kernel.dylib[0x0000000000001000] (libsystem_kernel.dylib.__TEXT.__text + 1116)
      Summary: libsystem_kernel.dylib`clock_get_time + 102
...
```

为此可以使用动态库名以限定动态库的范围：

```
(lldb) image lookup -a 0x1000 a.out
      Address: a.out[0x0000000000001000] (a.out.__PAGEZERO + 4096)
```

## 为Sections设置加载地址

&#8195;&#8195;当符号化crash log的时候，需要将crashlog-address转换为库符号所对应的地址。为了避免逐个计算映射，我们可以为target中的modules的每个区设置加载地址；也可以为所有的sections设置相同的偏移量slide。‘target modules load --slide’命令为所有sections设置偏移量。

```
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out
(lldb) target modules load --file a.out --slide 0x123000
```

&#8195;&#8195;通常符号化的时候需要为每个区设置不同的加载地址。Crash logs文件中 ‘Binary Images’区会显示所有动态库中\_\_TEXT segment'的加载地址，通过‘target modules load section address’命令为每个\_\_Text segment 指定各自的加载地址，可以省去很多地址映射的计算，并且指定加载地址后image lookup命令依旧可以匹配查找地址。

```
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out
(lldb) target modules load --file a.out __TEXT 0x100123000
```

如上讲 __TEXT segment 指定的加载地址为 0x100123000，可以通过image lookup命令验证下：

```
(lldb) image lookup --address 0x100123aa3
      Address: a.out[0x0000000100000aa3] (a.out.__TEXT.__text + 131)
      Summary: a.out`main + 67 at main.c:13
```

如上验证结果地址100000aa3 + 0x100123000 = 0x100123aa3。

## 加载多个库

通常crash log文件中会涉及到许多动态库的符号，这时我们需要为主程序/一个动态库创建target，然后通过'target modules add'为target添加多个modules。

```
Binary Images:
   0x100000000 -    0x100000ff7 <A866975B-CA1E-3649-98D0-6C5FAA444ECF> /tmp/a.out
0x7fff83f32000 - 0x7fff83ffefe7 <8CBCF9B9-EBB7-365E-A3FF-2F3850763C6B> /usr/lib/system/libsystem_c.dylib
0x7fff883db000 - 0x7fff883e3ff7 <62AA0B84-188A-348B-8F9E-3E2DB08DB93C> /usr/lib/system/libsystem_dnssd.dylib
0x7fff8c0dc000 - 0x7fff8c0f7ff7 <C0535565-35D1-31A7-A744-63D9F10F12A4> /usr/lib/system/libsystem_kernel.dylib
```

创建一个target并添加多个modules:

```
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out
(lldb) target modules add /usr/lib/system/libsystem_c.dylib
(lldb) target modules add /usr/lib/system/libsystem_dnssd.dylib
(lldb) target modules add /usr/lib/system/libsystem_kernel.dylib
```

如果dSYM文件是单独分开的，需要通过--symfile参数指定下：

```
(lldb) target create --no-dependents --arch x86_64 /tmp/a.out --symfile /tmp/a.out.dSYM
(lldb) target modules add /usr/lib/system/libsystem_c.dylib --symfile /build/server/a/libsystem_c.dylib.dSYM
(lldb) target modules add /usr/lib/system/libsystem_dnssd.dylib --symfile /build/server/b/libsystem_dnssd.dylib.dSYM
(lldb) target modules add /usr/lib/system/libsystem_kernel.dylib --symfile /build/server/c/libsystem_kernel.dylib.dSYM
```

按照Binary Images中每个库对应的加载地址分别为各个库设置加载地址：

```
(lldb) target modules load --file a.out 0x100000000
(lldb) target modules load --file libsystem_c.dylib 0x7fff83f32000
(lldb) target modules load --file libsystem_dnssd.dylib 0x7fff883db000
(lldb) target modules load --file libsystem_kernel.dylib 0x7fff8c0dc000
```

然后就可以通过‘image lookup’命令调用栈中的符号地址查找对应的信息了。

```
Thread 0 Crashed:: Dispatch queue: com.apple.main-thread
0   libsystem_kernel.dylib           0x00007fff8a1e6d46 __kill + 10
1   libsystem_c.dylib                0x00007fff84597df0 abort + 177
2   libsystem_c.dylib                0x00007fff84598e2a __assert_rtn + 146
3   a.out                            0x0000000100000f46 main + 70
4   libdyld.dylib                    0x00007fff8c4197e1 start + 1
......

(lldb) image lookup -a 0x00007fff8a1e6d46
(lldb) image lookup -a 0x00007fff84597df0
(lldb) image lookup -a 0x00007fff84598e2a
(lldb) image lookup -a 0x0000000100000f46
```

通过image lookup --address 添加--verbose可以查看地址的详细信息：

```
(lldb) image lookup --address 0x100123aa3 --verbose
      Address: a.out[0x0000000100000aa3] (a.out.__TEXT.__text + 110)
      Summary: a.out`main + 50 at main.c:13
      Module: file = "/tmp/a.out", arch = "x86_64"
CompileUnit: id = {0x00000000}, file = "/tmp/main.c", language = "ISO C:1999"
   Function: id = {0x0000004f}, name = "main", range = [0x0000000100000bc0-0x0000000100000dc9)
   FuncType: id = {0x0000004f}, decl = main.c:9, compiler_type = "int (int, const char **, const char **, const char **)"
     Blocks: id = {0x0000004f}, range = [0x100000bc0-0x100000dc9)
             id = {0x000000ae}, range = [0x100000bf2-0x100000dc4)
   LineEntry: [0x0000000100000bf2-0x0000000100000bfa): /tmp/main.c:13:23
     Symbol: id = {0x00000004}, range = [0x0000000100000bc0-0x0000000100000dc9), name="main"
   Variable: id = {0x000000bf}, name = "path", type= "char [1024]", location = DW_OP_fbreg(-1072), decl = main.c:28
   Variable: id = {0x00000072}, name = "argc", type= "int", location = r13, decl = main.c:8
   Variable: id = {0x00000081}, name = "argv", type= "const char **", location = r12, decl = main.c:8
   Variable: id = {0x00000090}, name = "envp", type= "const char **", location = r15, decl = main.c:8
   Variable: id = {0x0000009f}, name = "aapl", type= "const char **", location = rbx, decl = main.c:8
```

如上详细信息中会包含地址所在函数的详细信息，其中包含变量名称、变量类型、变量所在寄存器/栈区的偏移量、在文件中的偏移地址等。

## 使用Python API进行符号化

以上所有命令可以通过Python脚本实现：

```
triple = "x86_64-apple-macosx"
platform_name = None
add_dependents = False
target = lldb.debugger.CreateTarget("/tmp/a.out", triple, platform_name, add_dependents, lldb.SBError())
if target:
      # Get the executable module
      module = target.GetModuleAtIndex(0)
      target.SetSectionLoadAddress(module.FindSection("__TEXT"), 0x100000000)
      module = target.AddModule ("/usr/lib/system/libsystem_c.dylib", triple, None, "/build/server/a/libsystem_c.dylib.dSYM")
      target.SetSectionLoadAddress(module.FindSection("__TEXT"), 0x7fff83f32000)
      module = target.AddModule ("/usr/lib/system/libsystem_dnssd.dylib", triple, None, "/build/server/b/libsystem_dnssd.dylib.dSYM")
      target.SetSectionLoadAddress(module.FindSection("__TEXT"), 0x7fff883db000)
      module = target.AddModule ("/usr/lib/system/libsystem_kernel.dylib", triple, None, "/build/server/c/libsystem_kernel.dylib.dSYM")
      target.SetSectionLoadAddress(module.FindSection("__TEXT"), 0x7fff8c0dc000)

      load_addr = 0x00007fff8a1e6d46
      # so_addr is a section offset address, or a lldb.SBAddress object
      so_addr = target.ResolveLoadAddress (load_addr)
      # Get a symbol context for the section offset address which includes
      # a module, compile unit, function, block, line entry, and symbol
      sym_ctx = so_addr.GetSymbolContext (lldb.eSymbolContextEverything)
      print sym_ctx
```

## 使用Python符号化模块

LLDB中有一个lldb.utils.symbolication模块，这个模块包含了许多符号化函数，以创建对象的形式可化符号化过程，其中有这些相关类：

```
lldb.utils.symbolication.Address
lldb.utils.symbolication.Section
lldb.utils.symbolication.Image
lldb.utils.symbolication.Symbolicator
```

**lldb.macosx.crashlog**

该模块解析Crash logs文件信息，并创建代表images、section、thread frames符号化对象，然后可以使用lldb.utils.symbolication解析Crash logs。

```
(lldb) command script import lldb.macosx.crashlog
"crashlog" and "save_crashlog" command installed, use the "--help" option for detailed help
(lldb) crashlog /tmp/crash.log
...
```

```
(lldb) crashlog --help
Usage: crashlog [options]  [FILE ...]
```

```
Options:
-h, --help            show this help message and exit
-v, --verbose         display verbose debug info
-g, --debug           display verbose debug logging
-a, --load-all        load all executable images, not just the images found
                        in the crashed stack frames
--images              show image list
--debug-delay=NSEC    pause for NSEC seconds for debugger
-c, --crashed-only    only symbolicate the crashed thread
-d DISASSEMBLE_DEPTH, --disasm-depth=DISASSEMBLE_DEPTH
                        set the depth in stack frames that should be
                        disassembled (default is 1)
-D, --disasm-all      enabled disassembly of frames on all threads (not just
                        the crashed thread)
-B DISASSEMBLE_BEFORE, --disasm-before=DISASSEMBLE_BEFORE
                        the number of instructions to disassemble before the
                        frame PC
-A DISASSEMBLE_AFTER, --disasm-after=DISASSEMBLE_AFTER
                        the number of instructions to disassemble after the
                        frame PC
-C NLINES, --source-context=NLINES
                        show NLINES source lines of source context (default =
                        4)
--source-frames=NFRAMES
                        show source for NFRAMES (default = 4)
--source-all          show source for all threads, not just the crashed
                        thread
-i, --interactive     parse all crash logs and enter interactive mode
```

## 相关文章

[LLDB官方文档原文](https://lldb.llvm.org/use/symbolication.html)

[iOS Crash文件获取及符号化](https://gsl201600.github.io/2020/04/29/iOSCrash%E6%96%87%E4%BB%B6%E8%8E%B7%E5%8F%96%E5%8F%8A%E7%AC%A6%E5%8F%B7%E5%8C%96/)

[理解iOS Crash Log](https://blog.csdn.net/Hello_Hwc/article/details/80946318)
