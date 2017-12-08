---
title: 【汇编】C++ 中的数组初始化的实现
date: 2013-07-27 00:00:00
tags:
  - tech
  - cpp
  - asm
---

我们已经知道，数组存放一组数据类型相同的变量，各个变量连续存放，地址由低到高。

如整形数组：

```cpp
int s[5]={1,2,3,4,5};
```

有5个元素。

对应有汇编码：

```cpp
int s[5]={1,2,3,4,5};
00B917BE  mov         dword ptr [ebp-18h],1  
00B917C5  mov         dword ptr [ebp-14h],2  
00B917CC  mov         dword ptr [ebp-10h],3  
00B917D3  mov         dword ptr [ebp-0Ch],4  
00B917DA  mov         dword ptr [ebp-8],5
```

可以看到5个元素有5条mov指令对每个元素进行初始化，地址从低到高，其中ebp是main的栈底，查看内存如下：

![](123107_G17V_580940.png)

可以看到确实是连续存放的。

如果这样初始化呢？

```cpp
int s[5]={0};
00EA17BE  mov         dword ptr [ebp-18h],0  
00EA17C5  xor         eax,eax  
00EA17C7  mov         dword ptr [ebp-14h],eax  
00EA17CA  mov         dword ptr [ebp-10h],eax  
00EA17CD  mov         dword ptr [ebp-0Ch],eax  
00EA17D0  mov         dword ptr [ebp-8],eax
比较有意思的是先把第一个元素清0，然后再用eax去初始化其他元素，这样做可能是为了效率吧。
有另一个问题，就是我们常常要用到一个比较大的缓冲区，比如：

char buffer[1024]; 这样的话难道会像上面一样有 1024个mov吗？答案肯定否定的。

int s[100]={0};
000717BE  mov         dword ptr [ebp+FFFFFE6Ch],0  
//首先将s的第一个元素清0
000717C8  push        18Ch  
//再将18c压栈，这个 18c = (100-1)*sizeof(int)，就是数组的字节长度少4
//因为第一个元素已经初始化过了
000717CD  push        0  
//将0压栈
000717CF  lea         eax,[ebp+FFFFFE70h]  
000717D5  push        eax  
//把地址ebp+FFFFFE70h压栈
//上面的三个push应该是为这个call做准备，可以知道这个call有三个参数，那这个call究竟是是什么呢？
000717D6  call        000711E0  
//找到跳转表看看：
################
_memset:
000711E0  jmp         000713FA  
000711E5  int         3  
000711E6  int         3  
实际上那个call调用了memset函数，但是我们并没有引入任何头文件，我猜想是编译器自动引入了string头文件。
还记得上面的push吗？
push 396
push 0
push &s[0]
换成高级语言就是 memset(&s[0],0,396)，很明显是一个初始化一段内存的操作
################
000717DB  add         esp,0Ch  //三个参数，三个DWORD=Ch
```

看来编译器很是聪明，知道这个时候调用函数来完成初始化操作。当然如果不在声明时初始化数组，那么我们自己也可以直接：

memset(s,0,sizeof(s))
