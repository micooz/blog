---
title: 【汇编】C++ 函数调用之——有参有返回调用
date: 2013-07-27 00:00:01
tags:
  - tech
  - cpp
  - asm
---

大多数函数都通过return语句返回一个值，那么被调函数是如何返回的，主调函数又是如何接受的呢？下面让我们来一探究竟。

有测试代码：

```cpp
int func(int a){
	return a+1;
}

int main(int argc,char *argv[])
{
	//call func
	int x;
	x=func(10);
	return 0;
}
```

这段代码很简单，调用函数func向其传递参数10，然后函数返回10+1=11，并保存到变量x中。

对应有汇编代码：

```cpp
//call func
	int x;
	x=func(10);
001A142E  push        0Ah  
001A1430  call        001A11F4  
001A1435  add         esp,4  
001A1438  mov         dword ptr [ebp-8],eax
```

正如前几篇所讲的函数调用，这里的汇编流程几乎和前面一样，只是这里要接受一个返回值。

重点在最后一个mov指令，他将寄存器eax的内容传送进内存单元[ebp-8]，这个内存单元在main的栈内，其实就是变量x，这个mov显然是为x赋值，赋的是eax。

可以猜测，**函数func的返回值是放在了寄存器eax中**。

下面为了验证上面的猜想，我们看看func的汇编实现：

```cpp
return a+1;
001A13EE  mov         eax,dword ptr [ebp+8]  
001A13F1  add         eax,1
```

内存单元[ebp+8]在函数func参数表中，即是参数a，mov指令将参数a传送到eax中
add指令将eax的内容加1，似乎并没有所谓的return返回，其实真正意义上的返回是通过ret指令实现的，ret指令将进行一个pop ip；执行之后就返回到了调用func的下一条指令的地方。

现在应该容易理解了，自定义函数的返回值一般是放在eax中的，其实早在{% post_link cpp-main-asm %}的时候就提到过：

```cpp
    return 0;
//将返回值eax清空
00B413BE  xor         eax,eax 
```

函数通过return语句只能返回一个值，这个值大多数情况下放在eax中。所以在调用别的函数时，只要调用完成之后及时把eax的内容取出来就行了。
