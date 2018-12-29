---
title: C++右值引用
date: 2018-12-28 10:44:08
tags: C++
---

本文主要学习于[从4行代码看右值引用](https://www.cnblogs.com/qicosmos/p/4283455.html)

文章的主线是4行代码的故事...写博客的过程中感觉文脉有点混乱，但是其中的很多概念都是比较清晰的

> 和右值引用相关的概念比较多，比如：右值、纯右值、将亡值、universal references、引用折叠、移动语义、move语义和完美转发等等

引入右值引用主要为了解决C++98/03遇到的两个问题

> 1、临时对象非必要的昂贵的拷贝操作
>
> 2、模板函数中如何按照参数的实际类型进行转发



# 第一行代码

`int i = getVar();`

代码很简单，从getVar()函数获取一个整形值

然而，这行代码会产生两种类型的值，一种是左值i，一种是getVar()返回的**临时值**，这个临时值在表达式结束之后就销毁了

这个**临时值**就是一个**右值(rvalue)**，具体来说是一个**纯右值(prvalue)**，是不具名的

> 区分左值和右值的简单方法：看能不能对表达式取地址，如果能，则为左值；反之则为右值

在C++11中所有的值必属于**左值、将亡值、纯右值**三者之一

比如，非引用返回的临时变量、运算表达式产生的临时变量、原始字面量和lambda表达式都是纯右值

而**将亡值**是C++11新增的、与右值引用相关的表达式，比如，将要被移动的对象、T&&函数返回值、`std::move`返回值和转换为T&&的类型的转换函数的返回值等

**将亡值**将在后文中介绍

```C++
int j = 5;
auto f = []{ return 5;};
```

代码中5是一个原始字面量，`[]{ return 5;}`是lambda表达式，都属于纯右值，在表达式结束之后就销毁了



# 第二行代码

## 右值引用的特点

### 特点1

`T&& k = getVar();`

相比第一行多了"&&"，即右值引用

表达式结束后，getVar()产生的临时值不会被销毁，而是会被"**续命**"(......)，它的生命周期将会通过右值引用得以延续，和变量k的生命周期一样长

> 这里不展开了，《深度探索C++对象模型》中探讨过**NRV(Named Return Value)**优化

事实上，**常量左值引用**在C++98/03中也经常用于性能优化，因为**常量左值引用**是一个**万能**的引用类型，可以接受左值、右值、常量左值和常量右值

注意是`const A& a = GetA();`必须加上**const**

而`A& a = GetA();`会报一个编译错误，因为**非常量左值引用只能接受左值**



### 特点2

右值引用独立于左值和右值，意思是右值引用类型的变量可能是左值也可能是右值

举个例子 `int** var1 = 1;`

var1为右值引用，但var1本身是左值，因为具名变量都是左值

有一个有意思的问题，T&&是什么？一定是右值吗？代码如下

```C++
template<typename T>
void f(T&& t){}

f(10); //t是右值

int x = 10;
f(x); //t是左值
```

从代码中可以看到，T&&表示的值类型**不确定**，可能是左值也可能是右值，这一点看起来有点奇怪，但这就是右值引用的一个**特点**

### 特点3

`T&& t`在发生自动类型推断时，它是**未定的引用类型(universal references)**

如果被左值初始化那么它就是左值，如果被右值初始化那么它就是右值

~~你说你证明了哥德巴赫猜想，那你就是证明了哥德巴赫猜想；你说你没有证明哥德巴赫猜想，那你就是没有证明哥德巴赫猜想~~

它是左值还是右值取决于它的初始化

> 需要注意的是，仅仅是当发生**自动类型推导**（如函数模板的类型推断，或auto关键字）时，T&&才是**universal references**

再看看以下的例子

```C++
template<typename T>
void f(T&& param); 

template<typename T>
class Test {
    Test(Test&& rhs); 
};
```

param是universal reference，而rhs是Test&&右值引用

因为模板函数f发生了类型推导，而Test&&并没有发生类型推导，因为Test&&是确定的类型了

（再看一下特点2的例子就基本明白了）

> 根据这个特点，我们可以用来做很多事情，比如下文要介绍的**移动语义**和**完美转发**

再提一下引用折叠，可能存在左值引用和右值引用、右值引用和右值引用的折叠，C++11确定了引用折叠的规则

> 所有右值引用叠加到右值引用上仍然是一个右值引用
>
> 所有其他类型的引用之间的叠加都将变成左值引用



# 第三行代码

## 移动构造函数

`T(T&& a) : m_val(val){ a.m_val = nullptr; }`

这行代码实际来自一个类的构造函数，参数是一个右值引用

```C++
class A {
public:
	A() :m_ptr(new int(0)) { cout << "construct" << endl; }
	A(const A& a) :m_ptr(new int(*a.m_ptr)) {
		//深拷贝的拷贝构造函数
		cout << "copy construct" << endl;
	}
	~A() { delete m_ptr; }
private:
	int* m_ptr;
};
```

一个带有堆内存的类，必须提供一个深拷贝构造函数，否则会发生**指针悬挂**问题（类似double free）

提供深拷贝的拷贝构造函数虽然可以保证正确，但在有时候会产生额外的性能损耗

C++11给出了**移动构造函数**的解决方案

```C++
class A {
public:
	A() :m_ptr(new int(0)) {}
	A(const A& a) :m_ptr(new int(*a.m_ptr)) { //深拷贝的拷贝构造函数
		cout << "copy construct" << endl;
	}
	A(A&& a) :m_ptr(a.m_ptr) {
		a.m_ptr = nullptr;
		cout << "move construct" << endl;
	}
	~A() { delete m_ptr; }
private:
	int* m_ptr;
};
```

` A a = Get(false); `

输出结果会是

```C++
construct
move construct
move construct
```

相比之下只多了一个构造函数，拷贝并没有调用深拷贝构造函数，而是调用了**移动构造函数**

它仅仅是将指针的**所有者**转移到了另外一个对象，同时将参数a的指针置为空（安全隐患？）

移动构造函数的参数是一个右值引用类型，前面已经提到，这里并没有发生**类型推断**，是确定的右值引用

> 为什么会匹配到这个构造函数？
>
> 因为这个构造函数只能接受右值参数，而函数返回值是右值（暂时没怎么看懂）

这里A&&可以看作是临时值的标识

**待更细节**



> 需要注意的是，在提供移动构造函数时也会提供一个拷贝构造函数，以防移动不成功时还能拷贝构造



## std::move

既然移动语义是通过右值引用来匹配临时值的，自然会想左值是否也能借助移动语义来优化性能呢？

答案是肯定的，C++11提供了**std::move**来解决这个问题，它将左值转换为右值，从而方便应用移动语义

**std::move**将对象资源的所有权从一个对象转移到另一个对象

举个例子

```C++
std::list< std::string> tokens;
//省略初始化...
std::list< std::string> t = tokens; //这里存在拷贝
//省略代码块标志...
std::list< std::string> tokens;
//省略初始化...
std::list< std::string> t = std::move(tokens);  //这里没有拷贝 
```

使用**std::move**几乎没有任何代价，只是转换了资源的**所有权**，实际将左值变成右值引用，然后应用**移动语义**，调用**移动构造函数**，避免了拷贝

> 事实上，C++11中**所有的**容器都实现了移动语义，以便进行性能优化

这里也要注意对move语义的理解，move实际上不能移动任何东西，唯一的功能是**将一个左值强制转换为一个右值引用**

如果是一些基本类型比如int和char[10]定长数组等类型，使用move仍然会发生拷贝（因为没有对应的移动构造函数）

因此，move对于含资源（堆内存或句柄）的对象来说更有意义



# 第四行代码

```C++
template <typename T>void f(T&& val){ foo(std::forward<T>(val)); }
```

C++11之前调用模板函数时，如何**正确地传递参数**是一个难以解决的问题

```C++
template<typename T>
void forwardValue(T& val) {
	processValue(val);	//右值参数会变成左值
}

template<typename T>
void forwardValue(const T& val) {
	processValue(val);	//参数都变为常量左值引用
}
```

C++11引入了**完美转发**：在函数模板中，完全依照模板的参数的类型（即保持参数的左值、右值特征），将参数传递给模板函数中调用的另一个函数

`std::forward`即**按照参数的实际类型转发**

```C++
#include <iostream>

using namespace std;

void processValue(int& a) { cout << "lvalue" << endl; }
void processValue(int&& a) { cout << "rvalue" << endl; }

template<typename T>
void forwardValue(T&& val) {
	processValue(std::forward<T>(val));	//照参数本来的类型转发
}

void Testdelcl() {
	int i = 0;
	forwardValue(i);	//传入左值
	forwardValue(0);	//传入右值
}

int main(int argc, char**argv) {
	Testdelcl();
	//输出:
	//lvalue
	//rvalue
	return 0;
}

```

右值引用T&&是一个universal references，可以接受左值或者右值，正是这个特性使它适合作为一个**参数的路由**，然后通过`std::forward`按照参数的实际类型去匹配对应的重载函数，最终实现**完美转发**

我们可以结合**完美转发**和**移动语义**来实现一个**泛型的工厂函数**，这个工厂函数可以创建所有类型的对象

```C++
template<typename... Args>
T* Instance(Args... args){
	return new T(std::forward<Args >(args)…);
}
//原文代码，没有看懂，T是什么...
```

这个工厂函数的参数是右值引用类型，内部使用std::forward按照参数实际类型进行转发

如果参数是右值，那么创建时自动匹配移动构造函数；如果是左值则匹配拷贝构造