---
title: 【汇编】C++ 函数调用之——局部变量
date: 2013-07-26 00:00:01
tags:
  - tech
  - cpp
  - asm
---

我们知道，栈作为一种数据结构，是一种只能在一端进行插入和删除操作的特殊线性表。它按照后进先出的原则存储数据，先进入的数据被压入栈底，最后的数据在栈顶，需要读数据的时候从栈顶开始弹出数据（最后一个数据被第一个读出来）。

C++函数内的参数和局部变量都是存放在栈空间内，那么具体是如何存放和利用的呢？让我们来一探究竟。

有如下测试代码：

```cpp
void func(){
	int a;
	char b;
	double c;

	a=97;
	b=a;
	c=7.2;
}

int main(int argc,char *argv[])
{
	//call func
	func();
	return 0;
}
```

func函数内有三个类型不相同的变量a,b,c 然后对a,b,c分别赋值。
对应的汇编码大致如下：

```cpp
void func(){
00B013D0  push        ebp  
00B013D1  mov         ebp,esp  
00B013D3  sub         esp,0E8h  
00B013D9  push        ebx  
00B013DA  push        esi  
00B013DB  push        edi  
00B013DC  lea         edi,[ebp+FFFFFF18h]  
00B013E2  mov         ecx,3Ah  
00B013E7  mov         eax,0CCCCCCCCh  
00B013EC  rep stos    dword ptr es:[edi]  
//上面的指令意思同样是分配栈空间，只不过大小变成了3A*4=E8(232)字节
//如果没有这三个局部变量和参数，那么分配的空间应该是192字节，232-192=40，
//也就是说多分配了40字节
	int a;
	char b;
	double c;
//a,b,c在栈中的位置如下图所示，从汇编也可以看到：
//&a = ebp-8h
//&b = ebp-11h
//&c = ebp-24h
//好像并不是想像中的那样相邻存放的，但是依然符合栈的存放特点：从高地址往低地址
	a=97;
00B013EE  mov         dword ptr [ebp-8],61h  
	b=a;
00B013F5  mov         al,byte ptr [ebp-8]  
00B013F8  mov         byte ptr [ebp-11h],al  
	c=7.2;
//这里存7.2的时候是从前面某个内存区域通过双字传送指令实现的，
//C语言中7.2默认是double(64位8字节)是一个32位寄存器存放不了的，
//因此需要在编译期先放到内存里面，然后拷贝到栈内
00B013FB  movsd       xmm0,mmword ptr ds:[00B058A8h]  
00B01403  movsd       mmword ptr [ebp-24h],xmm0  
}
00B01408  pop         edi  
00B01409  pop         esi  
00B0140A  pop         ebx  
00B0140B  mov         esp,ebp  
00B0140D  pop         ebp  
00B0140E  ret  
--- 无源文件 -----------------------------------------------------------------------
00B0140F  int         3  
00B01410  int         3  
00B01411  int         3  
00B01412  int         3  
00B01413  int         3  
00B01414  int         3  
00B01415  int         3  
00B01416  int         3  
00B01417  int         3  
00B01418  int         3  
00B01419  int         3  
00B0141A  int         3  
00B0141B  int         3  
00B0141C  int         3  
00B0141D  int         3  
00B0141E  int         3  
00B0141F  int         3  
00B01420  int         3  
00B01421  int         3  
00B01422  int         3  
00B01423  int         3  
00B01424  int         3  
00B01425  int         3  
00B01426  int         3  
00B01427  int         3  
00B01428  int         3  
00B01429  int         3  
00B0142A  int         3  
00B0142B  int         3  
00B0142C  int         3  
00B0142D  int         3  
00B0142E  int         3  
00B0142F  int         3  
--- f:\cpp\clr\consoleapplication1\源.cpp ---------------------------------------


int main(int argc,char *argv[])
{
00B01430  push        ebp  
00B01431  mov         ebp,esp  
00B01433  sub         esp,0C0h  
00B01439  push        ebx  
00B0143A  push        esi  
00B0143B  push        edi  
00B0143C  lea         edi,[ebp+FFFFFF40h]  
00B01442  mov         ecx,30h  
00B01447  mov         eax,0CCCCCCCCh  
00B0144C  rep stos    dword ptr es:[edi]  
	//call func
	func();
00B0144E  call        00B01082  
	return 0;
00B01453  xor         eax,eax  
}
00B01455  pop         edi  
00B01456  pop         esi  
00B01457  pop         ebx  
00B01458  add         esp,0C0h  
00B0145E  cmp         ebp,esp  
00B01460  call        00B01140  
00B01465  mov         esp,ebp  
00B01467  pop         ebp  
00B01468  ret
```

func函数栈区内存分布：

其中红框部分是变量a被赋值之前，

![](102143_ydP4_580940.png)

现在a被赋值为十进制的97，换成16进制就是 0x0000 0061，和下图一致，注意存放规则！

高八位存0000，低八位存0061

![](102259_e2wS_580940.png)

现在a,b,c都被赋值：

可以看到占用情况：

a是int 占4字节；b是char占1字节；c是double占8字节；且a,b,c并未连续存放

![](102130_9DLL_580940.png)
