---
title: C++我也来写个工厂模式
date: 2014-08-28 00:00:00
tags:
  - tech
  - cpp
---

> 工厂方法模式（Factory method pattern）是一种实现了“工厂”概念的面向对象设计模式。就像其他创建型模式一样，它也是处理在不指定对象具体类型的情况下创建对象的问题。工厂方法模式的实质是“定义一个创建对象的接口，但让实现这个接口的类来决定实例化哪个类。工厂方法让类的实例化推迟到子类中进行。

以前是没有实现过工厂模式，这里我用到了template来创建类型不同的Products，内存管理这块没想到更好的办法来cleanup，打算是利用析构自动release，不过貌似到模版里就捉禁见肘了。。大家有什么高见？

```cpp
#include <iostream>
#include <vector>

// ProductA

class ProductA
{
public:
	void name();
};

void ProductA::name()
{
	std::cout << "I'm Product A." << std::endl;
}

// ProductB

class ProductB
{
public:
	void name();
};

void ProductB::name()
{
	std::cout << "I'm Product B." << std::endl;
}


// Factory
template<typename T>
class Factory
{
public:
	static T* create();
	static void cleanup();
private:
	Factory();
	static std::vector<T*> objs_;
};
template<typename T>
std::vector<T*> Factory<T>::objs_;

template<typename T>
Factory<T>::Factory()
{
}

template<typename T>
void Factory<T>::cleanup()
{
	for each (T* obj in objs_)
		if (obj)
		{
		std::cout << "release " << obj << std::endl;
		delete obj;
		obj = nullptr;
		}
	objs_.clear();
}

template<typename T>
T* Factory<T>::create()
{
	T * obj = new T;
	objs_.push_back(obj);

	return obj;
}

int main(int argc, char *argv[])
{
	//create 10 ProductAs
	for (size_t i = 1; i <= 10; i++)
	{
		auto pa = Factory<ProductA>::create();
		pa->name();
	}

	//create 10 ProductBs
	for (size_t i = 1; i <= 10; i++)
	{
		auto pb = Factory<ProductB>::create();
		pb->name();
	}

	//free memory
	Factory<ProductA>::cleanup();
	Factory<ProductB>::cleanup();

	return 0;
```
