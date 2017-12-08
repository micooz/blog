---
title: 【汇编】C++ 函数调用之——有参无返回调用（传引用）
date: 2013-07-27 00:00:00
tags:
  - tech
  - cpp
  - asm
---

1. {% post_link cpp-function-call-2 %}
2. {% post_link cpp-function-call-3 %}

传引用调用函数，就是向被调函数传递的是一个引用类型，而引用是作为变量的别名存在，本质上同样是一个地址。

有源代码：

```cpp
void func(int &b){
	b=10;
}

int main(int argc,char *argv[])
{
	//call func
	int a;
	func(a);
	return 0;
}
```

此代码向func函数传递一个局部变量a的引用，从而修改这个a的值。

对应有汇编码：

main中，

```cpp
//call func
	int a;
	func(a);
00931438  lea         eax,[ebp-0Ch]  
0093143B  push        eax  
0093143C  call        009311E5  
00931441  add         esp,4
```

可见和传递指针一样，push进去的是a的地址&a

func中，

```cpp
b=10;
009313EE  mov         eax,dword ptr [ebp+8]  
009313F1  mov         dword ptr [eax],0Ah
```

同样，和指针实现并没有任何区别。

小结：传递引用其实是通过指针实现的，传递的是地址，而不是对象的拷贝。
