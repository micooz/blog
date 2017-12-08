---
title: C++使用模板类实现任意类型switch和变量case
date: 2015-06-02 00:00:00
tags:
  - tech
  - cpp
---

最近自己维护的一个项目[program_options](https://github.com/micooz/program_options)(是一个命令行生成与解析的C++库)在实际应用的时候遇到一个需求：

需要switch一个字符串来执行相应代码块，然而原生的switch-case条件选择语法针对condition有严格的限制，下面摘录一段switch的语法标准：

## switch statement

Transfers control to one of the several statements, depending on the value of a condition.

Syntax

```
attr(optional) switch ( condition ) statement	

attr(C++11)	-	any number of attributes
condition	-	any expression of integral or enumeration type, or of a class type contextually implicitly convertible to an integral or enumeration type, or a declaration of a single non-array variable of such type with a brace-or-equals initializer.
statement	-	any statement (typically a compound statement). case: and default: labels are permitted in statement and break; statement has special meaning.

attr(optional) case constant_expression : statement	(1)	
attr(optional) default : statement	(2)	

constant_expression	-	a constant expression of the same type as the type of condition after conversions and integral promotions
```

### condition

原生switch语法的condition支持整数，枚举或者根据上下文能够隐式转换为整数或者枚举的类，再或者是非数组类型的=或{}初始化语句，举例来说就是如下四类：

```cpp
switch(100)
enum color {r, g, b}
switch(r)
switch(int n = 1)
switch(int n = {1})
```

### case

必须是常量，这样一来就无法做变量与变量之间的比较。

### statement

当statement没有被{}包围的时候，在其内使用声明语句会导致编译错误。比如：

```cpp
switch(num){
  case 1:
    int n = 0; // error: jump bypasses variable initialization
    break;
  default:
    break;
}
```

显然我的需求不能用原生的switch来实现，得老老实实if-elseif-elseif...来分别判断，导致代码又长又臭，if越套越多，逐条判断效率也让人心塞。

有需求就有想法，有想法就有创新，相信这样的需求早就有人实现过了，比如：

* 为C++添加短字符串的switch-case支持
  将字符串通过#@转换为整形再进行case
* 使用C++11新特性，实现用字符串作为switch的case子句
  利用C++11的constexpr，计算字符串的hash值再进行case

想法很不错，但是我想要更灵活的解决方案，我希望新的switch支持switch(object)，case(object)，还希望statement对变量声明没有限制，何不完全抛开原生switch的枷锁，自己利用标准库造一个switch来解决问题呢。

## 蓝图

我希望利用C++强大的**template**来兼容任意类型，用C++11的**lambda**匿名函数实现statement，用**操作符重载**operator==来匹配条件，用**hash表**来提升匹配效率，看起来很容易不是吗？

开始Coding之前我先拟定好蓝图：

```cpp
// 蓝图1
select(condition, {
  {"case1", []() {
    // code goes here
  }},
  {"case2", []() {
    // code goes here
  }}
});
// 蓝图2
select(condition)
  .found("case1", []() {
    // code goes here
  })
  .found("case2", []() {
    // code goes here
  })
  .others([]() {
    // default
  });
```

我承认我是受到了javascript的影响，我一直以为C++越来越像是一种高级的脚本语言，或许也是它未来的发展趋势。

蓝图的设计首先符合C++的语法规范，没有语法错误，其次力求语义明确，简洁。

蓝图1的大括号太多，书写时容易出错。

蓝图2语法简洁明了，我相信任何会闭包的Coder都能理解。

## 实现

有了蓝图后我们就可以照着这个模样来写代码了，首先分析一下蓝图2。

* 存在链式操作，显然select函数要返回一个对象，该对象有found和others方法，并且，found方法要返回实例本身。
* condition和found的第一个参数类型必须一致，但不一定是string，也可以是int，Object，可用template实现
* found第二个参数是lambda表达式，类型是std::function<...>，类似C里面的函数指针，可定义为回调函数。
* 每个found块对应于switch里的case，是一个kv关系，可用std::map来存储关联。

C++建议模板类的声明和定义必须写在同一个文件里，因此起一个**switch.hpp**文件：

```cpp
#include <functional>
#include <map>

template <typename Ty>
class Switch {
 public:
  Switch(){}

  explicit Switch(const Ty& target)
      : target_(target) {}

  Switch& found(const Ty& _case, const std::function<void(void)>& callback) {
    reflections_[_case] = callback;
    return *this;
  }
  
 private:
  const Ty& target_;
  std::map<const Ty, std::function<void(void)>> reflections_;
};

template <typename Ty>
Switch<Ty> select(const Ty& expression) {
  return Switch<Ty>(expression);
}
```

这么一来就实现了found得链式操作，存储了kv对，全局(也可以在某个命名空间内)select函数是一个简化书写的帮助函数，创建对象后返回该对象的拷贝，实现了如下调用：

```cpp
select(std::string("condition"))
  .found ...
```

接下来我需要实现查找到对应的target，然后调用它的callback。

增加一个done()方法，该方法被调用意味着结束整个Switch，开始匹配found块，如果没找到，调用others函数(对应default块)：

```cpp
  inline void done() {
    auto kv = reflections_.find(target_);
    if (kv != reflections_.end()) {
      // found
      auto scope = kv->second;
      scope();
    } else if (has_others_scope_) {
      // not found, call others
      others_();
    }
  }
```

std::map的find方法时间复杂度是O(logN)，而原生switch匹配时间复杂度是O(1)，肯定是有很大差距的，但是为了实现switch没有的功能，这点损失也是十分值得的。

others方法如下：

```cpp
  inline void others(const Scope& callback) {
    has_others_scope_ = true;
    others_ = callback;
    this->done();
  }
```

当用户调用others方法定义了default块之后，就没必要再调用done了，又可以减少7个字符的书写。

这里has_others_scope_为bool成员；others_是单独存放的lambda表达式成员，为了简化查找，不宜放在reflections_中。

再简化书写，用typedef缩短类型，然后替换原类中相应类型为短类型：

```cpp
  typedef std::function<void(void)> Scope;
  typedef std::map<const Ty, Scope> Reflections;
```

这么一来几乎很完美了，全新的Switch如下：

```cpp
  #define printl(line) printf((line)); printf("\n")
  std::string condition("windows");

  // match std::string
  select(condition)
    .found("apple", []() {
      printl("it's apple");
    })
    .found("windows", []() {
      printl("it's windows");
    })
    .found("linux", []() {
      printl("it's linux");
    }).done();

  // match int
  select(100)
    .found(10, []() {
      printl("it's 10");
    })
    .found(20, []() {
      printl("it's 20");
    })
    .others([]() {
      printl("nothing found");
    });

  // output
  // it's windows
  // nothing found
```

我想进一步实现**自定义class**的case，定义一个User类：

```cpp
class User {
 public:
  explicit User(int age) : age_(age) {}

  bool operator<(const User& user) const { return this->age_ < user.age(); }

  int age() const { return age_; }

 private:
  int age_;
};
```

Switch如下：

```cpp
User u1(20), u2(22), ux(20);
  select(ux)
    .found(u1, []() {
      printl("it's u1");
    })
    .found(u2, []() {
      printl("it's u2");
    }).done();
  // it's u2
```

非常有必要说明的是这个重载：

```cpp
bool operator<(const User& user) const { return this->age_ < user.age(); }
```

返回bool没有问题，但为什么必须是operator<呢，原因在这句：

```cpp
auto kv = reflections_.find(target_);
```

std::map<>::find不是通过==进行查找的，而是<，因此必须重载<。

该重载必须被const修饰，原因也是find这句里面，const对象只能调用const方法。

标准库里的实现如下：

```cpp
struct _LIBCPP_TYPE_VIS_ONLY less : binary_function<_Tp, _Tp, bool>
{
    _LIBCPP_CONSTEXPR_AFTER_CXX11 _LIBCPP_INLINE_VISIBILITY 
    bool operator()(const _Tp& __x, const _Tp& __y) const
        {return __x < __y;}
};
```

可以非常明显的看到const和<。

此外我还实现了Switch之间的found块组合，比较简单就不阐述了。

## 存在的问题

常量字符串的转型问题：

```cpp
select("condition")
  .found("case", ...)
  .done();
```

编译器将"condition"理解为const char[10]，数组类型有固定长度，found块的_case参数类型是const char[5]，导致编译错误。原因在于：

```cpp
Switch& found(const Ty& _case, const Scope& callback)
```

这里传递const引用，因此编译器把"case"当做了const char[5]。此时Ty的类型和说好的const char[10]不一致，编译失败。

解决方法是通过std::string来避免数组长度不匹配问题：

```cpp
select(std::string("condition"))
  .found("case", ...)
  .done();
```

希望读者有更好地解决方案。

## 完整代码

这里直接引用我项目里面的实现：

```cpp
#ifndef PROGRAM_OPTIONS_SWITCH_HPP_
#define PROGRAM_OPTIONS_SWITCH_HPP_

#include <functional>
#include <map>

namespace program_options {

/**
* @brief The Switch template class.
* @param Ty The target type.
*/
template <typename Ty>
class Switch {
 public:
  typedef std::function<void(void)> Scope;
  typedef std::map<const Ty, Scope> Reflections;

  Switch() : has_others_scope_(false) {}

  explicit Switch(const Ty& target)
      : target_(target), has_others_scope_(false) {}

  /**
   * @brief Create a case block with an expression and a callback function.
   * @param _case The case expression, variable is allowed.
   * @param callback The callback function, can be a lambda expression.
   * @return The current Switch instance.
   */
  Switch& found(const Ty& _case, const Scope& callback) {
    reflections_[_case] = callback;
    return *this;
  }

  /**
   * @brief Create a default block with a callback function,
   *        if no cases matched, this block will be called.
   * @param callback
   */
  inline void others(const Scope& callback) {
    has_others_scope_ = true;
    others_ = callback;
    this->done();
  }

  /**
   * @brief Finish the cases,
   * others() will call this method automatically.
   */
  inline void done() {
    auto kv = reflections_.find(target_);
    if (kv != reflections_.end()) {
      // found
      auto scope = kv->second;
      scope();
    } else if (has_others_scope_) {
      // not found, call others
      others_();
    }
  }

  /**
   * @brief Combine the cases to this Switch from another Switch.
   *        Note that this two Switch should be the same template.
   * @param _switch Another Switch instance.
   * @return
   */
  inline Switch& combine(const Switch& _switch) {
    for (auto kv : _switch.reflections()) {
      this->reflections_[kv.first] = kv.second;
    }
    return *this;
  }

  /**
   * @brief Return the case-callback pairs.
   * @return
   */
  inline Reflections reflections() const { return reflections_; }

 private:
  const Ty& target_;
  bool has_others_scope_;
  Scope others_;
  Reflections reflections_;
};

/**
 * @brief Define which expression does the Switch match.
 * @param expression
 * @return
 */
template <typename Ty>
Switch<Ty> select(const Ty& expression) {
  return Switch<Ty>(expression);
}
}

#endif  // PROGRAM_OPTIONS_SWITCH_HPP_
```

欢迎各位读者指正。
