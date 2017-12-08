---
title: 【汇编】C++ 数组寻址实现
date: 2013-07-29 00:00:00
tags:
  - tech
  - cpp
  - asm
---

上一节探究C++数组的初始化问题，编译器做了很多事。这一节再来看看数组的寻址问题，数组元素的读取，写入，元素之间的内容传送。

有测试代码：

```cpp
int main(int argc,char *argv[])
{
	int s[5]={0};

	s[3]=3;

	int x=s[2];

	s[0]=s[1];

	return 0;
}
```

第一行是数组的初始化，

第二行是写入值，

第三行读取值，

第四行是数组元素之间的值传递。

下面看看汇编代码：

```cpp
int s[5]={0};
//数组元素初始化在上一节介绍过，这里不再说明
008633D8  mov         dword ptr [ebp-1Ch],0  
008633DF  xor         eax,eax  
008633E1  mov         dword ptr [ebp-18h],eax  
008633E4  mov         dword ptr [ebp-14h],eax  
008633E7  mov         dword ptr [ebp-10h],eax  
008633EA  mov         dword ptr [ebp-0Ch],eax  

	s[3]=3;
008633ED  mov         eax,4  
008633F2  imul        eax,eax,3  
008633F5  mov         dword ptr [ebp+eax-1Ch],3  
/*
对数组元素的改写相对容易，注意指令imul为有符号乘法，就是eax=eax*3
内存单元[ebp+eax-1Ch]显然表示的是第四个元素s[3]，这里如何算的？
ebp是当前main函数的栈底基址，使用ebp加减一个数来访问栈变量；
ebp-1Ch是数组s的首地址，
eax赋值为sizeof(int),
因此s[3]就应该是 [ &s+3*sizeof(int) ] 就是从首地址开始偏移3个元素所占内存大小(字节)
*/
	int x=s[2];
008633FD  mov         eax,4  
00863402  shl         eax,1  
00863404  mov         ecx,dword ptr [ebp+eax-1Ch]  
00863408  mov         dword ptr [ebp-28h],ecx  
/*
这里指令shl是左移操作，将eax左移1相当于 eax*=2
这里的寻址方式和上面有所不同，通过ecx转存给x
s[2]=[&s+2*sizeof(int)]
这里也可以向上面一样写成：
### 008633ED  mov         eax,4  
### 008633F2  imul        eax,eax,2
编译器使用shl位操作应该是提升效率，因为位操作的速度非常快
*/
	s[0]=s[1];
0086340B  mov         eax,4  
00863410  shl         eax,0  
00863413  mov         ecx,4  
00863418  imul        ecx,ecx,0  
0086341B  mov         edx,dword ptr [ebp+eax-1Ch]  
0086341F  mov         dword ptr [ebp+ecx-1Ch],edx
/*
经过上面的变化，eax=4，ecx=0，其中正如上面一样，ecx是用来寻s[0]，eax是寻s[1]
使用edx作为桥梁，实现内存单元之间的赋值
*/
```

可见C++对于数组元素的引用最重要的一点是数组基址(首地址)，和偏移量，首地址在编译期就可以确定不变，偏移量通过数组元素所占内存大小sizeof(type)和数组索引 [index] 算出，从而找到里面的元素。
