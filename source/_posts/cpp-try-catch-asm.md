---
title: 【C++&汇编】try-catch异常处理机制的汇编实现
date: 2013-12-21 00:00:00
tags:
  - tech
  - cpp
  - asm
---

C++中支持使用try-catch的语法处理异常，防止程序崩溃。那么编译器是如何实现的呢？

有如下测试代码：

```cpp
int main(int argc,char** argv){

	try{
		throw "error";
	}
	catch (char* err){
		err = nullptr;
	}

	return 0;
}
```

这里直接在try中抛出异常，程序捕获这个异常将错误信息传给err，为了使catch块不被编译器忽略，这里随便附一个值，顺便看看这个nullptr是什么。

编译出的汇编代码如下：

```cpp
int main(int argc,char** argv){
//ebp用于定位函数局部变量，先保存ebp，然后移动到栈顶
00A91410  push        ebp  
00A91411  mov         ebp,esp  
//数0xFFFFFFFF和地址0A94EF0压栈，这个其实是一个函数的地址(后面有说明)
00A91413  push        0FFFFFFFFh  
00A91415  push        0A94EF0h  
//这里出现了一个fs寄存器，取fs的0偏移，即得到“指向SEH链指针”(后面说明)
//SEH链指针压栈，ecx压栈
00A9141A  mov         eax,dword ptr fs:[00000000h]  
00A91420  push        eax  
00A91421  push        ecx  
//分配栈空间
00A91422  sub         esp,0D8h  
//保护现场
00A91428  push        ebx  
00A91429  push        esi  
00A9142A  push        edi  
00A9142B  lea         edi,[ebp-0E8h]  
//清理栈
00A91431  mov         ecx,36h  
00A91436  mov         eax,0CCCCCCCCh  
00A9143B  rep stos    dword ptr es:[edi]  
//这里把ds:0xA99000(0x6DAAB763,应该是某dword型的标志)的数据和ebp(栈顶)做异或，将结果保存
00A9143D  mov         eax,dword ptr ds:[00A99000h]  
00A91442  xor         eax,ebp  
00A91444  push        eax  
//这里ebp-0c显然是一个SEH链指针，因为它被保存到fs:0
00A91445  lea         eax,[ebp-0Ch]  
00A91448  mov         dword ptr fs:[00000000h],eax  
//保存esp是为了什么？
00A9144E  mov         dword ptr [ebp-10h],esp  

	try{
//ebp-4这个变量的值很关键，进入try就被清0
00A91451  mov         dword ptr [ebp-4],0  
		throw "error";
//直接一个地址0A96858是静态存储区"error"字符串的首地址，保存到栈空间，临时变量无疑
00A91458  mov         dword ptr [ebp-0E4h],0A96858h  
//参数 0A98078 -- 第二个参数比较奇怪
00A91462  push        0A98078h  
//参数 &"error"
00A91467  lea         eax,[ebp-0E4h]  
00A9146D  push        eax  
//调用函数 相当于 __CxxThrowException("error",0x0A98078);
//该函数从名字上可以略知一二，由于多次调用比较复杂，所以就不分析里面了
00A9146E  call        __CxxThrowException@8 (0A910FAh)  
	}
	catch (char* err){
		err = nullptr;
//可以看到nullptr就是0
00A91473  mov         dword ptr [ebp-18h],0  
	}
//标签$LN7位置的指令地址做返回值，返回处理后可能会再次跳到$LN7，而将ebp-4这个变量取反
//可能意味着出现错误
00A9147A  mov         eax,0A91489h  
00A9147F  ret  
00A91480  mov         dword ptr [ebp-4],0FFFFFFFFh  
//去return
00A91487  jmp         $LN7+7h (0A91490h)  
$LN7:
00A91489  mov         dword ptr [ebp-4],0FFFFFFFFh  

	return 0;
00A91490  xor         eax,eax
```

这里省略最后一个括弧的编译结果，把重点放在try-catch块。

下面对上面的一些地方进行详细说明：

**1. 地址0A94EF0**

```cpp
__ehhandler$_main:
00A94EF0  mov         edx,dword ptr [esp+8]  
00A94EF4  lea         eax,[edx+0Ch]  
00A94EF7  mov         ecx,dword ptr [edx-0ECh]  
00A94EFD  xor         ecx,eax  
00A94EFF  call        @__security_check_cookie@4 (0A9101Eh)  
00A94F04  mov         eax,0A9804Ch  
00A94F09  jmp         ___CxxFrameHandler3 (0A91190h)  
00A94F0E  int         3  
00A94F0F  int         3  
00A94F10  int         3
```

**2. fs寄存器和SEH链**

关于fs寄存器的作用引用一个博客里面的：

```
FS寄存器指向当前活动线程的TEB结构（线程结构）

偏移   说明 

000   指向SEH链指针 

004   线程堆栈顶部 

008   线程堆栈底部 

00C   SubSystemTib 

010   FiberData 

014   ArbitraryUserPointer 

018   FS段寄存器在内存中的镜像地址 

020   进程PID 

024   线程ID 

02C   指向线程局部存储指针 

030   PEB结构地址（进程结构） 

034   上个错误号
```

http://blog.sina.com.cn/s/blog_a5ece79401016os9.html

SEH链：

> (structured exception handling) 是一种处理程序异常的机制，当程序异常 (例如除零异常，非法存取异常，等等) 发生的时候，系统便会把执行位置切换到thread 的 exception handler。

更多关于SEH机制可以参考：

http://blog.csdn.net/pofante/article/details/7044028
