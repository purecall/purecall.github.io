---
title: C++模板元编程
date: 2018-12-19 10:14:21
tags:
 - C++ 
mathjav: true
---

介绍一些基本技巧，本文是很多博客和书籍的学习笔记，各部分内容可能会有少许重复

其中的代码我虽然已经写了很多遍了，但是让我不看资料......我还真写不出来

权当自己存的编译期代码库吧

越来越感觉自己代码量的不足，这么菜还怎么看chromium里10个G的C++代码

# 元函数与type_traits

## 类型元函数

考虑如下情形：我们希望进行类型映射F，F(int)=unsigned int，F(long)=unsigned long

这种映射可以看成函数，只不过输入和输出都是类型而已

```c++
template<typename T>	//输入(模板参数)
struct Fun_ { using type = T; };

template<>		//特化（映射）
struct Fun_<int> { using type = unsigned int; };

template<>		//特化（映射）
struct Fun_<long> { using type = unsigned long; };

Fun_<int>::type num = 3;	//unsigned int num = 3;

template<typename T>
using Fun = typename Fun_<T>::type;

Fun<int> num1 = 3;
```

> Fun不是一个标准的元函数，因为它没有内嵌类型type。但是它具有输入(T)，输出(Fun<T>)，也明确定义了映射规则，因此可以视作元函数。

事实上，代码中展示的也是C++标准库中定义元函数的常用方式，比如`std::enable_if`和`std::enable_if_t`，前者就像Fun_一样是内嵌了type类型的元函数；后者像Fun一样是基于前者给出的定义，用于简化使用。

### 命名方式

此处约定如下：如果函数返回值需要用某种依赖型的名称表示，那么函数被命名为XXX_的形式（以下划线为后缀）；反之非依赖型则不包含下划线

```C++
template<int a,int b>
struct Add_ {	//依赖型
	constexpr static int value = a + b;
	//value静态变量，依赖于Add_存在
};

template<int a,int b>	//非依赖型
constexpr int Add = a + b;	

constexpr int x1 = Add_<2, 3>::value;
constexpr int x2 = Add<2, 3>;
```



## type_traits

type_traits是由boost库引入的，由C++11纳入其中，头文件<type_traits>，实现了类型变换、类型比较与判断等功能。

```C++
#include<type_traits>

std::remove_reference<int&>::type num1 = 3;
std::remove_reference_t<int&> num2 = 4;
```

更多的一些用于编译期判断的函数

```C++
#include <iostream>
#include <type_traits>

using namespace std;

int main() {
	std::cout << "int: " << std::is_const<int>::value << std::endl;
	std::cout << "const int: " << std::is_const<const int>::value << std::endl;

	//判断类型是否相同
	std::cout << std::is_same<int, int>::value << "\n";// true
	std::cout << std::is_same<int, unsigned int>::value << "\n";// false

	//添加、移除const
	cout << std::is_same<const int, add_const<int>::type>::value << endl;
	cout << std::is_same<int, remove_const<const int>::type>::value << endl;

	//添加引用
	cout << std::is_same<int&, add_lvalue_reference<int>::type>::value << endl;
	cout << std::is_same<int&&, add_rvalue_reference<int>::type>::value << endl;

	//取公共类型
	typedef std::common_type<unsigned char, short, int>::type NumericType;
	cout << std::is_same<int, NumericType>::value << endl;

	return 0;
}
```

还有一个**朽化(std::decay)**操作

>为类型T应用从**左值到右值（lvalue-to-rvalue）**、**数组到指针（array-to-pointer）**和**函数到指针（function-to-pointer）**的隐式转换。转换将移除类型T的**cv限定符**（const和volatile限定符），并定义结果类型为成员decay<T>::type的类型。

```C++
typedef std::decay<int>::type A;           // int
typedef std::decay<int&>::type B;          // int
typedef std::decay<int&&>::type C;         // int
typedef std::decay<constint&>::type D;     // int
typedef std::decay<int[2]>::type E;        // int*
typedef std::decay<int(int)>::type F;      // int(*)(int)
```

朽化还可以将函数类型转换成函数指针类型，从而将函数指针变量保存起来，以便在后面延迟执行

> using FnType = typename std::decay<F>::type;实现函数指针类型的定义

（暂时不太理解，可能与惰性求值有关）









# 模板型模板参数&容器模板

C++元函数可以操作的数据包含3类：**数值**、**类型**与**模板**，统一被称作**元数据**，以示与运行期所操作的数据的区别。



## 模板作为元函数的输入

```C++
#include<type_traits>
template< template<typename>class T1, typename T2 >
struct Fun_ {
	using type = typename T1<T2>::type;
};

template< template<typename>class T1, typename T2 >
using Fun = typename Fun_<T1, T2>::type;

Fun<std::remove_reference, int&> num1 = 3;
Fun_<std::remove_reference, int&>::type num2 = 4;
```

其中元函数Fun接收两个参数：一个模板和一个类型

从函数式编程的角度来说，Fun是一个典型的**高阶函数**，即以另一个函数为输入参数的函数

总结为数学表达式 $Fun(T_1,t_2)=T_1(t_2)$

而模板作为元函数的输出相关内容在实际中使用较少，就不介绍了~~（其实是我不会）~~



## 容器模板

我们需要的是一个容器：用来保存数组中的每个元素，元素可以是数值、类型或模板。典型的容器仅能保存相同类型的数据，但已经可以满足绝大多数的使用需求。

C++11中引入了**变长参数模板(variadic template)**，借由此实现我们的容器

```C++
template<int... Vals> struct IntContainer;
template<typename... Types> struct TypeContainer;

template<template<typename>class... T>struct TemplateCont;
//可以存放模板作为其元素，每个模板元素可以接收一个类型作为参数

template<template<typename...>class... T>struct TemplateCont2;
//同样以模板作为其元素，但每个模板可以放置多个类型信息
```

这些语句实际上只是声明而非定义，事实上这是一个元编程中的一个惯用法：仅在必要时才引入定义，其他的时候直接使用声明即可。

关于变长参数模板后文还会提到

# 顺序、分支与循环

## 顺序执行

```Cpp
#include<type_traits>
template<typename T>
struct RemoveReferenceConst_ {
private:	//private确保函数的使用者不会误用中间结果inter_type作为返回值
	using inter_type = typename std::remove_reference<T>::type;
public:		
	using type = typename std::remove_const<inter_type>::type;
};

template<typename T>
using RemoveReferenceConst = typename RemoveReferenceConst_<T>::type;

RemoveReferenceConst<const int&> num = 1;

```

代码中先根据T计算出inter_type，再用这个中间结果计算出type

结构体中的所有声明都要看成执行的语句，不能更换顺序

**这里改变顺序确实会报错，类似地，在下文 分支选择与短路逻辑 中有一些想法**

> 在编译期，编译器会扫描两遍结构体中的代码，第一遍处理声明，第二遍才会深入到函数定义中。

如果先扫描到type，发现它依赖于未定义的inter_type，就不会继续扫描而是报错。



## 分支执行

### std::conditional(_t)

conditional和conditional_t为type_traits中提供的2个元函数，定义如下：

```C++
namespace std {
	template<bool B, typename T, typename F>
	struct conditional {
		using type = T;
	};

	template<typename T, typename F>
	struct conditional<false, T, F> {
		using type = F;
	};

	template<bool B, typename T, typename F>
	using conditional_t = typename conditional<B, T, F>::type;
}
```

> 注意这里只**偏特化**了false的情况，但是却可以完美的表达true的情形。
>
> 我个人猜测与编译器最佳匹配的实现有关，后面也会提到**SFINAE**

逻辑行为：如果B为真，则函数返回T，否则返回F

```C++
//测试代码
int main(int argc, char**argv) {
	std::conditional<true, char, int>::type x = 3;
	std::cout << sizeof(x) << std::endl;	// 1 char
	std::conditional_t<false, char, int> y = 4;
	std::cout << sizeof(y) << std::endl;	// 4 int
	return 0;
}
```

### 使用特化实现分支

特化天生就是用于引入差异的，因此可以用于实现分支

```C++
struct A; struct B;

template<typename T>
struct Fun_{
	constexpr static size_t value = 0;
};

template<>
struct Fun_<A> {
	constexpr static size_t value = 1;
};

template<>
struct Fun_<B> {
	constexpr static size_t value = 2;
};

constexpr size_t v = Fun_<B>::value;	//依赖型 2
// Fun_<int>::value == 0; <typename> 传入int
```

Fun_元函数实际上引入了3个分支，分别对应输入参数为A、B与默认情况

> 这里A和B只是用于特化的类型，因此只需要声明，不需要定义
>
> 可能与下一章异类词典中用类名作为键类似

C++14中另一种特化方式：

```C++
#include<iostream>

struct A; struct B;

//非依赖
template<typename T>
constexpr size_t Fun = 0; 

template<>
constexpr size_t Fun<A> = 1;

template<>
constexpr size_t Fun<B> = 2;

constexpr size_t h = Fun<int>;

int main(int argc, char**argv) {
	std::cout << Fun<int><<std::endl;	// 0
	std::cout << Fun<A> << std::endl;	// 1
	return 0;
}
```



这段代码里特化的2处模板，vs报constexpr在此处无效，不清楚为什么

~~必须在其首次使用之前对 变量 "Fun [其中 T=A]" 进行显式专用化()~~



书上提醒：在**非完全特化**的类模板中引入**完全特化**的分支代码是**非法**的

注释代码g++编译失败，vs成功

```C++
#include<iostream>

template<typename TW>
struct Wrapper {	//包装
	/*
	template<typename T>
	struct Fun_ {
		constexpr static size_t value = 0;
	};

	template<>
	struct Fun_<int> {
		constexpr static size_t value = 1;
	};
	// vs中运行正常
	// g++报错:
	//explicit specialization in non-namespace scope
	//template parameters not deducible in partial specialization
	*/

	template<typename T, typename TDummy = void>
	struct Fun_ {
		constexpr static size_t value = 0;
	};

	template<typename TDummy> // Dummy 挂名，傀儡
	struct Fun_<int, TDummy> {
		constexpr static size_t value = 1;
	};
};

int main(int argc, char**argv) {
	std::cout << Wrapper<int>::Fun_<int>::value;
	return 0;
}
```

改进后的代码引入了一个伪参数**TDummy**，用于将原有的**完全特化**转换为**部分特化**

但是设定了默认值void，因此可以直接以`Fun_<int>`调用这个元函数，无需赋值



### 使用std::enable_if(_t)

```C++
//代码原型
namespace std {
	template<bool B, typename T = void>
	struct enable_if {};

	template<class T>	//这里当然不会直接传true...可以其他计算结果
	struct enable_if<true, T> { using type = T; };

	template<bool B, class T = void>
	using enable_if_t = typename enable_if<B, T>::type;
}
```

这里T不是特别重要，重要的是当B为true时，enable_if元函数可以返回结果type，可以基于此构造实现分支

代码实例

```C++
template<bool IsFeedbackOut, typename T,
	std::enable_if_t<IsFeedbackOut>* = nullptr>	//这里多了  * = nullptr，暂时没搞懂作用，
auto FeedbackOut_(T&&) { /*...*/ }

template<bool IsFeedbackOut, typename T,
	std::enable_if_t<IsFeedbackOut>* = nullptr> //可能是习惯用法吧，可作为伪参数？
auto FeedbackOut_(T&&) { /*...*/ }
```

这里引入分支，根据IsFeedback的值来匹配模板

#### SFINAE

SFINAE(Substitution Failure Is Not An Error)，译为"匹配失败并非错误"

当匹配模板时，编译器即使已经匹配到了一个足够好的选择，也会把所有选择都尝试匹配，最后选择最佳的

这里std::enable_if(_t)正是利用了这一点

> 有些情况下，我们希望引入**重名函数**，它们无法通过参数类型加以区分，这时enable_if(_t)能在一定程度上解决相应的重载问题

#### 补充

std::enable_if(_t)也有一些缺点，并不像模板特化那么直观，可读性较差

这里给出的代码实例是一个典型的**编译期与运行期结合**的使用方式，FeedbackOut\_中包含了运行期的逻辑，而选择哪个FeedbackOut\_则是通过编译期的分支来实现



### 计算实例

利用二分法计算整数的平方根，结果向下取整

```C++
#include<iostream>

template <bool Flag, class MaybeA, class MaybeB> class IfElse;

template <class MaybeA, class MaybeB>
class IfElse<true, MaybeA, MaybeB> {
public:
	using ResultType = MaybeA;
};

template <class MaybeA, class MaybeB>
class IfElse<false, MaybeA, MaybeB> {
public:
	using ResultType = MaybeB;
};

template <int N, int L, int R> struct __Sqrt {
	enum { mid = (L + R + 1) / 2 };

	using ResultType = typename IfElse<(N < mid * mid),
		__Sqrt<N, L, mid - 1>, __Sqrt<N, mid, R> >::ResultType;

	enum { result = ResultType::result };
};

template <int N, int L> struct __Sqrt<N, L, L> { enum { result = L }; };

template <int N> struct _Sqrt { enum { result = __Sqrt<N, 1, N>::result }; };


int main(int argc, char**argv) {
	std::cout << _Sqrt<10>::result;	// 3
	return 0;
}
```



### 编译期分支与多返回类型

```C++
#include<iostream>

/*
运行期函数返回地址在编译时必须确定，因此会报错
auto wrap1(bool Check) {
	if (Check)return (int)0;
	else return (double)0;
}
*/

//但在编译期，可以一定程度上打破这种限制
template<bool Check, std::enable_if_t<Check>* = nullptr>
auto fun() {
	return (int)0;
}

template<bool Check, std::enable_if_t<!Check>* = nullptr>
auto fun() {
	return (double)0;
}

template<bool Check>
auto wrap2() {
	return fun<Check>();
}

int main(int argc, char**argv) {
	std::cerr << wrap2<true>() << std::endl;
	return 0;
}

```

这是一种典型的编译期分支和运行期函数结合的例子，C++17引入了`if constexpr`来简化代码

```C++
#include<iostream>

template<bool Check>
auto fun() {
	if constexpr (Check) {
		return (int)0;
	}
	else {
		return (double)0;
	}
}

int main(int argc, char**argv) {
	std::cerr << fun<true>() << std::endl;
	return 0;
}
```

其中`if constexpr`必须接收一个常量表达式

## 循环执行

为了更有效地操纵元数据，往往选择递归的形式来实现循环

举个例子：给定一个无符号数，求该整数所对应的二进制表示中1的个数

```C++
#include<iostream>

template<size_t Input>
constexpr size_t OnesCount = (Input % 2) + OnesCount< (Input / 2) >;

template<> constexpr size_t OnesCount<0> = 0;
//这里vs又报错了...好像和上次一样
//E2386:"constexpr"在此处无效
//E1449:必须在其首次使用之前对变量"OnesCount[其中Input=0Ui64]"进行显式专用化
//虽然报错但还是能运行...之前也是

constexpr size_t res = OnesCount<45>;
```

循环使用更多的一类情况则是处理**数组**，以下给出一个实例

```C++
template<size_t...Inputs>
constexpr size_t Accumulate = 0;

template<size_t CurInput,size_t...Inputs>
constexpr size_t Accumulate<CurInput, Inputs...>
	= CurInput + Accumulate<Inputs>;

constexpr size_t res = Accumulate<1, 2, 3, 4, 5>;
```

当输入数组为空时，会匹配第一个模板特化，返回0；如果有正数个参数，则匹配第二个模板特化

以下使用C++17中的**折叠表达式(fold expression)**简化循环

```C++
template<size_t... values>
constexpr size_t fun() {
	return (0 + ... + values);
}

const size_t res = fun<1, 2, 3, 4, 5>();
```

## 小心实例化爆炸与编译崩溃

编译时实例化的模板会被保存起来用以可能的复用，对于一般的C++程序，可以极大地提升编译速度；但是对模板元编程来说，很可能造成灾难，考虑以下代码：

计算   $\sum_{i=1}^{A+ID}i$

```C++
#include<iostream>

template<size_t A>
struct Wrap_ {
	template<size_t ID, typename TDummy = void>
	struct imp {
		constexpr static size_t value = ID + imp<ID - 1>::value;
	};

	template<typename TDummy>
	struct imp<0, TDummy> {	//伪参数
		constexpr static size_t value = 0;
	};

	template<size_t ID>
	constexpr static size_t value = imp<A + ID>::value;
};

int main(int argc, char**argv) {
	std::cerr << Wrap_<3>::value<2> << std::endl;
	//产生Wrap_<3>::imp的一系列实例
	std::cerr << Wrap_<10>::value<2> << std::endl;
	//产生Wrap_<10>::imp的一系列实例
	//这两个系列不同名，不能复用
	return 0;
}
```

循环所产生的全部实例都会在编译器中保存，如果有大量实例，很可能会内存超限甚至崩溃

解决方案：把循环拆分出来以复用

```C++
#include<iostream>

template<size_t ID>
struct imp {
	constexpr static size_t value = ID + imp<ID - 1>::value;
};

template<>
struct imp<0> {
	constexpr static size_t value = 0;
};

template<size_t A>
struct Wrap_ {
	template<size_t ID>
	constexpr static size_t value = imp<A + ID>::value;
};

int main(int argc, char**agrv) {
	std::cerr << Wrap_<3>::value<2> << std::endl;	// == imp<3+2>
	std::cerr << Wrap_<10>::value<2> << std::endl;	// == imp<10+2>
	//思考实例化的过程，确实是可复用的
	return 0;
}
```

但是也有一些不足之处：之前的代码imp被置于Wrap\_中，体现二者的紧密联系，从**名称污染**的角度来说，这样做不会让imp污染Wrap\_外围的名字空间

后一种实现中，imp将对名字空间造成污染：在相同的名字空间中，我们无法再引入一个名为imp的构造，以供其他元函数使用

**权衡**：如果元函数的逻辑比较简单，同时不会产生大量实例，那么保留前一种（对编译器来说比较糟糕的形式）；反之，如果元函数逻辑比较复杂（典型情况是多重循环嵌套），又可能产生很多实例，就选择后一种以节省编译资源。

当然，选择后一种方式时，我们可以引入专用的名字空间来尽力避免名称污染



## 分支选择与短路逻辑

计算实例：计算1~N是否全为奇数

```C++
#include<iostream>

template<size_t N>
constexpr bool is_odd = ((N % 2) == 1);

template<size_t N>
struct AllOdd_ {
	constexpr static bool is_cur_odd = is_odd<N>;
	constexpr static bool is_pre_odd = AllOdd_<N - 1>::value;
	constexpr static bool value = is_cur_odd && is_pre_odd;
	//突然想问，既然前面提到了顺序执行，依照递归展开的逻辑，计算is_pre_odd需要依赖value
	//而这时value还没出现...
	//我猜想可能是编译器智能地预测了展开形式，直接匹配到了AllOdd_<0>，找到了退出时的value
	//因此栈回溯时也就找到了value，大概是这样智能的结束的递归吧...
};

template<>
struct AllOdd_<0> {
	constexpr static bool value = is_odd<0>;
};

int main(int argc, char**argv) {
	std::cout << AllOdd_<10>::value;
	getchar();
	return 0;
}
```

但这种逻辑短路的行为在上述程序中并没有得到利用——无论is_cur_odd是什么，AllOdd_都会对is_pre_odd进行求值，改进如下：

```C++
#include<iostream>

template<size_t N>
constexpr bool is_odd = ((N % 2) == 1);


template<bool cur, typename TNext>
constexpr static bool AndValue = false;	

template<typename TNext>	//模板作为模板参数
constexpr static bool AndValue<true, TNext> = TNext::value;
//只特化了true的情况，一旦cur为false，最佳匹配是上一个模板，直接返回false，停止展开
//和std::enable_if一样是 SFINAE


template<size_t N>
struct AllOdd_ {
	constexpr static bool is_cur_odd = is_odd<N>;
	constexpr static bool value = AndValue<is_cur_odd, AllOdd_<N - 1>>;
	//上一个版本的代码，相当于在AndValue第一个参数为false时依然展开AllOdd_<N-1>
};

template<>
struct AllOdd_<0> {
	constexpr static bool value = is_odd<0>;
};


int main(int argc, char**argv) {
	std::cout << AllOdd_<10>::value;
	return 0;
}
```

# 奇特的递归模板式

奇特的递归模板式(Cruiously Recurring Template Pattern,CRTP)是一种派生类的声明方式

奇特之处在于：派生类会将其**本身**作为模板参数传递给其基类

```C++
template<typename D>class Base{ /*...*/ };
//基类以派生类的名字作为模板参数...
class Derived:public Base<Derived>{ /*...*/ };
```

似乎看上去有循环定义的嫌疑，但它确实是合法的...

CRTP的应用场景之一：模拟虚函数

```C++
#include<iostream>

using namespace std;

template<typename D>
struct Base {
	static void Fun() {
		D::Imp();
	}
};

struct Derive : public Base<Derive> {
	static void Imp() {
		cout << "Implementation from derive class" << endl;
	}
};

int main(int argc, char**argv) {
	Derive::Fun();
	return 0;
}
```

整个函数调用过程不难理解



# 模板中的变长参数

## 可变模板参数函数

### 基本形式

```C++
template<typename... Args>	//模板参数包，包含函数调用中的参数匹配的类型，比如char*,int
void show(Args... args) {	//args函数参数包，指出args的类型为Args
	//函数调用时，根据args反推Args类型（auto）
}
```

### 获取参数包信息

#### 计算参数个数

一个实例

```C++
#include<iostream>

using namespace std;

template<class... T>
void f(T... args) {
	cout << sizeof...(args) << endl;	//打印变参的个数
}

int main(int argc, char**argv) {
	f();				// 0
	f(1, 2);			// 2
	f(1, 2.4, "hello");	 // 3

	return 0;
}
```

#### 获得每个参数

##### 递归

递归方法需要一个终止函数

```C++
#include<iostream>

using namespace std;

//递归终止函数
void print() {
	cout << "empty" << endl;
}

//展开函数
template<class T,class... Args>
void print(T head, Args... rest) {
	cout << "parameter " << head << endl;
	print(rest...);
}

int main(int argc, char**argv) {
	print(1, 2, 3, 4);
	// parameter 1
	// parameter 2
	// parameter 3
	// parameter 4
	// empty

	return 0;
}
```

求和

```C++
#include<iostream>

using namespace std;

template<typename T>
T sum(T t) {
	return t;
}

template<typename T,typename... Types>
T sum(T first, Types... rest) {
	return first + sum<T>(rest...);
}

int main(int argc, char**argv) {
	cout << sum(1, 2, 3, 4, 5)<<endl;	// 15
	return 0;
}
```

##### 逗号表达式

实例如下

```C++
#include<iostream>

using namespace std;

template<class T>
void printarg(T t) {
	cout << t << "\t";
}

template<class... Args>
void expand(Args... args) {
	int arr[] = { (printarg(args),0)... };
}

int main(int argc, char**argv) {
	expand("hello",123, 'x');
	// hello   123     x
	return 0;
}
```



C/C++中的表达式会按顺序执行，同时这里用到了C++11的特性——**初始化列表**

`(printarg(args),0)`会先执行函数，再得到逗号表达式的结果0

通过初始化列表来初始化一个变长数组

`{(printarg(args),0)...}`将会展开成`((printarg(arg1),0),(printarg(arg2),0),etc...)`

最终会创建一个元素值都为0的数组int arr[sizeof...(Args)]

> 这里只用于展开参数包，我们可以将函数作为参数，就可以支持lambda表达式了





## 可变模板参数类

比较基本的就是这个tuple了...

```C++
std::tuple<int> tp1 = std::make_tuple(1);
std::tuple<int, double> tp2 = std::make_tuple(1, 2.5);
std::tuple<int, double, string> tp3 = std::make_tuple(1, 2.5, "");
```

由于可变参数模板的模板参数个数（绕口令?...）可以为0，所以以下定义也是合法的

`std::tuple<> tp;`

### 展开参数包的方法

#### 模板偏特化和递归

（感觉和刚刚的求和没什么区别，换些实例）

##### 实例1：IntegerMax

```C++
#include <iostream>
#include <type_traits>

using namespace std;

//获取最大的整数
template<size_t arg, size_t... rest>
struct IntegerMax;

template<size_t arg>
struct IntegerMax<arg> :std::integral_constant<size_t, arg> {};

template<size_t arg1, size_t arg2, size_t... rest>
struct IntegerMax<arg1, arg2, rest...> :	//继承之后这个类也有value了...
	std::integral_constant<size_t, arg1 >= arg2 ?
	IntegerMax<arg1, rest...>::value :
	IntegerMax<arg2, rest...>::value > {
};

int main(int argc,char**argv) {
	cout << IntegerMax<2, 5, 1, 7, 3>::value << endl;	// 7
	return 0;
}
```

这段代码看起来比较复杂......实际上是一个继承，`integral_constant<size_t,num>`应该就是`size_t num`



##### 实例2：MaxAlign

上一段代码改一下可以轻松实现获取**最大内存对齐值的元函数MaxAlign**

增加以下部分

```C++
template<typename... Args>
struct MaxAlign : std::integral_constant<int,
	IntegerMax<std::alignment_of<Args>::value...>::value > {};


int main() {
	cout << MaxAlign<int, short, double, char>::value << endl;	//  => 8
	return 0;
}
```



##### 实例3：TypeSizeSum

我自己写的也不太熟练，还是再来一个求和吧，计算参数包中参数类型的size之和

~~突然想用LaTeX写Σ了...~~

```C++
#include <iostream>
#include <type_traits>

using namespace std;

//前向声明
template<typename... Args>
struct Sum;

//基本定义
template<typename First, typename... Rest>
struct Sum<First, Rest...>{
	enum { value = Sum<First>::value + Sum<Rest...>::value };
};

//递归终止
template<typename Last>
struct Sum<Last>{
	enum { value = sizeof(Last) };
};


int main(int argc,char**argv) {
	cout<<Sum<int, double, short>::value;	//值为14
	return 0;
}
```



#### 继承方式

MakeIndexes的作用是生成一个可变参数模板类的整数序列，最终输出的类型是：struct IndexSeq<0,1,2>

（看懂代码已属不易，这个类型的实际使用我还没有考虑）

```C++
#include <iostream>
#include <type_traits>

using namespace std;

//整形序列的定义
template<int...>
struct IndexSeq{};

//继承方式展开参数包
template<int N,int... Indexes>
struct MakeIndexes:MakeIndexes<N-1,N-1,Indexes...>{};

//说明（往前展开）
//MakeIndexes<3>: MakeIndexes<2, 2>{}
//MakeIndexes<2, 2>: MakeIndexes<1, 1, 2>{}
//MakeIndexes<1, 1, 2>: MakeIndexes<0, 0, 1, 2>
//{
//	typedef IndexSeq<0, 1, 2> type;
//}

//模板特化，终止展开参数包条件
template<int... Indexes>
struct MakeIndexes<0, Indexes...> {
	typedef IndexSeq<Indexes...> type;
};
//这里去掉了多出来的一个0


int main(int argc,char**argv) {
	using T = MakeIndexes<3>::type;
	cout << typeid(T).name() << endl;
	// struct IndexSeq<0,1,2>
	return 0;
}
```



