---
title: 【汇编】C++ 函数调用之——有参无返回调用（传值）
date: 2013-07-26 00:00:02
tags:
  - tech
  - cpp
  - asm
---

C++函数有参调用有几种传参方式：

1. 传值
2. 传指针（地址）
3. 传引用

其中参数可被const修饰，也可以有默认值。下面分情况讨论：

为了简洁，省略main函数的汇编码而直接给出func函数的汇编码。

有源代码：

```cpp
void func(int a,char b){
	int c;
	c=a+b;
}

int main(int argc,char *argv[])
{
	//call func
	func(10,'a');
	return 0;
}
```

下面看看汇编码：

调用发生时：

```cpp
//call func
	func(10,'a');
//进行参数压栈操作，首先是'a'压入栈，然后是10压栈，然后call跳转表，再由调转表call函数
00F1141E  push        61h  
00F11420  push        0Ah  
00F11422  call        00F1113B  
//函数调用完成后，栈减小8字节，两个dword，因为CPU对栈的操作都是双字操作，这里两个参数就是两个双字
00F11427  add         esp,8
```

具体内存中的表现是这样的（先让func把栈初始化）：

![](124421_kiVQ_580940.png)

显然不在func的stack内，注意两个参数前面还有两个DWORD，

一个是00f1 1427,另一个是00dd f794；这两个DWORD的产生应该是在PUSH两个参数之后，

又有的两个PUSH，显然，第一个PUSH 00f1 1427是在call 时将ip压栈导致的：

这个ip是当前这条call 指令的下一条指令(add)的地址，请参考上面的main函数。

第二个PUSH是在 func函数中完成的，可以参考func函数的汇编码：

```cpp
void func(int a,char b){

00F113D0  push        ebp  
//这里第二个PUSH，压入ebp，显然这个ebp的值可以在main函数里面看到，
//有两条：
//## 00F11401  mov   ebp,esp 
//## 00F11403  sub   esp,0C0h 
//那么ebp就是main的栈底

00F113D1  mov         ebp,esp  
00F113D3  sub         esp,0CCh  
00F113D9  push        ebx  
00F113DA  push        esi  
00F113DB  push        edi  
00F113DC  lea         edi,[ebp+FFFFFF34h]  
00F113E2  mov         ecx,33h  
00F113E7  mov         eax,0CCCCCCCCh  
00F113EC  rep stos    dword ptr es:[edi]  
	int c;
	c=a+b;
00F113EE  movsx       eax,byte ptr [ebp+0Ch]  
00F113F2  add         eax,dword ptr [ebp+8]  
00F113F5  mov         dword ptr [ebp-8],eax  
}
00F113F8  pop         edi  
00F113F9  pop         esi  
00F113FA  pop         ebx  
00F113FB  mov         esp,ebp  
00F113FD  pop         ebp  
00F113FE  ret
```

调用发生时，压入两个参数后，必须再保存下一条指令的位置，因此有一个压栈操作，这个操作是有call指令来完成的。 其次，func函数将ebp压栈是为了为恢复堆栈做准备。因为CPU只有两个寄存器用于堆栈操作：SS:SP，为了调用func函数完成时能进入main的堆栈，必须先保存(push ebp)再恢复(pop ebp),这一点从func函数末尾也看得出。

此外，更直观一点，从内存中看得出：第二个push 00ddf794和func的stack靠的很近:

![](124842_QbXs_580940.png)

恰好是指向了main的栈底。

再来看看func里面:

```cpp
int c;
	c=a+b;
00F113EE  movsx       eax,byte ptr [ebp+0Ch]  
00F113F2  add         eax,dword ptr [ebp+8]  
00F113F5  mov         dword ptr [ebp-8],eax  

//经过分析可以知道：
//&b = ebp+0ch
//&a = ebp+8
//&c = ebp-8
//在上面的分析中我们知道这个ebp是指向栈底的，局部变量c在栈内，参数a 和 b 是之前push进来的
```

经过上述分析，可以得出一些结论：

有参函数调用发生时：

1.先将参数从右向左依次压栈
2.将下一条指令的地址压栈
3.被调函数将主调函数的栈底位置压栈
4.被调函数初始化自己的栈
5.取出参数进行运算(并不是pop)
6.恢复栈指针
7.执行ret恢复(pop)ip，此时程序转到call的下一条add esp
8.向下移动栈顶指针sp，所谓的释放局部变量。

可以看到局部变量的"释放"其实是在主调函数中完成的，而不是在被调用函数末尾。

"释放"不是清除内存，而是修改栈指针使局部变量不能访问。
