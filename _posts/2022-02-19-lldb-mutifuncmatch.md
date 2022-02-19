---
title: LLDB断点脚本——调用栈中多函数名匹配
tags:
- LLDB
- LLDB脚本
---

## 诉求的产生

项目中有某一个函数funZ，调用到函数funZ的路径可能有多个，例如：

1. funA -> funB -> funZ
2. funC -> funB -> funZ
3. funD -> funE -> funZ
4. funA -> funX -> ... -> funZ

此时我们在函数funZ打下断点的时候，就可能是由以上路径到达该函数（即funZ函数断点触发时候，调用栈可能存在的函数名称列表）。如果我们只想在特定的一条路线到达funZ的时候才触发断点，此时LLDB基本的断点命令已经解决不了了，那我们该怎么达到这种效果呢？可以通过自定义断点命令脚本来完成目标。

## 脚本的功能Case目标

当匹配成功时，funZ函数设置的断点才会生效。

1. 有序且紧邻的函数调用栈匹配，即函数栈中的函数名称要依次相邻，举例如下：（函数调用栈举例）
   - funA -> funB -> funZ，指定的匹配目标
   - funB -> funA -> funZ,  不匹配（A与B顺序反了）
   - funA -> funB -> funX -> funZ，不匹配（中间插入了funX）
2. 有序但不紧邻的函数调用栈匹配，调用栈中间允许有别的的函数被调用，举例如下：
   - funA -> ... -> funB ->... -> funZ，指定的匹配目标
   - funB -> funA -> funZ,  不匹配（A与B顺序反了）
   - funA -> funB -> funX -> funZ，匹配（中间插入了funX也可以匹配）
3. 无顺序要求的函数调用栈匹配，只要求最后一个函数是funZ，其他指定函数不要求顺序，但调用栈中包含就匹配成功，举例如下：
   - (funA，funB) -> funZ，指定的匹配目标（只要调用栈中有funA、funB，就认为匹配成功）
   - funB -> funA -> funZ，匹配
   - funA -> funB -> funX -> funZ，匹配成功

以上3个目标中，从1-3条件逐渐变松。

## 脚本的实现逻辑

脚本逻辑流程图如下：
<img src ="/assets/article/LLDB-biofset1.png" width="800" height="760.0" />
1. 途中的1、2、3对应case目标中的1、2、3。
2. 使用target.BreakpointCreateByRegex生成断点breakpoint。
3. 通过breakpoint.SetScriptCallbackFunction为断点设置回调函数，会根据回调的函数返回值决定该断点是否暂停，true为暂停。
4. 断点回调函数中写下目标匹配规则。
5. 匹配规则中，通过遍历thread.frames 中每一帧的符号名称，判断函数栈中是否存在指定的多个函数名称。 


## 脚本的使用

使用命令如下：

1. 匹配目标1

   ```
   biofset -d d regex0 [Optional_ModuleName] ||| regex1  ModuleName1 ||| regex2 ModuleName2
   命令解释：
   命令  option 函数名正则 [可选的模块名称] ||| 函数名正则 模块名称 ||| 函数名正则 模块名称
   可以使用'|||' 匹配多个函数名，对应到函数调用栈匹配多个函数名
   ```

2. 匹配目标2

   ```
   biofset -d s regex0 [Optional_ModuleName] ||| regex1  ModuleName1 ||| regex2 ModuleName2
   此处只是替换了 -d 后边 为‘s’，s（sort）代表顺序但不一定挨着的匹配
   ```

3. 匹配目标3

   ```
   biofset -d m regex0 [Optional_ModuleName] ||| regex1  ModuleName1 ||| regex2 ModuleName2
   此处只是替换了 -d 后边 为‘m’，m（messy)代表乱序匹配
   ```
	 
## 脚本地址

[脚本](https://github.com/YuBo-Zhou/LLDBScript/blob/main/breakifonfuncset.py)存在github仓库里。
