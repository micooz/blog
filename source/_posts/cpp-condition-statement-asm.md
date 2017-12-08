---
title: 【汇编】C++条件控制语句分析
date: 2013-07-24 00:00:00
tags:
  - tech
  - cpp
  - asm
---

【测试环境】：Visual Studio Ultimate 2012 (V110) 

C++中条件控制语句主要分3种： 

**一、单条件结构**

if(condition)  
    {statements} 

**二、带else条件结构**

if(condition) 
    {statements}  
else  
    {statements} 

**三、多条件结构**

if(condition1) 
    {statements1} 
else if(condition2) 
    {statements2}  
...  
else 
    {statements last} 

本文将对这三类结构进行汇编分析，看看编译器对其的底层实现。帮助我们更好得了解高级编译性语言的实现过程。 

另外，文末将对简化的条件控制结构——三目运算进行汇编实现分析。 

首先不妨看看对空statement编译器是如何处理的？ 

有如下代码段：

```cpp
int main(int argc,char *argv[])
{
    bool test;
    test=0;
 
    if(test)
        ;
 
    return 0;
}
```

编译上述代码，将得到一条警告消息： 

警告1warning C4390: “;”: 找到空的受控语句；这是否是有意的? 

忽略警告调试程序，将得到下面的汇编码，这里只取{}内的汇编码：

```cpp
 bool test;
    test=0;
//可见内存单元[ebp-5]就是test变量的位置，且test占用一个字节(byte ptr)
//编译器为test随机分配的一块内存，在内存窗口查看该内存单元可以发现该单元周围都被cc (204) 填充，
//因此test的准确值为0xcc也就是十进制的204，换成bool就是true
//这句将test清0
004013BE  mov         byte ptr [ebp-5],0 
//达到if空语句，可见并没有编译出任何汇编码，编译器选择了直接跳过空if语句，因为这条语句本身并没有任何意义。
    if(test)
        ;
//将main函数返回值清0
    return 0;
004013C2  xor         eax,eax
```

下面介绍第一种if结构： 

**一、单条件结构** 

if(condition)  
    {statements} 

有如下测试代码：

```cpp
int main(int argc,char *argv[])
{
    bool test;
    test=0;
 
    if(test)
        test=1;
 
    return 0;
}
```

对应汇编代码如下： 

```cpp
bool test;
    test=0;
000313BE  mov         byte ptr [ebp-5],0 
 
    if(test)
//使用movzx指令将一字节test传送至通用寄存器eax中临时保存，根据上述代码可以预见，
//传送后eax将变为0x0000 0000
000313C2  movzx       eax,byte ptr [ebp-5] 
//将eax与自己本身作And运算  下面两条其实就是 实现了 if(test != 0) ...
000313C6  test        eax,eax 
//条件不成立则跳出这个if，执行return 0;
000313C8  je          000313CE 
        test=1;
//将test置1
000313CA  mov         byte ptr [ebp-5],1 
 
    return 0;
000313CE  xor         eax,eax
```

如果将条件改为if(test==1)，这对应的汇编码变为： 

```cpp
movzx       eax,byte ptr [ebp-5]   
cmp         eax,1   
jne         003A13CF
```

应该容易理解了。

**二、带else条件结构** 

if(condition) 
    {statements}  
else  
    {statements} 


有如下测试代码：

```cpp
int main(int argc,char *argv[])
{
    bool test;
    test=0;
 
    if(test)
        test=0;
    else
        test=1;
 
    return 0;
}
```

对应汇编代码： 

```cpp
bool test;
    test=0;
003513BE  mov         byte ptr [ebp-5],0 
 
    if(test)
003513C2  movzx       eax,byte ptr [ebp-5] 
003513C6  test        eax,eax 
//主要是看这里，条件不成立不是跳出if，而是跳到test=1这上面了。
003513C8  je          003513D0 
        test=0;
003513CA  mov         byte ptr [ebp-5],0 
    else
//这里有点不好理解，else编译出来是跳出if
//如果条件成立，执行test=0;然后跳出if 是符合逻辑的，
//编译器在else上下手，创造跳出if的条件，这一点很值得学习
003513CE  jmp         003513D4 
        test=1;
003513D0  mov         byte ptr [ebp-5],1 
 
    return 0;
003513D4  xor         eax,eax
```

**三、多条件结构** 

if(condition1) 
    {statements1} 
else if(condition2) 
    {statements2}  
...  
else 
    {statements last} 

有如下测试代码：

```cpp
 int main(int argc,char *argv[])
{
    int test=10;
 
    if(test==20)
        test=20;
    else if(test==30)
        test=30;
    else
        test=10;
 
    return 0;
}
```

更换一下test的类型为int，然后进行多条件判断。 

对应的汇编代码如下： 

```cpp
int test=10;
//可见VC下int是双字型(DWORD)，占用4个字节
001517CE  mov         dword ptr [ebp-8],0Ah 
 
    if(test==20)
//将test与20进行比较
001517D5  cmp         dword ptr [ebp-8],14h 
//不等则跳到下面一个else if
001517D9  jne         001517E4 
        test=20;
001517DB  mov         dword ptr [ebp-8],14h 
        test=20;
//跳出整个 if
001517E2  jmp         001517FA 
    else if(test==30)
//将test与30进行比较
001517E4  cmp         dword ptr [ebp-8],1Eh 
//不等则跳到下一个else if 或else
001517E8  jne         001517F3 
        test=30;
001517EA  mov         dword ptr [ebp-8],1Eh 
    else
//跳出整个 if  关键还是最后这个else 上，总会有一个结束整个if的jmp指令
001517F1  jmp         001517FA 
        test=10;
001517F3  mov         dword ptr [ebp-8],0Ah 
 
    return 0;
001517FA  xor         eax,eax
```

可见多条件控制语句是依次计算条件成立与否。  

最后，看看三目表达式的实现：

有如下测试是代码：

```cpp
int main(int argc,char *argv[])
{
    int test=10;
     
    (test==20)?1:0;
 
    return 0;
}
```

为了方便不接收三目表达式的返回值。 

对应的汇编代码如下： 

```cpp
 int test=10;
00BE17CE  mov         dword ptr [ebp-8],0Ah 
     
    (test==20)?1:0;
//比较是不是20
00BE17D5  cmp         dword ptr [ebp-8],14h 
//不成立则跳到冒号后的语句
00BE17D9  jne         00BE17E7 
//成立则执行问号后的语句
//这里把表达式返回值放进栈空间[ebp-208]
00BE17DB  mov         dword ptr [ebp+FFFFFF30h],1 
//跳出表达式
00BE17E5  jmp         00BE17F1 
00BE17E7  mov         dword ptr [ebp+FFFFFF30h],0 
 
    return 0;
00BE17F1  xor         eax,eax
```

与第二类if结构比较，三目运算则显得更紧凑，逻辑性更强些。语法规则也很简单。
