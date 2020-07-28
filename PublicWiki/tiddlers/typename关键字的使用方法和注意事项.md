## 一些关键概念

在我们揭开真实原因的面纱之前，先保持一点神秘感，因为为了更好的理解C++标准，有几个重要的概念需要先行介绍一下。

### 限定名和非限定名

限定名(qualified name)，故名思义，是限定了命名空间的名称。看下面这段代码，`cout`和`endl`就是限定名：

```cpp
#include <iostream>
int main()  {
    std::cout << "Hello world!" << std::endl;
}
```

`cout`和`endl`前面都有`std::`，它限定了`std`这个命名空间，因此称其为限定名。

如果在上面这段代码中，前面用`using std::cout;`或者`using namespace std;`，然后使用时只用`cout`和`endl`，它们的前面不再有空间限定`std::`，所以此时的`cout`和`endl`就叫做非限定名(unqualified name)。

### 依赖名和非依赖名

依赖名(dependent name)是指依赖于模板参数的名称，而非依赖名(non-dependent name)则相反，指不依赖于模板参数的名称。看下面这段代码：

```cpp
template <class T>
class MyClass {
    int i;
    vector<int> vi;
    vector<int>::iterator vitr;
 
    T t;
    vector<T> vt;
    vector<T>::iterator viter;
};
```

因为是内置类型，所以类中前三个定义的类型在声明这个模板类时就已知。然而对于接下来的三行定义，只有在模板实例化时才能知道它们的类型，因为它们都依赖于模板参数`T`。因此，`T`, `vector<T>`和`vector<T>::iterator`称为依赖名。前三个定义叫做非依赖名。

更为复杂一点，如果用了`typedef T U; U u;`，虽然`T`没再出现，但是`U`仍然是依赖名。由此可见，不管是直接还是间接，只要依赖于模板参数，该名称就是依赖名。

### 类作用域

在类外部访问类中的名称时，可以使用类作用域操作符，形如`MyClass::name`的调用通常存在三种：静态数据成员、静态成员函数和嵌套类型：

```cpp
struct MyClass {
    static int A;
    static int B();
    typedef int C;
}
```

`MyClass::A`, `MyClass::B`, `MyClass::C`分别对应着上面三种。

## 引入typename的真实原因

结束以上三个概念的讨论，让我们接着揭开`typename`的神秘面纱。

### 一个例子

在Stroustrup起草了最初的模板规范之后，人们更加无忧无虑的使用了`class`很长一段时间。可是，随着标准化C++工作的到来，人们发现了模板这样一种定义：

```cpp
template <class T>
void foo() {
    T::iterator * iter;
    // ...
}
```

这段代码的目的是什么？多数人第一反应可能是：作者想定义一个指针`iter`，它指向的类型是包含在类作用域`T`中的`iterator`。可能存在这样一个包含`iterator`类型的结构：

```cpp
struct ContainsAType {
    struct iterator { /*...*/ };
    // ...
};
```

然后像这样实例化`foo`：

```cpp
foo<ContainsAType>();
```

这样一来，`iter`那行代码就很明显了，它是一个`ContainsAType::iterator`类型的指针。到目前为止，咱们猜测的一点不错，一切都看起来很美好。

### 问题浮现

在类作用域一节中，我们介绍了三种名称，由于`MyClass`已经是一个完整的定义，因此编译期它的类型就可以确定下来，也就是说`MyClass::A`这些名称对于编译器来说也是已知的。

可是，如果是像`T::iterator`这样呢？`T`是模板中的类型参数，它只有等到模板实例化时才会知道是哪种类型，更不用说内部的`iterator`。通过前面类作用域一节的介绍，我们可以知道，`T::iterator`实际上可以是以下三种中的任何一种类型：

*   静态数据成员
*   静态成员函数
*   嵌套类型

前面例子中的`ContainsAType::iterator`是嵌套类型，完全没有问题。可如果是静态数据成员呢？如果实例化`foo`模板函数的类型是像这样的：

```cpp
struct ContainsAnotherType {
    static int iterator;
    // ...
};
```

然后如此实例化`foo`的类型参数：

```cpp
foo<ContainsAnotherType>();
```

那么，`T::iterator * iter;`被编译器实例化为`ContainsAnotherType::iterator * iter;`，这是什么？前面是一个静态成员变量而不是类型，那么这便成了一个乘法表达式，只不过`iter`在这里没有定义，编译器会报错：

> error C2065: ‘iter’ : undeclared identifier

但如果`iter`是一个全局变量，那么这行代码将完全正确，它是表示计算两数相乘的表达式，返回值被抛弃。

同一行代码能以两种完全不同的方式解释，而且在模板实例化之前，完全没有办法来区分它们，这绝对是滋生各种bug的温床。这时C++标准委员会再也忍不住了，与其到实例化时才能知道到底选择哪种方式来解释以上代码，委员会决定引入一个新的关键字，这就是`typename`。

### 千呼万唤始出来

我们来看看[C++标准](http://en.wikipedia.org/wiki/Typename)：

> A name used in a template declaration or definition and that is dependent on a template-parameter is assumed not to name a type unless the applicable name lookup finds a type name or the name is qualified by the keyword typename.

对于用于模板定义的依赖于模板参数的名称，只有在实例化的参数中存在这个类型名，或者这个名称前使用了`typename`关键字来修饰，编译器才会将该名称当成是类型。除了以上这两种情况，绝不会被当成是类型。

因此，如果你想直接告诉编译器`T::iterator`是类型而不是变量，只需用`typename`修饰：

```cpp
template <class T>
void foo() {
    typename T::iterator * iter;
    // ...
}
```

这样编译器就可以确定`T::iterator`是一个类型，而不再需要等到实例化时期才能确定，因此消除了前面提到的歧义。

### 不同编译器对错误情况的处理

但是如果仍然用`ContainsAnotherType`来实例化`foo`，前者只有一个叫`iterator`的静态成员变量，而后者需要的是一个类型，结果会怎样？我在Visual C++ 2010和g++ 4.3.4上分别做了实验，结果如下：

Visual C++ 2010仍然报告了和前面一样的错误：

> error C2065: ‘iter’ : undeclared identifier

虽然我们已经用关键字`typename`告诉了编译器`iterator`应该是一个类型，但是用一个定义了`iterator`变量的结构来实例化模板时，编译器却选择忽略了此关键字。出现错误只是由于`iter`没有定义。

再来看看g++如何处理这种情况，它的错误信息如下：

> In function ‘void foo() \[with T = ContainsAnotherType\]’:instantiated from hereerror: no type named ‘iterator’ in ‘struct ContainsAnotherType’

g++在`ContainsAnotherType`中没有找到`iterator`类型，所以直接报错。它并没有尝试以另外一种方式来解释，由此可见，在这点上，g++更加严格，更遵循C++标准。

### 使用typename的规则

最后这个规则看起来有些复杂，可以参考[MSDN](http://msdn.microsoft.com/zh-cn/library/8y88s595.aspx)：

*   typename在下面情况下禁止使用：
    *   模板定义之外，即typename只能用于模板的定义中
    *   非限定类型，比如前面介绍过的`int`，`vector<int>`之类
    *   基类列表中，比如`template <class T> class C1 : T::InnerType`不能在`T::InnerType`前面加typename
    *   构造函数的初始化列表中
*   如果类型是依赖于模板参数的限定名，那么在它之前必须加typename(除非是基类列表，或者在类的初始化成员列表中)
*   其它情况下typename是可选的，也就是说对于一个不是依赖名的限定名，该名称是可选的，例如`vector<int> vi;`

### 其它例子

对于不会引起歧义的情况，仍然需要在前面加`typename`，比如：

```cpp
template <class T>
void foo() {
    typename T::iterator iter;
    // ...
}
```

不像前面的`T::iterator * iter`可能会被当成乘法表达式，这里不会引起歧义，但仍需加`typename`修饰。

再看下面这种：

```cpp
template <class T>
void foo() {
    typedef typename T::iterator iterator_type;
    // ...
}
```

是否和文章刚开始的那行令人头皮发麻的代码有些许相似？没错！现在终于可以解开`typename`之迷了，看到这里，我相信你也一定可以解释那行代码了，我们再看一眼：

```
typedef typename __type_traits<T>::has_trivial_destructor trivial_destructor;
```

它是将`__type_traits<T>`这个模板类中的`has_trivial_destructor`嵌套类型定义一个叫做`trivial_destructor`的别名，清晰明了。

### 再看常见用法

既然`typename`关键字已经存在，而且它也可以用于最常见的指定模板参数，那么为什么不废除`class`这一用法呢？答案其实也很明显，因为在最终的标准出来之前，所有已存在的书、文章、教学、代码中都是使用的是`class`，可以想像，如果标准不再支持`class`，会出现什么情况。

对于指定模板参数这一用法，虽然`class`和`typename`都支持，但就个人而言我还是倾向使用`typename`多一些，因为我始终过不了`class`表示用户定义类型这道坎。另外，从语义上来说，`typename`比`class`表达的更为清楚。C++ Primer也建议使用`typename`:

> 使用关键字typename代替关键字class指定模板类型形参也许更为直观，毕竟，可以使用内置类型（非类类型）作为实际的类型形参，而且，typename更清楚地指明后面的名字是一个类型名。但是，关键字typename是作为标准C++的组成部分加入到C++中的，因此旧的程序更有可能只用关键字class。

## 参考

1.  C++ Primer
2.  Effective C++
3.  [A Description of the C++ typename keyword](http://pages.cs.wisc.edu/~driscoll/typename.html)
4.  [维基百科typename](http://zh.wikipedia.org/wiki/Typename)
5.  另外关于`typename`的历史，Stan Lippman写过一篇[文章](http://blogs.msdn.com/b/slippman/archive/2004/08/11/212768.aspx)，Stan Lippman何许人，也许你不知道他的名字，但看完这些你一定会发出，“哦，原来是他！”：他是 C++ Primer, Inside the C++ Object Model, Essential C++, C# Primer 等著作的作者，另外他也曾是Visual C++的架构师。
6.  在[StackOverflow](http://stackoverflow.com/a/613132/973315)上有一个非常深入的回答，感谢@Emer 在本文评论中提供此链接。