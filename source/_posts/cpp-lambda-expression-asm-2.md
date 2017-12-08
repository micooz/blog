---
title: C++11新特性中的匿名函数Lambda表达式的汇编实现分析（二）
date: 2014-06-10 00:00:00
tags:
  - tech
  - cpp
  - asm
---

{% post_link cpp-lambda-expression-asm-1 %}

首先，让我们来看看以&方式进行变量捕获，同样没有参数和返回。

```cpp
int main()
{
	int a = 0xB;
	auto lambda = [&]{
		a = 0xA;
	};
	lambda();
	return 0;
}
```

闭包中将main中a变量改写为0xA。

main中的关键汇编代码：

```cpp
int a = 0xB;
 mov         dword ptr [ebp-8],0Bh  
	auto lambda = [&]{
		a = 0xA;
	};
 lea         eax,[ebp-8]  
 push        eax  
 lea         ecx,[ebp-14h]  
 call        002D1BE0  
	lambda();
 lea         ecx,[ebp-14h]  
 call        002D1C20  
	return 0;
```

同样的，进入闭包前要调用一个拷贝函数。

002D1BE0 内：

```cpp
 pop         ecx  
 mov         dword ptr [ebp-8],ecx  
 mov         eax,dword ptr [ebp-8]  
 mov         ecx,dword ptr [ebp+8]  
 mov         dword ptr [eax],ecx  
 mov         eax,dword ptr [ebp-8]  
 pop         edi  
 pop         esi  
 pop         ebx  
 mov         esp,ebp  
 pop         ebp  
 ret         4
```

和前面一篇文章中的代码基本一致，但是有两个地方不同，

上文写到：

```cpp
 pop         ecx  
 mov         dword ptr [ebp-8],ecx  
 mov         eax,dword ptr [ebp-8]  
 mov         ecx,dword ptr [ebp+8]  
 mov         edx,dword ptr [ecx]  
 mov         dword ptr [eax],edx  
 mov         eax,dword ptr [ebp-8]  
```

注意黑体部分，若采用**[=]**的捕获方式，那么将通过寄存器edx拷贝原变量的值；

若采用**[&]**方式，则直接通过ecx拷贝原变量的**地址**，而不取出值。

闭包内：
```cpp
 pop         ecx  
 mov         dword ptr [ebp-8],ecx  
		a = 0xA;
 mov         eax,dword ptr [ebp-8]  
 mov         ecx,dword ptr [eax]  
 mov         dword ptr [ecx],0Ah  
	};
 pop         edi  
 pop         esi  
 pop         ebx  
 mov         esp,ebp  
 pop         ebp  
 ret
```

对a进行赋值，直接是：

*this = 0xA;

因为事先this就取到了a的地址。

可以发现，按引用捕获实际上就如同函数参数传递引用，传递的是一个地址值，而不创建对象副本（可以和上一篇文中的内容比较）。

C++11标准中对Lambda表达式的捕获内容还有一些特定支持，比如可以以指定的方式捕获指定的变量：

```cpp
int main()
{
	int a = 0xB;
	bool b = true;
	auto lambda = [&a,b]{
		a = b;
	};
	lambda();
	return 0;
}
```

上面的代码对a进行引用捕获，对b按值捕获。根据前面分析的结果，可以预见，a的地址和b的值将被拷贝以供闭包函数使用。

```cpp
int a = 0xB;
 mov         dword ptr [ebp-8],0Bh  
	bool b = true;
 mov         byte ptr [ebp-11h],1  
	auto lambda = [&a,b]{
		a = b;
	};
 lea         eax,[ebp-11h]  
 push        eax  
 lea         ecx,[ebp-8]  
 push        ecx  
 lea         ecx,[ebp-24h]  
 call        00222060  
	lambda();
 lea         ecx,[ebp-24h]  
	lambda();
 call        00221C20  
	return 0;
```

调用Lambda之前，先调用复制函数，传入两个参数，&a和&b，而this被放在main的[ebp-24h]中。

复制函数或者叫准备函数：

```cpp
 pop         ecx  
 mov         dword ptr [ebp-8],ecx  
 mov         eax,dword ptr [ebp-8]  
 mov         ecx,dword ptr [ebp+8]  
 mov         dword ptr [eax],ecx  
 mov         eax,dword ptr [ebp-8]  
 mov         ecx,dword ptr [ebp+0Ch]  
 mov         dl,byte ptr [ecx]  
 mov         byte ptr [eax+4],dl  
 mov         eax,dword ptr [ebp-8]  
 pop         edi  
 pop         esi  
 pop         ebx  
 mov         esp,ebp  
 pop         ebp  
 ret         8
```

[eax] 就是 *this，也即a

[eax+4] 就是 *(this+4)，也即b

从内存图可以清楚的看到，a的地址被记录，b的值被复制

![](125531_7Agm_580940.png)

闭包函数内：

```cpp
 pop         ecx  
 mov         dword ptr [ebp-8],ecx  
		a = b;
 mov         eax,dword ptr [ebp-8]  
 movzx       ecx,byte ptr [eax+4]  
 
 mov         edx,dword ptr [ebp-8]  
 mov         eax,dword ptr [edx]  
 mov         dword ptr [eax],ecx
```

b的值是从[eax+4]也即this+4中取出，而a在this中，其实就是：

*(this) = *(this+4);

可以看到闭包内通过this作为基址，对闭包外的变量进行偏移访问。

下一篇中，我将对具有参数和返回值的Lambda表达式进行分析。
