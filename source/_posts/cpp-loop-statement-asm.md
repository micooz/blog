---
title: 【汇编】C++ 循环语句分析
date: 2013-07-24 00:00:00
tags:
  - tech
  - cpp
  - asm
---

测试环境Visual Studio Ultimate 2012 (v110) 

C++有三种循环控制语句：

**一.for循环**

for(initialization;condition;increase){
    statements;
}

**二.while循环**

while (expression){
    statements;
}

**三.do-while循环**

do 
{statements;}
 while (condition);

本文将分别利用反汇编分析上述三种循环的底层实现，帮助我们更好的了解循环机制。

**一.for循环**

for(initialization;condition;increase){
    statements;
}

有如下测试代码段:

```cpp
#include <iostream>

int main(int argc,char *argv[])
{
	for (int i=0;i<100;++i)
	{
		std::cout<<i<<"\n";
	}

	return 0;
}
```

该代码将在控制台输出0-99。 

Debug运行查看反汇编码大致如下： 

```cpp
 for (int i=0;i<100;++i)
//容易看到内存单元[ebp-8]就是循环变量i的位置，首先将它初始化清0
00B54F2E  mov         dword ptr [ebp-8],0  
//这里跳转至下面cmp指令
00B54F35  jmp         main+30h (0B54F40h)  
//将i放入eax暂存，然后自加1，然后再放回i里面，就是实现i++
//内存单元不能直接进行这样的操作：add dword ptr [ebp-8],1
00B54F37  mov         eax,dword ptr [ebp-8]  
00B54F3A  add         eax,1  
00B54F3D  mov         dword ptr [ebp-8],eax  
//验证i是不是==64h(100)了
00B54F40  cmp         dword ptr [ebp-8],64h  
//条件一旦成立，也就是i==100的时候，结束循环
00B54F44  jge         main+5Fh (0B54F6Fh)  
	{
		std::cout<<i<<"\n";
//打印的实现不是本文的主题，所以略去
00B54F46  push        0B5CC74h  
00B54F4B  mov         esi,esp  
00B54F4D  mov         eax,dword ptr [ebp-8]  
00B54F50  push        eax  
00B54F51  mov         ecx,dword ptr ds:[0B60310h]  
00B54F57  call        dword ptr ds:[0B60318h]  
00B54F5D  cmp         esi,esp  
00B54F5F  call        __RTC_CheckEsp (0B51325h)  
00B54F64  push        eax  
00B54F65  call        std::operator<<<std::char_traits<char> > (0B512A3h)  
00B54F6A  add         esp,8  
	}
//这里的}是for的结束位置，编译为一个jmp无条件跳转指令，跳到i++那里，
//这样就实现了循环
00B54F6D  jmp         main+27h (0B54F37h)  

	return 0;
00B54F6F  xor         eax,eax
```

经过上面的分析，可以知道，C++在实现for循环的时候，先后依次进行下面的操作：

1.执行initialization，初始化循环变量
2.执行condition，检查条件表达式成立与否
3.成立则进行循环，执行statements
4.循环返回
5.执行increase
6.转到第2步，直到condition不再成立

设想：如果将i++生成的三条指令放到 }  jmp main+27h (0B54F37h) 之前，是不是也可以顺利执行呢？

**二.while循环**

while (expression){
    statements;
}

有如下测试代码:

```cpp
#include <iostream>

int main(int argc,char *argv[])
{
	int n=100;
	while(n--)
		std::cout<<n<<"\n";

	return 0;
}
```

这段代码将打印99-0 

对应的汇编码如下： 

```cpp
int n=100;
//显而易见[ebp-8]是n的位置
00EC4F2E  mov         dword ptr [ebp-8],64h  
	while(n--)
//将n暂时放入eax
00EC4F35  mov         eax,dword ptr [ebp-8]  
//然后再将eax放入栈内[ebp-208]
00EC4F38  mov         dword ptr [ebp+FFFFFF30h],eax  
//然后把n放进ecx，自减1之后再放回来，类似上面的i++，这里实现的是n--
00EC4F3E  mov         ecx,dword ptr [ebp-8]  
00EC4F41  sub         ecx,1  
00EC4F44  mov         dword ptr [ebp-8],ecx  
//判断刚才放进n的临时变量是不是0,是的话结束整个while
00EC4F47  cmp         dword ptr [ebp+FFFFFF30h],0  
00EC4F4E  je          00EC4F79  
//打印略去
		std::cout<<n<<"\n";
00EC4F50  push        0ECCC74h  
00EC4F55  mov         esi,esp  
00EC4F57  mov         eax,dword ptr [ebp-8]  
00EC4F5A  push        eax  
00EC4F5B  mov         ecx,dword ptr ds:[00ED0310h]  
00EC4F61  call        dword ptr ds:[00ED0318h]  
00EC4F67  cmp         esi,esp  
00EC4F69  call        00EC1325  
00EC4F6E  push        eax  
00EC4F6F  call        00EC12A3  
00EC4F74  add         esp,8  
00EC4F77  jmp         00EC4F35  

	return 0;
00EC4F79  xor         eax,eax
```

相比for循环，while循环看起来很紧凑，本身也比for简单，由于涉及到一个条件表达式n--，这里判断的时候，要将表达式的返回值放进一个临时变量里面，这里变量是编译器规定的：

```cpp
dword ptr [ebp+FFFFFF30h]
```

同时这里也很明显的看到表达式n--其实返回的是n，因为cmp的是sub ecx,1之前的值，但是此时n已经被减去1了。

如果改为: --n呢？

暂时不纠结这个前减后减的问题，我们来看看do-while循环怎么样？

**三.do-while循环**

do 
{statements;}
 while (condition);
 
同样，有如下测试代码：

```cpp
#include <iostream>

int main(int argc,char *argv[])
{
	int n=100;
	do 
	{
		std::cout<<n<<"\n";
	} while (--n);

	return 0;
}
```

这段代码将打印100-1

为了和解决刚才遗留的一个问题，我特意写成--n。 

对应的汇编代码如下： 

```cpp
int n=100;
00FD4F2E  mov         dword ptr [ebp-8],64h  
	do 
	{
//打印实现略去
		std::cout<<n<<"\n";
00FD4F35  push        0FDCC74h  
00FD4F3A  mov         esi,esp  
00FD4F3C  mov         eax,dword ptr [ebp-8]  
00FD4F3F  push        eax  
00FD4F40  mov         ecx,dword ptr ds:[00FE0310h]  
00FD4F46  call        dword ptr ds:[00FE0318h]  
00FD4F4C  cmp         esi,esp  
00FD4F4E  call        00FD1325  
00FD4F53  push        eax  
00FD4F54  call        00FD12A3  
00FD4F59  add         esp,8  
	} while (--n);
//这里只有4条指令，前三条十分熟悉，实现n自减1
00FD4F5C  mov         eax,dword ptr [ebp-8]  
00FD4F5F  sub         eax,1  
00FD4F62  mov         dword ptr [ebp-8],eax  
//如果标志位ZF=0则跳转，这个ZF是由sub的运算结果影响的，
//当sub的运算结果变成0，则退出循环了。
//也就是说这里其实是在看 n减少1之后的结果是不是0，而不是减少前
00FD4F65  jne         00FD4F35  

	return 0;
00FD4F67  xor         eax,eax
```

现在将--n改成n-- 看看： 

结果是打印100-0，多打印了一个0 

再看看汇编如何实现？ 

```cpp
} while (n--);
00CC4F5C  mov         eax,dword ptr [ebp-8]  
00CC4F5F  mov         dword ptr [ebp+FFFFFF30h],eax  
00CC4F65  mov         ecx,dword ptr [ebp-8]  
00CC4F68  sub         ecx,1  
00CC4F6B  mov         dword ptr [ebp-8],ecx  
00CC4F6E  cmp         dword ptr [ebp+FFFFFF30h],0  
00CC4F75  jne         00CC4F35
```

同样产生一个临时变量，这和刚才while循环的情形十分相似。 

这里jne检查由cmp指令影响的标志位，是检查的临时变量而不是已经被减去1的n.

经过上面的验证，可以看到：

C++是通过变量的增加或减少，判断条件或者标志位，运用跳转指令来实现循环控制结构的。相比之下，while 和 do while比for更简洁，实际上书写高级语言也是如此。

此外，这里讨论了前减和后减的区别:

后减会产生一个临时变量，并对这个临时变量进行条件判断。前减不会产生临时变量，是对变量本身进行条件判断。

好了，本文到此结束，如有任何疑惑或者说的不恰当的地方还请大家批评指正。
