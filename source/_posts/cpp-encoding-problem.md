---
title: C++ 字符编码问题探究和中文乱码的产生
date: 2015-01-02 00:00:00
tags:
  - tech
  - cpp
---

# 引言

一直以来，C/C++对中文字符的处理时常让人摸不着头脑。

主要有下面几个原因：

1. 文件编码方式的差异

2. 系统环境对中文的解释有差异

3. 不同编译器对标准库的实现有差异

而这三者往往又相互影响，暗藏玄机，让人抓狂。

在写本文之前我查阅了很多博客，关于中文的输入输出，cout，wcout，fstream，wfstream，乱码解决方案等等问题都有了十分详细的解答，但是，很多博文具有片面性。

许多博主仅仅是针对自己所使用的环境做阐述，而又没有明确指明使用了何种IDE，何种编译器，何种系统。结果就是，博主们高高兴兴的解决的自己的问题并分享出来，大家高高兴兴的点赞，觉得自己和博主的问题是同一个问题，实际情况却大相径庭。

# 必要的说明

文本涉及的编译器和系统：

* msvc v120  Windows 8.1

* mingw 4.8 32bit Windows 8.1

* g++ 4.8.2 Linux 64bit

# 开始测试

测试之前很有必要说明一点：

> A program should not mix output operations on wcout with output operations on cout (or with other narrow-oriented output operations on stdout): Once an output operation has been performed on either, the standard output stream acquires an orientation (either narrow or wide) that can only be safely changed by calling freopen on stdout. 
---- cplusplus.com

就是说不要混用cout和wcout进行输出，因此下面的例子中都是单独使用cout或者wcout。

## cout测试

*下面的测试在Visual Studio 2013中进行。*

### MSVC，默认编码GB2312（可以在 **文件--高级保存选项** 中查看和修改）
    
    <!-- lang: cpp -->
    #include <iostream>
    
    int main() {
        using namespace std;
    
        const char *code = "abc中文def";
    
        cout << "abc中文def" << endl;
        cout << code << endl;
    
        return 0;
    }

**结果**

> abc中文def
> 
> abc中文def

均正确输出。

### MSVC，改变编码为UTF8（+bom）

**结果**

> abc中文def
> 
> abc中文def

均正确输出。

### MSVC，改变编码为UTF8（-bom）

**结果**

> abc涓枃def
> 
> abc涓枃def

出现乱码。

### 问题分析

可以看到源文件的编码方式会影响最后的输出，原因在于常量文本采用了**硬编码**的方式，也就是说源代码里面的中文会根据当前文件的编码方式直接翻译成对应字节码放进存储空间。

如“中文”二字，

**GB2312**（Codepage 936）的编码为：

D6 D0 CE C4

而**UTF8**是：

E4 B8 AD E6 96 87

而控制台也有一套编码方式，对于Windows的cmd，可以查看其 **属性** 下面的当前代码页，笔者是ANSI(936)。

当向控制台传送GB2312的字节码时，中文显示正常，当传入无签名的UTF8的字节码时，中文就不能被正确解释，出现了乱码。

Q：为什么带有签名的UTF8却可以正常显示呢？

A：实际上UTF8完全不需要带签名，M$自作聪明YY了一个bom头来识别文件是不是UTF8。因此带有签名的UTF8能被cmd识别为UTF8，中文才能显示正常。

为了进一步证实是不是和控制台的编码有关系，并正确理解上一个例子中乱码的产生缘由，我们可以做一个重定向，将结果输出到文本文件：

> test.exe > test.txt

使用任意可以改变编码的文本编辑器（笔者使用的是everedit）查看，可以发现以UTF8解释，显示正常，以ANSI(936)解释，将得到刚才那个乱码。

----------

*下面的测试在QtCreator中进行。*

### MinGW，UTF8

**结果**

> abc涓枃def
> 
> abc涓枃def

出现乱码。

### MinGW，ANSI-936

**结果**

> abc中文def
> 
> abc中文def

显示正确。

----------

*下面的测试在Linux的bash中进行。*

### g++，UTF8

**结果**

> abc中文def
> 
> abc中文def

显示正确。

### g++，gb2312

**结果**

> abc▒▒▒▒def
> 
> abc▒▒▒▒def

出现乱码。

Ubuntu查看/etc/default/locale，可以看到LANG="en_US.UTF-8"，说明bash能解释UTF8的字节码，而gb2312的变成了乱码。

### 小结

程序的输出编码必须和"显示程序"的显示编码适配时才能得到正确的结果。简而言之就是**解铃还须系铃人**。

----------

宽字符使用多个字节来表示一个字符，中文可以用char来表示没问题，用wchar来表示也没有问题。

## wcout测试

wcout输出wchar_t型的宽字符，测试代码如下：

    <!-- lang: cpp -->
    #include <iostream>
    
    int main() {
        using namespace std;
    
        const wchar_t *code = L"abc中文def";
    
        wcout << L"abc中文def" << endl;
        wcout << code << endl;
    
        return 0;
    }

### MSVC，无论上述何种编码

**结果**

abc

输出被截断，只有前几个英文字母可以被输出，传入指针输出无效。

### 问题分析

**L"abc中文def"** 在内存中表现为：

> (gb2312)    61 00 62 00 63 00 **2d 4e 87 65** 64 00 65 00 66 00

> (utf8-bom) 61 00 62 00 63 00 **2d 4e 87 65** 64 00 65 00 66 00

> (utf8+bom)61 00 62 00 63 00 **93 6d 5f e1 83 67** 64 00 65 00  66 00

wcout 在处理L"abc中文def"时，按宽字节依次遍历，前面的abc没问题（小端序第一个字节是00），遇到中文，识别不了，无输出，间接导致后续<<都没有输出了。

也就是说wcout不能用来处理中文输出。

第二个传入wchar_t指针，发现没有任何输出，为了验证是不是由于上一条输出语句中中文的影响，单独测试如下：

    <!-- lang: cpp -->
    #include <iostream>
    
    int main() {
        using namespace std;
    
        const wchar_t *code = L"abc中文def";
    
        wcout << code << endl;
    
        return 0;
    }

**结果**

abc

说明传入wchar_t指针是可以正常输出宽字节英文的，一旦遇到非00字节间隔，后续所有输出将无效。

MinGW的结果同样如此，无论编码与否，只要wcout遇到中文立马跪。

有博主称可以在输出前执行下面的函数或者进行全局设置来让wcout认识中文：

    <!-- lang: cpp -->
    std::wcout.imbue(std::locale("chs"));
    std::locale::global(std::locale(""));//全局设置

* MSVC下，没有问题，可以达到预期结果。

* MinGW下，第一条语句会抛出一个runtime_error异常崩溃掉，第二条语句无效。

* Linux g++下，没问题。

可见MinGW的libstdc++对locale的实现不理想，有传闻使用stlport可以避免这个问题。

----------

# 总结

* 认清你的代码处在何种编码的环境

* 认清放在你字符串里面的数据是何种编码

* 认清你要向具有何种编码的屏幕传送数据

* 解铃还须系铃人

* 非特殊情况下不建议使用wchar_t来存放中文字符

很多时候中文并不是硬编码进程序的，例如一段中文来自网络，以gb2312编码，而"屏幕"只认UTF8，这个时候就要进行必要的编码转换。boost库的boost::locale::conv中提供了很多常用的转换模板函数，来实现宽窄字符、ansi和utf之间的相互转换。
