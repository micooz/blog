---
title: C++11新特性中的匿名函数Lambda表达式的汇编实现分析（三）
date: 2014-06-10 00:00:00
tags:
  - tech
  - cpp
  - asm
---

1. {% post_link cpp-lambda-expression-asm-1 %}
2. {% post_link cpp-lambda-expression-asm-2 %}

Lambda表达式中较复杂的形式如下：

```
[ capture ] ( params ) -> ret { body }
```

现在我们构造一个简单的Lambda闭包函数进行分析：

```cpp
int main()
{
	int c = 10;
	auto lambda = [&] (int a, int b)->int{
		return a + b - c;
	};
	int r = lambda(1, 2);

	return 0;
}
```

上面的代码中，lambda表达式要求传递两个参数a和b，并按引用捕获c，计算后的结果返回给r。

相应的汇编码如下：

```cpp
int c = 10;
 mov         dword ptr [ebp-8],0Ah  
	auto lambda = [&] (int a, int b)->int{
		return a + b - c;
	};
 lea         eax,[ebp-8]  
 push        eax  
 lea         ecx,[ebp-14h]  
 call        010E13B0  
	int r = lambda(1, 2);
 push        2  
 push        1  
 lea         ecx,[ebp-14h]  
 call        010E1400  
 mov         dword ptr [ebp-20h],eax  

	return 0;
 xor         eax,eax
```

显而易见的，和前面两篇文中的一样，这里仅作简要说明：

由于Lambda表达式中捕获了c，因此这里第一个lea指令，向复制函数传递了c的地址，第二条lea指令向复制函数传递了this用于记录捕获对象的地址，

发生调用时，两个push按照stdcall的方式，从右向左压栈。并向表达式传入了this用于寻址。

lambda调用完毕的返回值默认放在eax中，因此，这里最后一个mov意思是将闭包的函数返回值写入r中。

那么，再看看闭包内如何处理传入参数的以及如何返回的？其实就像普通函数一样的原理，以前的博文也说到过函数调用的汇编原理，这里再简单说说吧。

```cpp
 pop         ecx  
 mov         dword ptr [ebp-8],ecx  
		return a + b - c;
 mov         eax,dword ptr [ebp+8]  
 add         eax,dword ptr [ebp+0Ch]  
 
 mov         ecx,dword ptr [ebp-8]  
 mov         edx,dword ptr [ecx]  
 sub         eax,dword ptr [edx]
```

参数[ebp+8] = a ；[ebp+0Ch] = b

eax往往是放临时量，edx往往是放地址，按照这个经验，很容易看出，后面两个mov取出*this（得到的是&c）然后从a+b中减去*(&c)，结果放在eax中，ret返回后供主函数中的r获取之。

**三篇博文的总结：**

C++11中lambda表达式在形式上改变了函数的书写，使函数调用更加简洁灵活，闭包函数也是许多高级语言的特性之一。

Lambda表达式并不是一个神奇的东西，万变不离其宗，他仍然是以一个函数的形式存在于汇编中，底层处理和普通函数基本一样。

Lambda表达式和普通函数在源程序的实现上有不同：

    Lambda表达式通常被作为参数传递给另一个函数，它本身作为callback，以此避免在其他地方写出完整函数或使用函数指针。

Lambda表达式和普通函数在汇编层上的实现基本相同：

    最特殊的地方是，当闭包中要使用本身作用域外的变量时，需要进行“捕获”，而捕获其实是通过另一个隐藏的(源代码不可见)，我叫它复制函数（或者叫准备函数吧）来实现的，具体实现根据捕获方式的不同而不同，大体上是一系列赋值语句。

    闭包中通过传入的this指针（不能直接使用）对捕获的变量或者对象进行操作。

关于捕获方式中的按值或者按引用的概念，和普通函数一致。

----

好了，说到这里，Lambda表达式的底层实现基本说到，本系列博文均属原创，感谢开源中国OSChina提供这样一个学习交流的平台，读者如有其它见解，欢迎评论！
