---
title: 【汇编】C++ Win32ConsoleApp main函数的构造分析
date: 2013-07-25 00:00:00
tags:
  - tech
  - cpp
  - asm
---

C++控制台标准main函数的固定格式如下：

```cpp
int main(int argc,char *argv[])
{
	return 0;
}
```

有两个参数：

1.argc 整形 记录参数个数(包括路径)
2.argv 字符双指针或者叫字符串数组 记录各个参数(包括路径全名)

假设使用下面的命令启动上述代码产生的可执行文件A.exe：

C:\>A let's run cpp main

则argc将是5，argv[5]分别是"A","let's","run","cpp","main"的首地址

下面看看汇编结果：

```cpp
int main(int argc,char *argv[])
{
//栈指针上移，为main函数的局部变量预留0xC0的空间 -191
00B413A0  push        ebp  
00B413A1  mov         ebp,esp  
00B413A3  sub         esp,0C0h  
//注意到main末尾有对应的三个出栈操作，这里入栈是为了保护现场
00B413A9  push        ebx  
00B413AA  push        esi  
00B413AB  push        edi  
//之前ebp是被赋值为了esp，代表栈头，用edi记录栈尾位置 +192
00B413AC  lea         edi,[ebp+FFFFFF40h]  
//重复30h(48)次，每次一个dword(4字节)，48*4恰好是192该函数栈的大小
//eax放的是0x0CCCCCCCCh，即int3中断 ，下面三条指令将int 3 中断填满栈
//前后内存状态参考图
00B413B2  mov         ecx,30h  
00B413B7  mov         eax,0CCCCCCCCh  
00B413BC  rep stos    dword ptr es:[edi]  

	
	return 0;
//将返回值eax清空
00B413BE  xor         eax,eax  
}
//上面push了，这里就pop，恢复现场
00B413C0  pop         edi  
00B413C1  pop         esi  
00B413C2  pop         ebx  
00B413C3  mov         esp,ebp  
00B413C5  pop         ebp  
//恢复cs:ip返回系统调用者
00B413C6  ret
```

edi是栈尾位置，下图是执行rep stos 之前：

![](215246_QbWl_580940.png)

下图是执行之后：(红色的应该就是这192字节的栈了)

![](215454_dxuD_580940.png)

这里可以看到系统调用main函数，函数先申请栈空间，只有192字节，然后将这192字节全部初始化为cc，所以但我们 int a; 之后，这个a就是0xcccc cccc，char b; 这个b就是0xcc(换成ascii码就是一个英文问号?) 所以新手没有对变量初始化打印出来就是所谓的乱码~~~

这里由于没有使用参数的原因，所以看不到对参数的操作。
