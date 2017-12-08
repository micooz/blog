---
title: 【汇编】C++ 函数调用分析之——无参调用。
date: 2013-07-25 00:00:00
tags:
  - tech
  - cpp
  - asm
---

由于C++函数调用多种多样，本文将分几个章节分别讲解。

首先来最简单的一个无参数调用。

有如下测试代码：

```cpp
void func(){
	//do nothing here
}

int main(int argc,char *argv[])
{
	//call func
	func();
	return 0;
}
```

在主函数中调用自定义func函数。

再看看两个函数的汇编码：

```cpp
void func(){
//显然和之前讲过的main的实现雷同，挖一个192字节栈空间，然后填int 3中断(清cc)
002F13A0  push        ebp  
002F13A1  mov         ebp,esp  
002F13A3  sub         esp,0C0h  
002F13A9  push        ebx  
002F13AA  push        esi  
002F13AB  push        edi  
002F13AC  lea         edi,[ebp+FFFFFF40h]  
002F13B2  mov         ecx,30h  
002F13B7  mov         eax,0CCCCCCCCh  
002F13BC  rep stos    dword ptr es:[edi]  
	//do nothing here
}
002F13BE  pop         edi  
002F13BF  pop         esi  
002F13C0  pop         ebx  
002F13C1  mov         esp,ebp  
002F13C3  pop         ebp  
002F13C4  ret  
--- 无源文件 -----------------------------------------------------------------------
002F13C5  int         3  
002F13C6  int         3  
002F13C7  int         3  
002F13C8  int         3  
002F13C9  int         3  
002F13CA  int         3  
002F13CB  int         3  
002F13CC  int         3  
002F13CD  int         3  
002F13CE  int         3  
002F13CF  int         3  
--- f:\cpp\clr\consoleapplication1\源.cpp ---------------------------------------


int main(int argc,char *argv[])
{
002F13D0  push        ebp  
002F13D1  mov         ebp,esp  
002F13D3  sub         esp,0C0h  
002F13D9  push        ebx  
002F13DA  push        esi  
002F13DB  push        edi  
002F13DC  lea         edi,[ebp+FFFFFF40h]  
002F13E2  mov         ecx,30h  
002F13E7  mov         eax,0CCCCCCCCh  
002F13EC  rep stos    dword ptr es:[edi]  
	//call func
	func();
//这里就编译出一条指令 call 002F11D1 这个地址是什么呢，在上面比较远的地方，为了简化，我只取一部分汇编码：
##################################
_GetLastError@0:
002F11BD  jmp         002F3972  
_RTC_GetErrorFunc:
002F11C2  jmp         002F1980  
_wcscpy_s:
002F11C7  jmp         002F391E  
@_RTC_AllocaHelper@12:
002F11CC  jmp         002F1410  
func:
002F11D1  jmp         002F13A0  ###002F11D1在这里
@_RTC_CheckStackVars@8:
002F11D6  jmp         002F1820  
@_RTC_CheckStackVars2@12:
002F11DB  jmp         002F39D0  
__RTC_CheckEsp:
##################################
//显然上面call的是一个跳转表里面的func的调转地址，而不是直接call 002F13A0(真正地址)

002F13EE  call        002F11D1  
	return 0;
002F13F3  xor         eax,eax  
}
002F13F5  pop         edi  
002F13F6  pop         esi  
002F13F7  pop         ebx  
002F13F8  add         esp,0C0h  
002F13FE  cmp         ebp,esp  
002F1400  call        002F11E0  
002F1405  mov         esp,ebp  
002F1407  pop         ebp  
002F1408  ret
```

C++编译时，把函数地址放到一个跳转表里面，调用函数时，去找跳转表对应函数地址，然后再jmp一次。

可以看到，调用函数需要跳两次，同时，会产生许多指令进行栈操作，不管怎样，调用一个函数必须付出192字节(本例中)代价，不管你有没有使用局部变量。所以尽量避免调用过多无意义的函数。
