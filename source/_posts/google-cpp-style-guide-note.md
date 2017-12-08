---
title: Google C++ Style Guide 笔记
date: 2015-02-23 00:00:00
tags:
  - tech
  - cpp
---

英文原版: http://google-styleguide.googlecode.com/svn/trunk/cppguide.html

# 头文件

每个源文件都要对应一个头文件。例外：单元测试文件和仅包含main的小型源文件。

## 独立头文件

以.h结尾的都是应该是独立的，以.inc结尾的仅用作文本包含，所有头文件都必须是独立的。

inline和template函数的声明和定义(实现)应该在同一个文件中。

Note:这里的独立是指，用户和重构工具可以无特殊限制地包含头文件。

## define宏保护

所有头文件都应该使用#define来防止重复包含。格式如下：

头文件路径：foo/src/bar/baz.h

    <!-- lang: cpp -->
    #ifndef FOO_BAR_BAZ_H_
    #define FOO_BAR_BAZ_H_
    
    ...
    
    #endif  // FOO_BAR_BAZ_H_

## 前置声明

前置声明用来避免不必要的include，但有很多缺陷。

* 当使用在一个头文件中已声明的函数时，总是include那个头文件
* 当使用类模板，最好include它的头文件
* 当使用一个一般的class时，可以进行前置声明，但是要小心它可能是不完整或不正确的，不确定时，就include合适的头文件
* 不要为了避免使用include而把数据成员换成指针

## 内联函数

仅在函数少于10行时，才可以把函数定义为inline。

过度使用inline，实际上会使程序变慢。inline一个非常短的函数可以缩短代码长度，而inline一个很长的函数反而会戏剧性地增大代码长度。

小心析构函数，他们的代码通常比表面上的更长，因为存在虚函数和父析构函数调用。

虚函数和递归函数不应该被inline。

## 参数顺序

顺序应该是：输入，输出

输入参数通常是按值，或按const引用传入。输出则是non-const的指针（引用也是指针实现的）。

当要新增一个参数时，应该按照上面顺序加入。

注意这个规则不是硬性的，可以放宽这个规则以确保一致性。

## includes的名称和顺序

顺序：
1. 关联的头文件  ---->  确保编译当前源文件时提前报错，而不牵扯到别人的文件。
2. C库
3. C++库
4. 第三方库
5. 你的项目头文件

每个部分再按字典顺序排列。例如：

    <!-- lang: cpp -->
    #include "foo/server/fooserver.h"
    
    #include <sys/types.h>
    #include <unistd.h>
    #include <hash_map>
    #include <vector>
    
    #include "base/basictypes.h"
    #include "base/commandlineflags.h"
    #include "foo/server/bar.h"

系统相关的头文件可用宏来限定，以减小代码和保持本地化。

    <!-- lang: cpp -->
    #include "foo/public/fooserver.h"
    
    #include "base/port.h"  // For LANG_CXX11.
    
    #ifdef LANG_CXX11
    #include <initializer_list>
    #endif  // LANG_CXX11

# 作用域

## 命名空间

在源文件中鼓励使用匿名命名空间（ namespace {...} ），对于非匿名命名空间则选取它的路径来命名。不要使用using namespace。不要使用inline命名空间。

## 匿名命名空间

1. 可以避免链接期的名字冲突：

    <!-- lang: cpp -->
    namespace {                           // This is in a .cc file.
    
    // The content of a namespace is not indented.
    //
    // This function is guaranteed not to generate a colliding symbol
    // with other symbols at link time, and is only visible to
    // callers in this .cc file.
    bool UpdateInternals(Frobber* f, int newval) {
      ...
    }
    
    }  // namespace

2. 不要在头文件中使用匿名命名空间。

## 含名称的命名空间

* 在includes，gflags声明和定义，其他命名空间中class的前置声明之后，包裹全部的代码：

    <!-- lang: cpp -->
    // In the .h file
    namespace mynamespace {
    
    // All declarations are within the namespace scope.
    // Notice the lack of indentation.
    class MyClass {
     public:
      ...
      void Foo();
    };
    
    }  // namespace mynamespace
    // In the .cc file
    namespace mynamespace {
    
    // Definition of functions is within scope of the namespace.
    void MyClass::Foo() {
      ...
    }
    
    }  // namespace mynamespace

* 不要在std中声明任何东西，也不要对标准库的class进行前置声明。

* 不要使用using namespace

    <!-- lang: cpp -->
    // Forbidden -- This pollutes the namespace.
    using namespace foo;

* 可以在源文件的任意位置，在头文件的函数，方法，class中使用using声明

    <!-- lang: cpp -->
    // OK in .cc files.
    // Must be in a function, method or class in .h files.
    using ::foo::bar;

* 别名命名空间可以在源文件的任意位置，头文件的namespace内，函数，方法内使用。

    <!-- lang: cpp -->
    // Shorten access to some commonly used names in .cc files.
    namespace fbz = ::foo::bar::baz;
    
    // Shorten access to some commonly used names (in a .h file).
    namespace librarian {
    // The following alias is available to all files including
    // this header (in namespace librarian):
    // alias names should therefore be chosen consistently
    // within a project.
    namespace pd_s = ::pipeline_diagnostics::sidetable;
    
    inline void my_inline_function() {
      // namespace alias local to a function (or method).
      namespace fbz = ::foo::bar::baz;
      ...
    }
    }  // namespace librarian

Note：尽量避免在公共头文件中使用别名命名空间。

* 不要使用inline命名空间

## 成员类

    <!-- lang: cpp -->
    class Foo {
    
     private:
      // Bar is a member class, nested within Foo.
      class Bar {
        ...
      };
    
    };

不要把成员类公开，除非他们确实是这个接口的一部分。

## 非成员，静态成员和全局函数

最好在命名空间中使用非成员函数，或使用静态成员函数，而不要或很少使用全局函数。

非成员函数不应该依赖于外部变量，并且应该总处于一个命名空间中。

那些仅仅用来集结静态成员函数，且没有共享静态数据的类，应该用命名空间取而代之。

如果必须要使用非成员函数并且只在当前这个源文件中使用，则用一个匿名命名空间或static修饰来限制它的作用域。

## 局部变量

将一个函数中的变量放到尽可能有限的作用域内，并在声明时初始化。

    <!-- lang: cpp -->
    int i;
    i = f();      // Bad -- initialization separate from declaration.
    int j = g();  // Good -- declaration has initialization.
    vector<int> v;
    v.push_back(1);  // Prefer initializing using brace initialization.
    v.push_back(2);
    vector<int> v = {1, 2};  // Good -- v starts initialized.

一些供if,while,for使用的变量应该在statement处声明。

    <!-- lang: cpp -->
    while (const char* p = strchr(str, '/')) str = p + 1;

一点注意：如果是变量是一个对象，构造函数将在每次进入作用域时被执行，析构函数将在每次离开作用域时被执行。

    <!-- lang: cpp -->
    // Inefficient implementation:
    for (int i = 0; i < 1000000; ++i) {
      Foo f;  // My ctor and dtor get called 1000000 times each.
      f.DoSomething(i);
    }
    // 对象放在外面更高效
    Foo f;  // My ctor and dtor get called once each.
    for (int i = 0; i < 1000000; ++i) {
      f.DoSomething(i);
    }
    
## 静态和全局变量

类的静态或全局变量是禁止使用的。然而constexpr的变量是被允许的，因为他们不会动态初始化或销毁。

静态存储的对象，包括全局变量，静态变量，静态类成员变量以及函数内静态变量，必须是Plain Old Data(POD)。

只有int,char,float,pointer,或者arrays/structs属于POD。

静态vector应该用C数组代替，静态string应该用const char []代替。

如果需要使用静态或全局类对象，考虑初使用它的指针类型（不会被order-of-destrctor释放掉），注意必须是一个纯的指针，而不是一个智能指针。

# 类

## 构造函数该做的事

避免在构造函数中进行复杂初始化，比如那些可能会失败或者进行虚函数调用的步骤。

构造函数不应该调用虚函数，或者试图抛出非致命性错误。如果你的对象需要进行重要的初始化工作，考虑使用一个工厂函数或init方法。

## 初始化

## 明确构造函数

在具有一个参数的构造函数上使用explicit关键字。因为构造时传入一个参数可能会被编译器当做拷贝构造进行隐式转换。

    <!-- lang: cpp -->
    explicit Foo(string name);

除了构造函数的第一个参数外，其他参数都应该指定一个默认值,来防止不期望的类型转换。

    <!-- lang: cpp -->
    Foo::Foo(string name, int id = 42)

拷贝构造函数，以及作为其他类的透明包装的类，不应该被explicit修饰。

## 可拷贝和可移动类型

可拷贝的例子：std::string

可移动但不可拷贝的例子：std::unique_ptr<int>

对于一些不需要拷贝操作的类型，提供拷贝操作符可能会产生混淆，无意义或者完全是不正确的。

基类的拷贝、赋值操作符是有风险的，他们会导致对象分裂。

如果要加入拷贝特性，就要同时定义拷贝构造函数和赋值操作符。

如果你的类型可拷贝，但移动操作符更高效，那么就同时定义移动操作。

避免给试图被继承的类提供赋值操作符或公开的拷贝/移动构造函数。

如果你的基类需要被拷贝，提供一个公开的虚Clone()方法，和一个保护的拷贝构造函数，来使子类能够实现它。

如果你不想支持拷贝/移动操作，使用 = delete 来明确地禁用它们。

## 委托和继承构造函数

委托构造的例子：

    <!-- lang: cpp -->
    X::X(const string& name) : name_(name) {
      ...
    }
    
    X::X() : X("") { }

继承构造的例子：

    <!-- lang: cpp -->
    class Base {
     public:
      Base();
      Base(int n);
      Base(const string& s);
      ...
    };
    
    class Derived : public Base {
     public:
      using Base::Base;  // Base's constructors are redeclared here.
    };

在能减少冗余和改善可读性的前提下使用委托和继承。

## 结构体 vs. 类

struct仅在存储数据时使用，否则使用class。

struct可以直接访问字段而不通过方法调用。struct内的方法只用来设置数据成员。

如果需要更多地函数支持，class更合适，如果不确定用哪个，就用class。

注意struct和class内的成员变量具有不同的命名方式。

## 继承

组合通常比继承更合适。当使用继承时，指明为public。

实际上，继承在C++中主要应用于两个方面：

* 实现继承（最普通意义上的继承）
* 接口继承（仅继承方法，实现接口）

所有的继承都应当是public的，如果你需要用private继承，你就应该保存一份基类的成员实例来替代private继承（达到private的效果）。

不要过度使用实现继承，因为代码实现被分散于子类和基类。

继承应该被限定为"is-a"的关系，即"子类是基类的特例"。

如果你的类中有虚函数，那么你的析构函数也必须是virtual。

数据成员应该是private的。

## 多继承

仅有很少的多重实现继承是有用的。你通常可以找到一个不同的，更明确地，更干净的解决方案。

多继承仅允许在父类都是纯接口的时候使用。

## 接口

一个类是纯接口的条件：

* 只有public的纯虚函数（= 0）和静态方法（析构函数除外）
* 不含非静态的数据成员
* 不需要构造函数，如果有，一定没有参数以及被声明为protected
* 如果它是子类，它的父类也必须符合这些条件

## 操作符重载

不要重载那些很少使用的，特殊的操作符。不要给用户字面值定义操作符。

为了使类模板函数正常工作，你或许需要定义操作符。

虽然操作符重载可以使代码更简洁，但它有以下几个缺点：

* 它会让我们以为原本高代价的操作是划算的
* 不便于寻找到底调用了那个方法，例如Equals()要比==更方便查找
* 许多操作符对指针也有效，很容易出bug。例如：Foo + 4和 &Foo + 4截然不同
* 用户字面值允许创造出新的句法，即使经验丰富的C++程序员也会觉得陌生

重载也会出现意想不到的后果，比如，一个类重载了一元operator&，它将不能安全的被前置声明。

一般而言，不要定义操作符重载。需要时你可以用普通的函数例如Equals()来替代。

不要重载operator""，即用户字面值。

当然，有些情况下可以进行重载，例如对标准C++库：

    <!-- lang: cpp -->
    operator<<(ostream&, const T&) for logging

## 访问控制

使数据成员是private的，然后提供访问函数（getter/setter）来访问它们。例如：一个叫做foo_的变量有一个foo()访问函数，或者set_foo()。

例外：static const 数据成员(kFoo)不应该是private。

访问器的定义通常是内联在头文件中的。

## 声明顺序

public: protected: private: ；方法在数据成员之前。

每个区块内的顺序：

* typedef & enum
* 常量（包括静态数据成员）
* 构造函数
* 析构函数
* 方法（包括静态方法）
* 数据成员（静态数据除外）

友元函数总是在private内声明。

在源文件中的方法定义应该和声明顺序尽量保持一致。

## 写出短小的函数

书写短小的，清晰地函数。如果一个函数超过40行，可以考虑是否可以在不改变程序结构的前提下进行分割。

# Google经验技巧

## 所有权和智能指针

最好使用std::unique_ptr来使所有权传递更明确：

    <!-- lang: cpp -->
    std::unique_ptr<Foo> FooFactory();
    void FooConsumer(std::unique_ptr<Foo> ptr);

在没有一个非常好的理由的前提下，不要把你的代码设计为共享所有权。

不要再新的代码中使用scoped_ptr除非为了适配老版本的C++。

不要使用std::auto_ptr，用std::unique_ptr代替。

## cpplint

使用cpplint.py来检查风格错误。

# 其他C++特性

## 引用参数

所有按引用传递的参数必须被const修饰。

    <!-- lang: cpp -->
    void Foo(const string &in, string *out);

有些情况下用const T* 做输入参数比const T&好：

* 要传递一个空指针
* 该函数将指针或引用保存到输入参数

记住大多数时候输入参数都写为const T&。

## 右值引用

右值引用只在move构造函数以及move赋值操作符定义时使用，不要使用std::forward。

* 在move构造函数中应用右值引用，可以实现移动数据而不是复制数据，例如：auto v2(std::move(v1))会把v1数据移动到v2中，而不是完全复制一遍，可用于性能提升。
* 无论参数是否为临时对象，右值引用都可以经行参数传递。
* 右值引用对于没有明确定义拷贝操作，但你仍想向其传递参数的函数有很大的意义，比如向容器内存放数据而不需要copy。
* std::move对高效使用一些标准库类型，例如std::unique_ptr很有必要。
* 右值引用作为C++11的新特性，还没有被广泛得理解和应用。

## 函数重载

如果一个函数依靠参数类型的不同来进行重载，读者可能必须理解C++的复杂匹配机制来确定接下来会发生什么。

如果你像重载一个函数，考虑给出一些信息对参数进行限定，例如使用AppendString(), AppendInt()而不仅仅是Append()。

## 默认参数

除了下面几个情形，我们不允许默认函数参数。如果合适的话，请用函数重载在替代。

当默认参数做函数指针时容易使人混淆，因为函数签名常常不会匹配调用。加入默认参数时会改变函数的类型，这样会在获取它的地址时造成一些问题。使用函数重载可以避免这个问题。

一些例外：

当一个静态函数出现在源文件中时，由于本地化的缘故上述规则不再适用。

另外，默认参数可在构造函数中使用，上述规则也不适用，因为不可能获取到构造函数的地址。

还有一个例外是默认参数用来模拟变长数组时，例如：

    <!-- lang: cpp -->
    // Support up to 4 params by using a default empty AlphaNum.
    string StrCat(const AlphaNum &a,
                  const AlphaNum &b = gEmptyAlphaNum,
                  const AlphaNum &c = gEmptyAlphaNum,
                  const AlphaNum &d = gEmptyAlphaNum);

## 变长数组和alloca()

我们不允许使用变长数组或者alloca()

变长数组和alloca()并不是标准C++的一部分。更重要的是，他会在栈空间中分配大量的空间，可能会触发内存覆盖的bug：在我机器上运行的好好的，在生产环境却死掉了。。

## 友元

友元通常都应该被定义在同一个文件中。

通常会定义一个FooBuilder作为Foo的一个友元。

在创建一个单元测试的时候，使用友元很管用。

## 例外

我们不使用C++的exceptions。

## 运行时类型信息（RTTI）

避免使用RTTI。

在运行时查询对象的类型可以说是一个错误的设计问题。

## 类型转换

使用C++的类型转换如static_cast<int>()。而不要用C-style的int y = (int)x或者int y = int(x)。

使用const_cast来除去const限定，只在你知道你在做什么的情况下，使用reinterpret_cast来做不安全的指针转换。

## 流

流仅用于日志。使用类似printf的形式来代替流。

## 前加和前减

在迭代器和其他模板对象上使用前加或前减。

当表达式的返回值被忽略时，++i比i++更高效。如果i是一个迭代器，由于i++的复制，开销很大。

## const

只要讲得通，随时随地使用const。C++11中的constexpr是更好的选择。

* 如果一个函数不会修改按引用传递，指针传递的参数，那么这个参数应该是const。
* 如果可以，把方法声明为const。访问器应该总是const的。其他方法如果不修改任何数据成员，不调用任何非const方法，不返回非const指针或非const引用，也应该修饰为const。
* 数据成员在构造函数执行之后不会被修改，应该声明为const。

const应该放在哪儿？

    <!-- lang: cpp -->
    int const *foo;
    const int* foo;

把const放在第一位具有更好地可读性，因为它符合英语的习惯：形容词（const），然后是名词（int）。

## constexpr

在C++11中，可以使用constexpr来定义真实地常量或者确保常量初始化。

## 整形

<stdint.h>中定义了一些不同长度的整形：int16_t, uint32_t, int64_t 等等。你应该总是使用这些整形，特别是你需要保证整形的长度时。

当我们认为一个整数"比较大"时，用int64_t。

你不应该使用无符号整形，如uint32_t，除非有一个合理的原因。特别的，不要认为使用了无符号类型它就不会是负数，使用断言来检验正负。

如果你需要接收一个容器的大小，确保你的类型能够容纳这个数字，否则使用一个更大的类型。

注意整形转换、类型提升可能会导致非预期行为。

关于无符号整形

    <!-- lang: cpp -->
    for (unsigned int i = foo.Length()-1; i >= 0; --i) ...

这将是一个死循环，因为unsigned int 和 int 比较。因此使用断言来证明非负。不要使用无符号类型来表示非负数。

## 预处理宏

最好使用内联函数、枚举和const量代替宏。在使用宏之前，考虑是否有非宏的解决方案。

如果你使用了宏，应该注意：

* 不要在头文件中定义宏
* 使用前正确定义它，使用后undef它
* 不要仅仅是为了替换为自己的而undef一个宏，应该取一个唯一的新名字
* 最好不要用##来生成函数、类、变量名

## 0 、nullptr、NULL

用0表示整形，0.0表示实数、nullptr（或NULL）表示指针，'\0'表示字符。

特别在一些情况下，sizeof(NULL)和sizeof(0)不同。

## sizeof

最好使用sizeof(varname)而非sizeof(type)。因为当var改变时，sizeof(type)不会改变而sizeof(varname)会跟着改变。

    <!-- lang: cpp -->
    Struct data;
    memset(&data, 0, sizeof(data));
    memset(&data, 0, sizeof(Struct));
    if (raw_size < sizeof(int)) {
      LOG(ERROR) << "compressed record not big enough for count: " << raw_size;
      return false;
    }

## auto

只在类型名十分混乱时使用auto，如果能增加可读性，继续使用完整地类型声明，除了局部变量外不要到处都用auto。

不要在文件域，命名空间域，类成员中使用auto。从不对大括号初始化列表使用auto。

## 括号初始化列表

一些例子：

    <!-- lang: cpp -->
    // Basically the same, ignoring some small technicalities.
    // You may choose to use either form.
    vector<string> v = {"foo", "bar"};
    
    // Usable with 'new' expressions.
    auto p = new vector<string>{"foo", "bar"};
    
    // A map can take a list of pairs. Nested braced-init-lists work.
    map<int, string> m = {{1, "one"}, {2, "2"}};
    
    // A braced-init-list can be implicitly converted to a return type.
    vector<int> test_function() { return {1, 2, 3}; }
    
    // Iterate over a braced-init-list.
    for (int i : {-1, -2, -3}) {}
    
    // Call a function using a braced-init-list.
    void TestFunction2(vector<int> v) {}
    TestFunction2({1, 2, 3});

也可以给自己的类型定义初始化列表：

    <!-- lang: cpp -->
    class MyType {
     public:
      // std::initializer_list references the underlying init list.
      // It should be passed by value.
      MyType(std::initializer_list<int> init_list) {
        for (int i : init_list) append(i);
      }
      MyType& operator=(std::initializer_list<int> init_list) {
        clear();
        for (int i : init_list) append(i);
      }
    };
    MyType m{2, 3, 5, 7};

在没有使用std::initializer_list<T>的情况下也可以：

    <!-- lang: cpp -->
    double d{1.23};
    // Calls ordinary constructor as long as MyOtherType has no
    // std::initializer_list constructor.
    class MyOtherType {
     public:
      explicit MyOtherType(string);
      MyOtherType(int, string);
    };
    MyOtherType m = {1, "b"};
    // If the constructor is explicit, you can't use the "= {}" form.
    MyOtherType m{"b"};

不要给{}使用auto：

    <!-- lang: cpp -->
    auto d = {1.23};        // d is a std::initializer_list<double>
    auto d = double{1.23};  // Good -- d is a double, not a std::initializer_list.

## lambda表达式

在适当的条件下用lambda表达式，不要使用默认的lambda捕获，把所有捕获明确地写出来。

如果匿名函数超过了5行，考虑给它起一个名字或者使用一个有名函数代替lambda表达式。

## C++11

合适的时候，用C++11编写的类库。在使用C++11之前，考虑好对其他环境的可移植性。

# 命名规则

## 普遍命名规则

函数名，变量名和文件名应该是具有描述性的，而不是缩略的。

    <!-- lang: cpp -->
    int price_count_reader;    // No abbreviation.
    int num_errors;            // "num" is a widespread convention.
    int num_dns_connections;   // Most people know what "DNS" stands for.
    int n;                     // Meaningless.
    int nerr;                  // Ambiguous abbreviation.
    int n_comp_conns;          // Ambiguous abbreviation.
    int wgc_connections;       // Only your group knows what this stands for.
    int pc_reader;             // Lots of things can be abbreviated "pc".
    int cstmr_id;              // Deletes internal letters.

## 文件名

文件名都该是小写的，并且可以包含下划线_或者短划线-。如果你的项目没有约定，最好用下划线_。

一些例子：

    <!-- lang: cpp -->
    my_useful_class.cc
    my-useful-class.cc
    myusefulclass.cc
    myusefulclass_test.cc // _unittest and _regtest are deprecated.
    
不要使用在/usr/include中已近存在的文件名，例如：db.h

普遍地，明确你的文件名。例如：http_server_logs.h 比 logs.h 好很多。

如果内联函数非常短，应该直接写在头文件中。

## 类型名

类型名以大写字母开头，并且每个词开头都是大写的：MyExcitingClass, MyExcitingEnum。

    <!-- lang: cpp -->
    // classes and structs
    class UrlTable { ...
    class UrlTableTester { ...
    struct UrlTableProperties { ...
    
    // typedefs
    typedef hash_map<UrlTableProperties *, string> PropertiesMap;
    
    // enums
    enum UrlTableErrors { ...

## 变量名

变量和数据成员都是小写的，单词间有下划线：a_local_variable, a_struct_data_member, a_class_data_member_。

    <!-- lang: cpp -->
    string table_name;  // OK - uses underscore.
    string tablename;   // OK - all lowercase.
    
    string tableName;   // Bad - mixed case.

类数据成员，末尾一个下划线：

    <!-- lang: cpp -->
    class TableInfo {
      ...
     private:
      string table_name_;  // OK - underscore at end.
      string tablename_;   // OK.
      static Pool<TableInfo>* pool_;  // OK.
    };

结构体数据成员，末尾没有下划线：

    <!-- lang: cpp -->
    struct UrlTableProperties {
      string name;
      int num_entries;
      static Pool<UrlTableProperties>* pool;
    };

全局变量，没有一个特定的规则。如果你需要一个规则，考虑给全局变量加上g_前缀，来区分局部变量。

## 常量名

在常量之前加上k，例如：const int kDaysInWeek = 7。

## 函数名

和变量命名方式相似，但可以是大小写混合：

    <!-- lang: cpp -->
    MyExcitingFunction(), MyExcitingMethod(), 
    my_exciting_member_variable(), set_my_exciting_member_variable().

访问器(get)和修改器(set)应该匹配要访问或修改的那个变量名：

    <!-- lang: cpp -->
    class MyClass {
     public:
      ...
      int num_entries() const { return num_entries_; }
      void set_num_entries(int num_entries) { num_entries_ = num_entries; }
    
     private:
      int num_entries_;
    };

## 命名空间名

全为小写，并且尽可能是表示目录结构：google_awesome_project.

## 枚举名

    <!-- lang: cpp -->
    enum UrlTableErrors {
      kOK = 0,
      kErrorOutOfMemory,
      kErrorMalformedInput,
    };
    enum AlternateUrlTableErrors {
      OK = 0,
      OUT_OF_MEMORY = 1,
      MALFORMED_INPUT = 2,
    };

## 宏名

通常情况下宏不应该被使用，然而它又绝对是需要的，宏名应该全为大写。

    <!-- lang: cpp -->
    #define ROUND(x) ...
    #define PI_ROUNDED 3.0

## 命名规则的例外

如果你在为现存的C/C++代码工作，依据现存的命名方式，例如：

    <!-- lang: cpp -->
    bigopen()
        function name, follows form of open()
    uint
        typedef
    bigpos
        struct or class, follows form of pos
    sparse_hash_map
        STL-like entity; follows STL naming conventions
    LONGLONG_MAX
        a constant, as in INT_MAX

# 注释

为你的读者写注释，因为下一个读者也许就是你。

## 注释风格

保持//和/**/的一致性，通常//更普遍。

## 文件注释

以许可协议（例如：Apache 2.0, BSD, LGPL, GPL）开头，接下来是关于内容的描述。

如果你对一个已近存在作者标记的文件进行了修改，请删除作者那一行。

不要重复在头文件和源文件中书写注释。

## 类注释

每个类定义都应该有一个用于描述它的作用，如何使用的注释：

    <!-- lang: cpp -->
    // Iterates over the contents of a GargantuanTable.  Sample usage:
    //    GargantuanTableIterator* iter = table->NewIterator();
    //    for (iter->Seek("foo"); !iter->done(); iter->Next()) {
    //      process(iter->key(), iter->value());
    //    }
    //    delete iter;
    class GargantuanTableIterator {
      ...
    };

## 函数注释

    <!-- lang: cpp -->
    // Returns an iterator for this table.  It is the client's
    // responsibility to delete the iterator when it is done with it,
    // and it must not use the iterator once the GargantuanTable object
    // on which the iterator was created has been deleted.
    //
    // The iterator is initially positioned at the beginning of the table.
    //
    // This method is equivalent to:
    //    Iterator* iter = table->NewIterator();
    //    iter->Seek("");
    //    return iter;
    // If you are going to immediately seek to another place in the
    // returned iterator, it will be faster to use NewIterator()
    // and avoid the extra seek.
    Iterator* GetIterator() const;

去掉不必要的注释：

    <!-- lang: cpp -->
    // Returns true if the table cannot hold any more entries.
    bool IsTableFull();
    
## 变量注释

应该说明这个变量用来干什么，在特定的情况下，需要更多的注释。例如：

    <!-- lang: cpp -->
    private:
     // Keeps track of the total number of entries in the table.
     // Used to ensure we do not go over the limit. -1 means
     // that we don't yet know how many entries the table has.
     int num_total_entries_;

所有的全局变量都应该给出一个注释，来描述它用来做什么。

    <!-- lang: cpp -->
    // The total number of tests cases that we run through in this regression test.
    const int kNumTestCases = 6;

## 实现注释

一些难懂的，复杂的代码块应该被注释：

    <!-- lang: cpp -->
    // Divide result by two, taking into account that x
    // contains the carry from the add.
    for (int i = 0; i < result->size(); i++) {
      x = (x << 8) + (*result)[i];
      (*result)[i] = x >> 1;
      x &= 1;
    }

意义不明显的行末应该空两个字符并给出注释：

    <!-- lang: cpp -->
    // If we have enough memory, mmap the data portion too.
    mmap_budget = max<int64>(0, mmap_budget - index_->length());
    if (mmap_budget >= data_size_ && !MmapData(mmap_chunk_bytes, mlock))
      return;  // Error already logged.

如果接下来的几行都有注释，最好排列起来增强可读性：

    <!-- lang: cpp -->
    DoSomething();                           // Comment here so the comments line up.
    DoSomethingElseThatIsLonger();  // Two spaces between the code and the comment.
    { // One space before comment when opening a new scope is allowed,
      // thus the comment lines up with the following comments and code.
      DoSomethingElse();  // Two spaces before line comments normally.
    }
    vector<string> list{// Comments in braced lists describe the next element ..
                        "First item",
                        // .. and should be aligned appropriately.
                        "Second item"};
    DoSomething(); /* For trailing block comments, one space is fine. */

当你传递一个空指针，布尔值或者字面整形值时，你应该考虑添加注释来说明他们是什么，或者使你的代码自我注释：

    <!-- lang: cpp -->
    bool success = CalculateSomething(interesting_value,
                                      10,
                                      false,
                                      NULL);  // What are these arguments??
    versus:
    
    bool success = CalculateSomething(interesting_value,
                                      10,     // Default base value.
                                      false,  // Not the first time we're calling this.
                                      NULL);  // No callback.
    Or alternatively, constants or self-describing variables:
    
    const int kDefaultBaseValue = 10;
    const bool kFirstTimeCalling = false;
    Callback *null_callback = NULL;
    bool success = CalculateSomething(interesting_value,
                                      kDefaultBaseValue,
                                      kFirstTimeCalling,
                                      null_callback);

切记不要描述代码自身：

    <!-- lang: cpp -->
    // Now go through the b array and make sure that if i occurs,
    // the next element is i+1.
    ...        // Geez.  What a useless comment.

## 标点，拼写和语法

完整的句子往往比句子片段更容易理解。

## TODO 注释

使用TODO注释是一个临时的，短期的解决方案，很好的但不是完美的。

当你创建一个TODO时，总是给出你的名字：

    <!-- lang: cpp -->
    // TODO(kl@gmail.com): Use a "*" here for concatenation operator.
    // TODO(Zeke) change this to use relations.

## 弃用注释

使用DEPRECATED注释标记一个弃用的接口。

在DEPRECATED之后写出你的名字，email地址或其他能标识你的信息。

弃用注释必须包含简易的，清楚的指引来帮助使用者修复他们的问题。C++中，你可以将弃用的方法放在内联函数中，然后调用新的接口。

# 格式化

## 行长度

一行最多80个字符。

一些原始字符串可能会超出80个字符，除了测试外，这样的字符串应该出现在该文件的顶部。

#include 语句可能会超过80列。

## 非ascii字符

非ascii字符很少出现，但必须使用UTF-8编码。

十六进制也可以使用，在那些需要加强可读性的地方更建议使用。

你不应该使用C++11提供的char116_t和char32_t，因为他们用于非UTF8得文本。同样的，，你也不应该使用wchar_t，除非你是在用Windows API编写程序。

## 空格和制表符

只用空格并且缩进两个字符。

## 函数声明和定义

返回类型出现在函数名的同一行，如果能适应，参数也出现在同一行。如果不能适应，折行书写参数表。

    <!-- lang: cpp -->
    ReturnType ClassName::FunctionName(Type par_name1, Type par_name2) {
      DoSomething();
      ...
    }
    
    If you have too much text to fit on one line:
    
    ReturnType ClassName::ReallyLongFunctionName(Type par_name1, Type par_name2,
                                                 Type par_name3) {
      DoSomething();
      ...
    }
    
    or if you cannot fit even the first parameter:
    
    ReturnType LongClassName::ReallyReallyReallyLongFunctionName(
        Type par_name1,  // 4 space indent
        Type par_name2,
        Type par_name3) {
      DoSomething();  // 2 space indent
      ...
    }

需要指出：

* 如果不能把返回类型和函数名放在同一行，请折行
* 如果折行书写了返回类型，不要缩进
* 左圆括号始终和函数名处在同一行
* 括号和参数之间没有空格
* 左花括号始终和最后一个参数在同一行
* 右花括号在最后一行或者和左花括号在同一行
* 右小括号和左花括号之间有一个空格
* 所有参数都改命名，无论是在头文件，或者源文件中
* 如果可能，所有参数都要对齐
* 默认缩进2个字符
* 折行的参数有4个字符缩进

如果一些参数未使用，在函数声明处注释出来：

    <!-- lang: cpp -->
    // Always have named parameters in interfaces.
    class Shape {
     public:
      virtual void Rotate(double radians) = 0;
    };
    
    // Always have named parameters in the declaration.
    class Circle : public Shape {
     public:
      virtual void Rotate(double radians);
    };
    
    // Comment out unused named parameters in definitions.
    void Circle::Rotate(double /*radians*/) {}
    // Bad - if someone wants to implement later, it's not clear what the
    // variable means.
    void Circle::Rotate(double) {}

## lambda表达式

像其他函数一样格式化参数和函数体，捕获表如其他逗号分隔的列表。

    <!-- lang: cpp -->
    int x = 0;
    auto add_to_x = [&x](int n) { x += n; };

简短的lambda表达式应该作为函数参数内联：

    <!-- lang: cpp -->
    std::set<int> blacklist = {7, 8, 9};
    std::vector<int> digits = {3, 9, 1, 8, 4, 7, 1};
    digits.erase(std::remove_if(digits.begin(), digits.end(), [&blacklist](int i) {
                   return blacklist.find(i) != blacklist.end();
                 }),
                 digits.end());
    
## 函数调用

将函数调用写在一行，用小括号包裹参数；或者将参数置于新行，用4个空格缩进。使用最小的行数。

如下格式：

    <!-- lang: cpp -->
    bool retval = DoSomething(argument1, argument2, argument3);

如果参数太多，折行书写，括号左右不要有空格。

    <!-- lang: cpp -->
    bool retval = DoSomething(averyveryveryverylongargument1,
                                            argument2, argument3);
    
    if (...) {
      ...
      ...
      if (...) {
        DoSomething(
            argument1, argument2,  // 4 space indent
            argument3, argument4);
      }

## 花括号初始化列表格式

    <!-- lang: cpp -->
    // Examples of braced init list on a single line.
    return {foo, bar};
    functioncall({foo, bar});
    pair<int, int> p{foo, bar};
    
    // When you have to wrap.
    SomeFunction(
        {"assume a zero-length name before {"},
        some_other_function_parameter);
    SomeType variable{
        some, other, values,
        {"assume a zero-length name before {"},
        SomeOtherType{
            "Very long string requiring the surrounding breaks.",
            some, other values},
        SomeOtherType{"Slightly shorter string",
                      some, other, values}};
    SomeType variable{
        "This is too long to fit all in one line"};
    MyType m = {  // Here, you could also break before {.
        superlongvariablename1,
        superlongvariablename2,
        {short, interior, list},
        {interiorwrappinglist,
         interiorwrappinglist2}};

## 条件语句

最好括号内无空格。if和else关键字属于分立的行中。

    <!-- lang: cpp -->
    if(condition) {   // Bad - space missing after IF.
    if (condition){   // Bad - space missing before {.
    if(condition){    // Doubly bad.
    if (condition) {  // Good - proper space after IF and before {.

简短的条件块可以写在一行，如果能强化可读性。

    <!-- lang: cpp -->
    if (x == kFoo) return new Foo();
    if (x == kBar) return new Bar();

如果含有else块，则不允许写在一行：

    <!-- lang: cpp -->
    // Not allowed - IF statement on one line when there is an ELSE clause
    if (x) DoThis();
    else DoThat();

一般而言，单行条件语句不需要花括号，如果你喜欢也可以加上，特别是在循环中存在复杂的条件时，使用花括号可增加可读性。有些项目还要求if块必须含有完整地括号对。

然而，if-else一部分使用了花括号，那么所有块都必须使用：

    <!-- lang: cpp -->
    // Not allowed - curly on IF but not ELSE
    if (condition) {
      foo;
    } else
      bar;
    
    // Not allowed - curly on ELSE but not IF
    if (condition)
      foo;
    else {
      bar;
    }
    // Curly braces around both IF and ELSE required because
    // one of the clauses used braces.
    if (condition) {
      foo;
    } else {
      bar;
    }

## 循环和选择语句

switch块必须包含default，如果default永不会被执行，则写一个assert：

    <!-- lang: cpp -->
    switch (var) {
      case 0: {  // 2 space indent
        ...      // 4 space indent
        break;
      }
      case 1: {
        ...
        break;
      }
      default: {
        assert(false);
      }
    }

空的case块用花括号括起来。

单语句循环，花括号可以省略，空循环用花括号括起来或者写continue，而不仅仅是分号。

    <!-- lang: cpp -->
    while (condition) {
      // Repeat test until it returns false.
    }
    for (int i = 0; i < kSomeNumber; ++i) {}  // Good - empty body.
    while (condition) continue;  // Good - continue indicates no logic.
    while (condition);  // Bad - looks like part of do/while loop.

## 指针和引用表达式

句号和箭头周围无空格：

    <!-- lang: cpp -->
    x = *p;
    p = &x;
    x = r.y;
    x = r->y;

当声明一个指针类型变量时，保证星号两侧仅有一个空格：

    <!-- lang: cpp -->
    // These are fine, space preceding.
    char *c;
    const string &str;
    
    // These are fine, space following.
    char* c;    // but remember to do "char* c, *d, *e, ...;"!
    const string& str;
    char * c;  // Bad - spaces on both sides of *
    const string & str;  // Bad - spaces on both sides of &

## 布尔表达式

当一个布尔表达式超过标准行长度(80)时，保证折行书写的一致性。

    <!-- lang: cpp -->
    if (this_one_thing > this_other_thing &&
        a_third_thing == a_fourth_thing &&
        yet_another && last_one) {
      ...
    }

## 返回值

不要给return语句加上括号。仅在返回一个逻辑表达式的时候加括号。

    <!-- lang: cpp -->
    return result;                  // No parentheses in the simple case.
    // Parentheses OK to make a complex expression more readable.
    return (some_long_condition &&
            another_condition);
    return (value);                // You wouldn't write var = (value);
    return(result);                // return is not a function!

## 变量和数组初始化

可以选择=，()或者{}。下面都是正确的：

    <!-- lang: cpp -->
    int x = 3;
    int x(3);
    int x{3};
    string name = "Some Name";
    string name("Some Name");
    string name{"Some Name"};

当心{}会调用std::initializer_list 构造函数。为了避免这个问题，使用()：

    <!-- lang: cpp -->
    vector<int> v(100, 1);  // A vector of 100 1s.
    vector<int> v{100, 1};  // A vector of 100, 1.

## 预处理指令

总是在一行的开头书写，无论是不是在代码块内：

    <!-- lang: cpp -->
    // Good - directives at beginning of line
      if (lopsided_score) {
    #if DISASTER_PENDING      // Correct -- Starts at beginning of line
        DropEverything();
    # if NOTIFY               // OK but not required -- Spaces after #
        NotifyClient();
    # endif
    #endif
        BackToNormal();
      }
    // Bad - indented directives
      if (lopsided_score) {
        #if DISASTER_PENDING  // Wrong!  The "#if" should be at beginning of line
        DropEverything();
        #endif                // Wrong!  Do not indent "#endif"
        BackToNormal();
      }

## 类格式化

public,protected,private前面有一个空格：

    <!-- lang: cpp -->
    class MyClass : public OtherClass {
     public:      // Note the 1 space indent!
      MyClass();  // Regular 2 space indent.
      explicit MyClass(int var);
      ~MyClass() {}
    
      void SomeFunction();
      void SomeFunctionThatDoesNothing() {
      }
    
      void set_some_var(int var) { some_var_ = var; }
      int some_var() const { return some_var_; }
    
     private:
      bool SomeInternalFunction();
    
      int some_var_;
      int some_other_var_;
    };

注意：

* 基类名和子类名放在一行
* 除了第一个public，其他关键字前需要空一行，这个规则在小型类中可选。
* 这些关键字下面不要有空行

## 构造函数初始化列表

初始化列表可以在一行内，也可以折行，前面缩进4个字符：

    <!-- lang: cpp -->
    // When it all fits on one line:
    MyClass::MyClass(int var) : some_var_(var), some_other_var_(var + 1) {}
    or
    
    // When it requires multiple lines, indent 4 spaces, putting the colon on
    // the first initializer line:
    MyClass::MyClass(int var)
        : some_var_(var),             // 4 space indent
          some_other_var_(var + 1) {  // lined up
      ...
      DoSomething();
      ...
    }

## 命名空间格式

命名空间内的内容不缩进。

    <!-- lang: cpp -->
    namespace {
    
    void foo() {  // Correct.  No extra indentation within namespace.
      ...
    }
    
    }  // namespace
    Do not indent within a namespace:
    
    namespace {
    
      // Wrong.  Indented when it should not be.
      void foo() {
        ...
      }
    
    }  // namespace

## 水平空格

水平空格依赖于位置，不要行末添加空格。

    <!-- lang: cpp -->
    void f(bool b) {  // Open braces should always have a space before them.
      ...
    int i = 0;  // Semicolons usually have no space before them.
    // Spaces inside braces for braced-init-list are optional.  If you use them,
    // put them on both sides!
    int x[] = { 0 };
    int x[] = {0};
    
    // Spaces around the colon in inheritance and initializer lists.
    class Foo : public Bar {
     public:
      // For inline function implementations, put spaces between the braces
      // and the implementation itself.
      Foo(int b) : Bar(), baz_(b) {}  // No spaces inside empty braces.
      void Reset() { baz_ = 0; }  // Spaces separating braces from implementation.
      ...

循环和条件，

    <!-- lang: cpp -->
    if (b) {          // Space after the keyword in conditions and loops.
    } else {          // Spaces around else.
    }
    while (test) {}   // There is usually no space inside parentheses.
    switch (i) {
    for (int i = 0; i < 5; ++i) {
    // Loops and conditions may have spaces inside parentheses, but this
    // is rare.  Be consistent.
    switch ( i ) {
    if ( test ) {
    for ( int i = 0; i < 5; ++i ) {
    // For loops always have a space after the semicolon.  They may have a space
    // before the semicolon, but this is rare.
    for ( ; i < 5 ; ++i) {
      ...
    
    // Range-based for loops always have a space before and after the colon.
    for (auto x : counts) {
      ...
    }
    switch (i) {
      case 1:         // No space before colon in a switch case.
        ...
      case 2: break;  // Use a space after a colon if there's code after it.

操作符，

    <!-- lang: cpp -->
    // Assignment operators always have spaces around them.
    x = 0;
    
    // Other binary operators usually have spaces around them, but it's
    // OK to remove spaces around factors.  Parentheses should have no
    // internal padding.
    v = w * x + y / z;
    v = w*x + y/z;
    v = w * (x + z);
    
    // No spaces separating unary operators and their arguments.
    x = -5;
    ++x;
    if (x && !y)
      ...

模板和类型转换，

    <!-- lang: cpp -->
    // No spaces inside the angle brackets (< and >), before
    // <, or between >( in a cast
    vector<string> x;
    y = static_cast<char*>(x);
    
    // Spaces between type and pointer are OK, but be consistent.
    vector<char *> x;
    set<list<string>> x;        // Permitted in C++11 code.
    set<list<string> > x;       // C++03 required a space in > >.
    
    // You may optionally use symmetric spacing in < <.
    set< list<string> > x;

## 纵向空格

使纵向空格最小化。

当你没必要时，不要留很多空行。特别的，在函数间不要留超过一行或两行的空白。

函数开始不要有空行，函数结束也不要有空行。

基本原则：一个屏幕上有越多的代码，就越容易理解程序的控制流程。当然，可读性也很重要。

# 以上规则的例外情况

## 现存的不一致代码

你可能会在不符合本规则的代码上工作，为了保持一致性，你不应该生搬硬套这个规则。

## Windows代码

* 不要使用匈牙利命名法，例如 iNum。使用上面介绍的命名规则。
* Windows定义了许多自己的类型，如DWORD，HANDLE等等。即便这样，你也应该使用你熟知的C++类型，例如：const TCHAR * 来代替LPCTSTR。
* 当使用Microsoft Visual C++编译时，将编译器设置3或以上的警告等级，并把所有警告视为错误。
* 不要使用#pragma once。使用第一个介绍的头文件保护策略。
* 实际上，不要使用任何非标准的扩展指令，像#pragma和__declspec，除非你必须使用。允许使用__declspec(dllimport) 和 __declspec(dllexport)。当然，你必须通过DLLIMPORT和DLLEXPORT宏来间接使用它们。

上面的一些规则在Windows上不适用：

* 禁止多重实现继承。在使用COM和一些ATL/WTL类时，多重继承是必要的。
* 不使用exception。在ATL和一些Visual C++的STL中使用了exception。使用ATL时，你应该定义_ATL_NO_EXCEPTIONS来禁用异常。
* 使用预编译头文件的一贯方法是包含StdAfx.h或者precompile.h。你应该避免手动包含预编译头文件，使用/FI编译器选项来自动包含这个文件。
* 资源头文件resource.h，仅包含宏，不需要套用本规则。

# 最后的话

符合常识以及保持一致性。

在编辑代码之前，花几分钟看看当前代码的编码风格。并与之保持一致的风格。
