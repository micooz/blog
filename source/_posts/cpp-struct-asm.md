---
title: 【汇编】C++ 结构体实现
date: 2013-07-29 00:00:00
tags:
  - tech
  - cpp
  - asm
---

> C语言允许用户自己指定这样一种数据结构，它由不同类型的数据组合成一个整体，以便引用，这些组合在一个整体中的数据是互相联系的，这样的数据结构称为结构体。

C++结构体中可以使用函数，有下面的测试代码：

```cpp
int main(int argc,char *argv[])
{
	struct MyStruct
	{
		char *name;
		int income;
		int expenses;
		int profit(){
			return income-expenses;
		}
	};
	MyStruct person;
	person.name="Mike";
	person.income=3600;
	person.expenses=1200;

	int x=person.profit();

	return 0;
}
```

有一个名为MyStruct的结构体，有三个数据成员，有一个函数成员，下面声明一个这样的结构体变量，对其进行赋值，然后调用函数取得返回值。

对应汇编代码如下：

```cpp
struct MyStruct
	{
		char *name;
		int income;
		int expenses;
		int profit(){
			return income-expenses;
		}
	};
//可以看到上面的声明中并没有编译出任何汇编码，包括函数，只是告诉编译器有这么一个数据结构
	MyStruct person;
	person.name="Mike";
00E43A58  mov         dword ptr [ebp-14h],0E458A8h  
//将字符串"Mike"的首地址放进第一个成员char*name指针,这个地址在内存中的表现见下图
//通过查看&person和&name可以发现是同一个位置，
	person.income=3600;
00E43A5F  mov         dword ptr [ebp-10h],0E10h  
//这里是第二个成员的赋值
	person.expenses=1200;
00E43A66  mov         dword ptr [ebp-0Ch],4B0h  
//第三个赋值
	int x=person.profit();
//第四个调用函数
00E43A6D  lea         ecx,[ebp-14h]  
00E43A70  call        00E433B0  
00E43A75  mov         dword ptr [ebp-20h],eax  
//可以看到先是把这个结构的首地址放进ecx，由于无参数所以没有压栈操作
//函数计算结果同样是放在了eax中
```

字符串"Mike"(0E458A8h): 

![](112511_l7iJ_580940.png)

注意这个Mike周围还有许多英文单词，而且每个字母之间都是00隔开的，这就是所谓的宽字符WCHAR，在Windows编程中常常看到。

再来看看结构体里面的函数编译到哪里去了？

```cpp
struct MyStruct
	{
		char *name;
		int income;
		int expenses;
		int profit(){
//作为函数同样是要分配栈空间
00E433B0  push        ebp  
00E433B1  mov         ebp,esp  
00E433B3  sub         esp,0CCh  
00E433B9  push        ebx  
00E433BA  push        esi  
00E433BB  push        edi  
00E433BC  push        ecx  //由于下面必须要用ecx，这里先压入栈内保存
00E433BD  lea         edi,[ebp+FFFFFF34h]  
00E433C3  mov         ecx,33h  
00E433C8  mov         eax,0CCCCCCCCh  
00E433CD  rep stos    dword ptr es:[edi]  
00E433CF  pop         ecx  //这里恢复出来
00E433D0  mov         dword ptr [ebp-8],ecx  
//把传进来的该结构体地址(相当于this指针)放进栈
			return income-expenses;
00E433D3  mov         eax,dword ptr [ebp-8]  
00E433D6  mov         ecx,dword ptr [ebp-8]  
//把this放进eax和ecx，因为下面要用它来寻址
00E433D9  mov         eax,dword ptr [eax+4]  
00E433DC  sub         eax,dword ptr [ecx+8]  
//[this+4]就是变量income，[this+8]就是变量expenses，这里用[eax+8]应该也可以
//作差后返回
		}
00E433DF  pop         edi  
00E433E0  pop         esi  
00E433E1  pop         ebx  
00E433E2  mov         esp,ebp  
00E433E4  pop         ebp  
00E433E5  ret
```

小结：结构体最关键的还是首地址，或者叫做结构体的this指针，用它可以访问结构体中的每个成员，是通过[this+偏移]来实现的，这个偏移不同所访问到的成员就不同，偏移具体是多少是在编译期分析成员类型决定的。
