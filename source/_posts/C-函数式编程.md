---
title: C++函数式编程
date: 2018-12-24 13:40:26
tags: C++
---



由于我自己也是在学习的过程中，并没有形成具体的函数式思维，因此可能记录的学习过程比较乱

# 标准库的基本使用

## std::pair

>  **std::pair**是标准库中所有**键值数据(key-value data)**的默认返回类型，是快速使用标准库的第一把钥匙

```C++
#include<iostream>
#include<string>
#include<utility>
#include<functional>

using namespace std;

template<class T1,class T2>
ostream& operator<<(ostream& out, const pair<T1, T2>& _) {
	return out << "(" << _.first << "," << _.second << ")" << endl;
}

int main(int argc, char**argv) {
	std::string names[3] = { "lambda","C++","pwn" };
	int scores[3] = { 3,4,5 };
	std::pair <std::string, int> p0 = std::make_pair(names[1], scores[1]);
	//p0 = (C++,4)
	auto p1 = std::make_pair(names[2], scores[2]);
	//p1 = (pwn,5)
	auto p2 = std::make_pair(names[1], std::ref(scores[1]));
	scores[1] += 10;
	//p0不变
	//p2 = (C++,14)     std::ref()
	cout << p0 << p1 << p2;

	return 0;
}
```

## std::tuple

更一般地，**元组(tuple)**可以把多个不同数据类型的实例绑定到一个实例上

```C++
#include<iostream>
#include<string>
#include<tuple>
#include<functional>

using namespace std;

int main(int argc, char**argv) {
	std::string names[3] = { "C++","lambda","reverse" };
	char ranks[3] = { 'A','B','C' };
	int scores[3] = { 5,6,7 };

	auto student0 = std::make_tuple(names[0], ranks[0], scores[0]);
	std::cout << "student0  name:" << std::get<0>(student0)
		<< ", rank:" << std::get<1>(student0)
		<< ", score:" << std::get<2>(student0) << std::endl;
	//student0  name:C++, rank:A, score:5

	std::string s;
	char c;
	int i;
	auto student1 = std::make_tuple(names[1], ranks[1], scores[1]);
	std::tie(s, c, i) = student0;
	std::cout << "student1  name:" << s
		<< ", rank:" << c
		<< ", score:" << i << std::endl;
	//student1  name:C++, rank:A, score:5

	/*
	auto student2 = std::tie(names[2],ranks[2],scores[2]);
	auto[_1, _2, _3] = student2;	// C++17 structured binding
	std::cout << "student2  name:" << _1
		<< ", rank:" << _2
		<< ", score:" << _3 << std::endl;

	g++-7 main.cpp -std=c++17
	输出：student2  name:reverse, rank:C, score:7
	*/

	return 0;
}
```

**tuple**的组装和拆包任务分别由两个函数来担当

`std::make_tuple`创建一个元组，`std::get`获取确定位置的数据

注意到，`std::tie`既可以组装元组，也可以对一个元组进行拆包

```cpp
std::tie(std::ignore, std::ignore, xxx) = student0;
```

`std::ignore` 可以作为解包时的占位符

> auto[a,b,c]=xxx 是复制一份数据
>
> std::tie()默认会创建原变量的引用



## std::vector

基本的使用就不提了，后文会经常用到，考虑一下vector扩充内存的细节

当反复push_back()一个元素时，一旦达到了上限，vector会把可用空间大小扩充为之前的k倍，一般为2或1.5，具体由编译器决定





# lambda函数

## 基本使用

举个两个简单的例子

```C++
#include<iostream>
#include<vector>

using namespace std;

int main(int argc, char**argv) {
	auto addition = [](int _1, int _2)->int { return _1 + _2; };
	vector<int>a = { 1,2,3,4,5 };
	vector<int>b = { 6,7,8,9,10 };
	for (int i = 0, sz = a.size(); i < sz; i++) {
		a[i] = addition(a[i], b[i]);
	}
	for (auto e : a)cout << e << endl;

	return 0;
}
```

要注意的是，addition并不是lambda函数的名字，它只是被附身了而已...lambda本身是没有名字的，因此不能简单地实现递归

参数列表中的中括号，名为**捕捉列表**，用于捕捉当前lambda所在环境的变量

参数列表之后有一个箭头，箭头后跟着返回值的类型

最后，与函数声明完之后直接用大括号收尾不同，在这里可以暂时把lambda函数看做一个特殊的变量，所以最后用分号收尾

`auto a_alias = [capture](parameters)->return_type{...};`



```C++
#include<iostream>
#include<vector>

using namespace std;

int main(int argc, char**argv) {
	int j = 1;
	auto by_val_lambda = [=] { return j + 1; };
	auto by_ref_lambda = [&] { return j + 1; };
	auto print = [=] {
		cout << "print by val lambda: " << by_val_lambda() << endl;
		cout << "print by ref lambda: " << by_ref_lambda() << endl;
	};
	print();
	j += 10;
	print();

	return 0;
}
```

结果如下

```c++
print by val lambda: 2
print by ref lambda: 2
print by val lambda: 2
print by ref lambda: 12		//引用传递
```

[=]表示按照**值传递**的方法捕捉父作用域的所有变量

[&]表示按照**引用传递**的方法捕捉父作用域的所有变量

如果想要捕捉特定的变量，可以直接写[var]或[&var]

甚至可以捕捉this指针，如果父作用域存在的话

> lambda函数的出现减少了对命名空间的污染，也可以理解为增加了一种简单的局部函数的实现
>
> 可以直接把函数写在main函数里，用函数指针来管理



## lambda递归

计算阶乘

```C++
#include<iostream>
#include<functional>
#include<vector>

using namespace std;

int main(int argc, char**argv) {
	std::function<int(int)>fact;
	fact = [&fact](int x)->int { return x <= 1 ? x : x * fact(x - 1); };
	cout << fact(5) << endl;
	return 0;
}
```

lambda实现递归时，可以提前声明fact，然后再用[&fact]捕获

当右值赋给fact时，也就是fact函数指针被lambda”附身”，fact的值改变，那么fact的引用也对应改变



以下为照抄部分：[链接](https://zhuanlan.zhihu.com/p/45430715)

准确地讲，上面**根本不是**lambda函数的递归调用，只是一般函数的递归而已

想要实现lambda函数的递归调用，必须首先对**Y-组合子**有一定的了解

简单地说，虽然lambda没有名字，但我们可以把它作为一个参数传递给更上层的函数调用

熟悉函数式编程的话会想起**不动点组合子**，也就是构造一个函数，返回它的不动点

```haskell
fix :: (a -> a) -> a
fix f = let {x = f x} in x
```

抄不动了，我也不熟悉函数式啊.....但是直接用C++还是会写的...





# 函数式编程工具箱



## 把函数看作一等公民

传统的 C++ 把**承载数据的变量**作为一等公民，这些变量享有以下权利：

1、可以承载数据，也可以存在于更高级的数据结构中；

2、可以作为右值参与计算、函数调用；

3、可以作为左值参与赋值操作。

那么，如果把**函数**也当做一等公民，那么，函数也应当享有以下的权利：

1、函数就是数据，一个函数变量可以承载函数，也可以存在于更高级的数据结构中；

2、可以作为右值参与计算，或参与更高级的函数调用；

3、可以作为左值参与赋值操作。



这里给一个简单计算器的例子来理解

```C++
#include <iostream>
#include <functional>
#include <cmath>
#include <map>

using namespace std;

int main(int argc, char**argv) {
	std::map<char, std::function<double(double, double)> > host;
	host['+'] = [](double a, double b)->double { return a + b; };
	host['-'] = [](double a, double b)->double { return a - b; };
	host['*'] = [](double a, double b)->double { return a * b; };
	host['/'] = [](double a, double b)->double { return a / b; };
	host['^'] = [](double a, double b)->double { return pow(a, b); };
	cout << "1.1 + 2.2= " << host['+'](1.1, 2.2) << endl; // => 3.3
	cout << "1.1 - 2.2= " << host['-'](1.1, 2.2) << endl; // => -1.1
	cout << "1.1 * 2.2= " << host['*'](1.1, 2.2) << endl; // => 2.42
	cout << "1.1 / 2.2= " << host['/'](1.1, 2.2) << endl; // => 0.5
	cout << "1.1 ^ 2.2= " << host['^'](1.1, 2.2) << endl; // => 1.23329

	return 0;
}
```

这里我们使用了标准库提供的`std::map`来存储核心匹配关系，即一个运算符对应一个运算关系

这时map存储的数据不再是数字，而是一个**具体的函数**，一种**具体的运算方法**，这就是刚刚提到的公民权利的第一条和第三条('+'作为一个函数载体，作为**左值**)

 而第二条：作为**右值**参与计算或参与更高级的函数调用，根据以下的三个工具来理解

## 基本工具



**map**，**filter**，**fold**分别表示对集合而言的三种操作：映射、筛选、折叠

### 映射

映射操作：对一个列表`vec`内的所有元素依照某种运算法则`f`进行运算，从而得到一个全新的列表`[f(x) for x in vec]`，这确实是一个基于列表的循环操作

> 提到**函数式编程**时，遵循两个原则：
>
> 1、DRY(Don't Repeat Yourself)，一个组件只有一次实现，绝不特化过多的实现
>
> 2、重视逻辑推演而非执行过程，侧重如何通过逻辑上的组合、推演，来完备的定义问题的解，执行过程交给计算机即可



STL中提供了两个工具：`std::for_each`和`std::transform`，后者支持更好甚至支持两个列表同时计算

>  `for_each`写法比for+auto复杂，关键是还保证了顺序执行，相比for优势全无，不过这一点也可以使它应用于一个非纯函数的工具

> **纯函数**是指没有副作用、没有后继状态变化，对相同输入总会反馈相同结果的一类特殊函数
>
> 某种意义上说，函数式编程(尽量)要求使用纯函数从而方便测试(反馈唯一)、维护(没有副作用)、自动并行化(没有后继的状态变化)

```C++
#include<iostream>
#include<vector>
#include<algorithm>

using namespace std;

int main(int argc, char**argv) {
	std::vector<int> a = { 1,2,3,4 };
	int sum = 0, idx = 1, sum2 = 0;
	auto add_to_sum = [&sum](int _) {sum += _; };
	auto add_to_sum_with_idx = [&sum2, &idx](int _) {sum2 += (_ * idx++); };
	
	for (auto e : a) cout << " " << e;
	cout << endl;	
	for_each(a.begin(), a.end(), add_to_sum);
	cout << sum << endl;	// 1+2+3+4
	for_each(a.begin(), a.end(), add_to_sum_with_idx);
	cout << sum2 << endl;	// 1 * 1 + 2 * 2 + 3 * 3 + 4 * 4
	//可以考虑差比数列求和 AP * GP，这里只是个简单的非纯函数例子

	return 0;
}
```

回到`transform`，它不保证**按序执行**，它要求函数不改变迭代器（比如delete）或者修改范围内部元素（比如把下一个元素的值+1）

同时`transform`要求有返回值，每执行一次都会把结果存储在一个新的列表中

`transform`允许同时接受两个列表，但是此时要求后者的长度**必须大于等于**前者的长度，否则会产生未定义行为(Undefined Behavior)

实际上`transform`也可以实现就地操作，只要把输出的迭代器位置变换成自己的`begin()`就可以了

举个简单的例子

```C++
#include<iostream>
#include<vector>
#include<algorithm>

using namespace std;

int main(int argc, char**argv) {
	std::vector<int>a = { 1,2,3,4,5,6 };
	std::vector<int>b;
	std::transform(a.begin(), a.end(), std::back_inserter(b),
		[](int _)->int { return _ + 3; });
	//遍历a，每个数据+3后插入到vector b中

		return 0;
}
```



### 筛选

映射是所有操作的根基，因为所有操作都可以用映射进行解释，筛选和折叠也不例外

**筛选**：对一个列表内的所有元素应用一个**谓词(predicate，谓词特指判断事物返回True或False的函数)**，然后取出所有结果为True的元素，简单地说就是for循环 + if-else判断

STL提供了`copy_if`和`remove_if`来实现筛选，给出一个实例

```C++
#include<iostream>
#include<vector>
#include<algorithm>

using namespace std;

int main(int argc, char**argv) {
	auto print_value = [](auto v) {for (auto e : v)cout << " " << int(e); cout << endl; };
#define DISPLAY_VALUE(v)do{cout<<#v":";print_value(v);}while(0);
	std::vector<int> a = { 1,2,3,4,5,6,7,8,9,0 };
	std::vector<int> b;
	std::copy_if(a.begin(), a.end(), std::back_inserter(b),
		[](int _) {return _ % 2; });
	DISPLAY_VALUE(a);
	DISPLAY_VALUE(b);

	a.erase(std::remove_if(a.begin(), a.end(), [](int _) { return _ % 2; }), a.end());
	DISPLAY_VALUE(a);
#undef DISPLAY_VALUE
	// 输出结果
	// a: 1 2 3 4 5 6 7 8 9 0
	// b: 1 3 5 7 9
	// a: 2 4 6 8 0
	
	return 0;
}
```

需要注意的是，`copy_if`需要额外开一块空间出来，`remove_if`则返回了需要删掉的元素的迭代器，然后再通过`erase`函数彻底删除元素

与`transform`类似，这里的谓词函数同样要求不得修改原始数据



### 折叠

**折叠**：对一个列表和一个初始值应用一个二元函数，然后依次按某一特定方向将整个列表折叠起来

~~看完代码就能理解这个描述了~~

STL中提供了`accumulate`和`inner_product`，两者都是**左折叠函数**，分别是执行求和、计算向量内积的函数

后者实际上是先对两个列表应用一个二元运算（默认乘法），把结果存入一个列表中，再对这个列表折叠（默认求和）

```C++
#include<iostream>
#include<vector>
#include<algorithm>
#include<numeric>

using namespace std;

int main(int argc, char**argv) {
	auto print_value = [](auto v) {for (auto e : v)cout << " " << int(e); cout << endl; };
#define DISPLAY_VALUE(v)do{cout<<#v":";print_value(v);}while(0);
	std::vector<int> a = { 1,2,3,4,5,6,7,8,9,10 };
	int sum = std::accumulate(a.begin(), a.end(), 0,
		[](int acc, int _) { return acc + _; });
	cout << sum << ", ";
	DISPLAY_VALUE(a);
	// 55, a: 1 2 3 4 5 6 7 8 9 10

	int prod = std::accumulate(a.begin(), a.end(), 1,
		[](int acc, int _) { return acc * _; });
	cout << prod << ", ";
	DISPLAY_VALUE(a);
	// 3628800, a: 1 2 3 4 5 6 7 8 9 10

	//反转vector
	std::vector<int> a_re = std::accumulate(a.begin(), a.end(), std::vector<int>(),
		[](std::vector<int> acc, int _)->std::vector<int> {
		std::vector<int> __; __.push_back(_);
		std::copy(acc.begin(), acc.end(), std::back_inserter(__));
		return __;
	});
	DISPLAY_VALUE(a_re);
	// a_re: 10 9 8 7 6 5 4 3 2 1

#undef DISPLAY_VALUE

	return 0;
}
```

简单地说，应用的二元函数的第一个参数是累加值，第二个参数是列表中取出的值，返回值的类型应与初值和累加值的类型相同（熟练以后可以用auto代替）

最后一个反转vector略有一点难度，如果看不懂可以输出中间结果看看



> 考虑一个问题，由于C++依旧是基于**面向过程**和**面向对象**的语言，所以在函数式编程上的优化可能没有那么专业，而且C++会按语句依次执行，如果大量堆叠`transform`会不会造成性能下降呢？
>
> 这就是之后将要引入的内容：是否引入简单的**lazy**求值，以及简单的柯里化





> 题外话：这里我突然想到了一些东西，当看到这句话`期末考完可能得等`，它是很不符合语法的语句，但是我们可以轻易得知语句表达的含义。我猜测，眼睛读取信息时并行的读入这8个汉字，然后由大脑根据词组进行全排列，比如这8个字分成了四个词组，然后匹配最佳语义（又能扯到SFINAE了吗...）



## 函数的部分应用

在处理**柯里化**之前，我们先思考这个问题：如何对一个函数进行**部分应用**

举个例子，比如最常见的除法`a/b`，能否只传递其中的一个参数给这个除法运算，从而得到一个接收另一个参数的函数？~~话很绕口，但是代码不绕口~~

```C++
#include<iostream>

using namespace std;

int main(int argc, char**argv) {
	auto div = [](int a, int b)->int { return a / b; };
	auto div3 = [div](int a)->int {return div(a, 3); };
	auto ten_div = [div](int b)->int {return div(10, b); };
	cout << div(10, 3) << " " << div3(10) << " " << ten_div(3) << endl;
	// 输出都是3
	return 0;
}
```

这只是个简陋的例子，看不出来这有什么用......

再给出使用`std::bind()`的更优雅（繁琐）的写法

```C++
#include <iostream>
#include <functional>

using std::cout;
using std::endl;

void f(int x, int y, const int &z){
	cout << "=> f(" << x << ", " << y << ", " << z << ")" << endl;
}

int g(int x, int y, int z) { return x + y * z; }


int main(int argc,char**argv){
	using namespace std::placeholders;
	int n = 5;
    //std::placeholders::_1,_2... 占位符
	auto fyx5 = std::bind(f, _2, _1, n);
	fyx5(1, 2); // => f(2, 1, 5)
	auto f5yy = std::bind(f, n, std::ref(n), std::cref(n));
	f5yy(); // => f(5, 5, 5)
	n += 2; f5yy(); // => f(5, 7, 7)

	auto gxxx = std::bind(g, _1, _1, _1);
	cout << "5 + 5 * 5 = " << gxxx(5) << endl; // => 5 + 5 * 5= 30
    //这里gxxx(5,1,2,3,4,5)的结果也是一样的，因为占位符都用的第一个参数
    
	auto gx45 = std::bind(std::bind(g, _1, _2, 5), _1, 4);
    //内层std::bind返回一个函数对象，绑定了第三个参数；外层则绑定第二个参数
	cout << "3 + 4 * 5 = " << gx45(3) << endl; // => 3 + 4 * 5= 23

	return 0;
}
```

这个例子其实并没有做到**部分应用**，只是使用了占位符`std::placeholders::_1,_2,...`来实现效果

std::bind()的参数默认是值传递，若需要引用传递，必须使用`std::ref()`或`std::cref()`

更好的写法，一个简单的柯里化尝试

```C++
auto c_xor = [](char _1)->auto{
		return[_1](char  _2)->auto{
			return [_1, _2](char _3) {
				return _1 ^ _2 ^ _3;
			};
		};
	};
```

可以这样调用 `char c = c_xor('a')('b')('c');`



# 柯里化

待学习[知乎专栏](https://zhuanlan.zhihu.com/p/46737243)

也是用变长模板参数定义柯里化规则，和《C++模板元编程实战》中的异类词典、policy有相似之处

不过是真的难

