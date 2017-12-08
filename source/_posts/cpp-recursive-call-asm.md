---
title: 【汇编】C++递归调用实现
date: 2013-07-30 00:00:00
tags:
  - tech
  - cpp
  - asm
---

> 程序调用自身的编程技巧称为递归（ recursion）。一般来说，递归需要有边界条件、递归前进段和递归返回段。当边界条件不满足时，递归前进；当边界条件满足时，递归返回。

有一递归算法计算较小整数的阶乘:

```cpp
int factorial(int x)
{
	int t;
	if(x==0)	
		t=1;	
	else
		t=x*factorial(x-1);
	
	return t;
}

int main(int argc,char *argv[])
{
	int x;

	x=factorial(5);

	return 0;
}
```

对应有汇编码：

调用时：

```cpp
int x;

	x=factorial(5);
00071CAE  push        5  
00071CB0  call        000711D6  
00071CB5  add         esp,4  
00071CB8  mov         dword ptr [ebp-8],eax  
//和普通函数一样，将参数5压栈，调用完成后恢复堆栈，然后把返回值eax放进变量x
递归函数：
int t;
	if(x==0)	
000717CE  cmp         dword ptr [ebp+8],0  
000717D2  jne         000717DD  //x不为0则跳到else的下一条
		t=1;	
000717D4  mov         dword ptr [ebp-8],1  
	else
000717DB  jmp         000717F3  
		t=x*factorial(x-1);
000717DD  mov         eax,dword ptr [ebp+8]  
000717E0  sub         eax,1  
000717E3  push        eax  
//上面三条指令是计算表达式x-1，其值作为调用本身的参数压入栈内
000717E4  call        000711D6  
//调用函数
000717E9  add         esp,4  
//恢复堆栈
000717EC  imul        eax,dword ptr [ebp+8]  
//累乘
000717F0  mov         dword ptr [ebp-8],eax  
//给t赋值
	
	return t;
000717F3  mov         eax,dword ptr [ebp-8]  //设置返回值
```

关键在调用自身时的call 000711D6 这里先call跳转表，然后call该函数。由于是调用本身，再遇到ret指令之前，函数不会返回，因此会不断有压栈操作，每当调用一次函数就会分配一次栈内存，可以预见内存会消耗很快。下面是递归3次的栈内存表现：

![](175233_0C73_580940.png)

除了压栈参数3,4,5（**不在同一个栈空间内**）之外，还有一些函数创建的数据，可以看到大多"空白"(0xcc)的内存浪费掉了，随着递归次数的增加，内存消耗也越多。

因此当你不到万不得已之前，尽量避免使用递归。
