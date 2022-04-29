# 4、设计与声明

## 18、让接口更容易被正确使用，不易被误用

理想上，如果客户使用了某个接口但并没有获得他所预期的行为，这段代码不应该通过编译。如果代码通过了编译，它的作为就应该是客户想要的。

例如，如下代码很容易造成使用错误：

```cpp
class Data {
public:
    Data(int month, int day, int year);
};

Data d(3,20,1999);
```

可以修改为：

```cpp
class Data {
public:
    Data(const Month& m, const Day& d, const Year& y);
};

class Month {
public:
    explicit Month(int m) : _val(m) {}
private:
    int _val;
};

class Day {
    ...
};

class Year {
    ...
};

Data d(Month(3), Day(20), Year(1999)); //正确
Data d(Month(3), Year(1999), Day(20)); //错误，类型不正确
Data d(3, 20, 1999); //错误，类型不正确
```

如果想要对月份进行限制：

```cpp
class Month {
public:
    static Jan() {return Month(1)};
    static feb() {return Month(2)};
    ...
private:
    explicit Month(int m) : _val(m) {}
    ...
};

Data d(Month::Jan(), ..., ...);
```

预防客户端错误的另一个方法是，限制类型什么事情可以做，什么不可以做。常见的限制是加上const，例如 以const修饰 operator* 的返回类型，防止出现 ```a * b = c```。

另一个一般性准则是：让你的types与内置types一致。

要提供**行为一致的接口**，例如STL每个容器都有size()方法，STL的接口一致使它们很容易被使用。

任何接口如果要求客户必须记得做某些事情，就是有着“不正确使用”的倾向。如条款13的例子

```cpp
Investment* CreateInventment();
```

要求客户必须释放指针指向的内容，或者客户使用智能指针。

改进是返回一个智能指针：

```cpp
std::shared_ptr<Investment> CreateInvestment();
```

进一步，如果希望客户将Investment指针使用getRidOfInvestment函数（而不是delete），同样可以使用shared_ptr，也就是将getRidOfInvestment绑定为shared_ptr的删除器：

```cpp
std::shared_pt<Investment> createInvestment()
{
    //将0强制转换成Investment*， 当然，如果在这之前已经知道Investment*的值，可以直接用该值初始化，而不必用0初始化再进行赋值。
    std::shared_ptr<Investment> retVal(static_cast<Investment*>(0), getRidOfInvestment());
    retVal = ...; //令retVal指向正确的对象
    return retVal;
};
```

* **好的接口很容易被正确使用，不容易被误用。你应该在你的所有接口中努力达成这些性质。**

* **“促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容。**

* **“阻止误用”的办法包括建立新类型、限制类型上的操作、束缚对象值以及消除客户的资源管理责任。**

* **shared_ptr支持定制删除器（custom deleter）。这可以防范DLL问题，可被用来自动解除互斥锁等**

## 19、设计class犹如设计type

需要注意一下问题：

* **新type的对象应该如何被创建和销毁？** 这会影响到你的class的构造函数和析构函数以及内存分配函数和释放函数。

* **对象的初始化和对象的赋值该有什么样的差别？** 这个答案决定了构造函数和赋值操作符的行为，一起其差异。注意不要混淆初始化与赋值，因为其对应着不同的函数调用。

* **新type的对象如果被passed by value，意味着什么？** copy构造函数用来定义一个type的pass-by-value如何实现。

* **什么是新type的“合法值”？** 对class成员变量而言，通常只有某些数值集是有效的，这些数值决定了class需要维护的约束条件，也就决定了成员函数（特别是构造函数、赋值操作符和所谓的setter函数）必须进行的错误检查工作。它也影响函数抛出的异常、以及函数异常明系列（很少用）。

* **你的新type需要配合继承某个继承图系（inheritance graph）吗？** 如果继承自某些既有的classes，就需要受到那些classes的设计的约束，需要注意它们的函数是virtual或者non-virtual的（条款34、36）。如果允许其他class继承你的class，也会影响你声明的函数，特别是析构函数，是否是virtual的。

* **你的新type需要怎样的转换？** 如果希望T1被隐式转换为T2，需要在T1内写一个类型转换函数（operator T2）或者在T2内写一个non-explicit-one-argument的构造函数。如果只允许explicit构造函数存在，就得写出专门的负责执行转换的函数，且不得为类型转换操作符或者non-explicit-one-argument构造函数（条款15有范例）。

* **什么样的操作符和函数对此新type是合理的？** 这个问题的答案决定class声明哪些函数，其中某些是member函数，某些不是。

* **什么样的函数标准应该驳回？** 这些需要声明为private

* **谁该取用新的type成员？** 这个问题可以帮助你决定哪个成员为public，哪个为protected、哪个为private。也会帮助你决定哪个class或者function是friend，以及将它们嵌套于另一个是否合理。

* **什么是新type的“未声明接口”（undeclared interface）？** 他对效率、异常安全性（条款29）以及资源运用（例如多任务锁定和动态内存）提供何种保证？应该据此为代码添加约束条件。

* **你的新type有多么一般化？** 或许你其实并非定义一个新type，而是定义一个types家族。如果这样就不应该定义一个新class，而应该定义一个新的class template。

* **你真的需要一个新class吗？** 如果只是定义新的derived class以便为既有的class添加功能，说不定单纯定义一个或多个non-member函数或templates更好。

* **class的设计就是type的设计，在定义一个新class之前，请确定你已经考虑过上述问题。**

## 20、宁以pass-by-reference-to-const替换pass-by-value

以传值的方式调用会调用多次copy构造函数和析构函数，所以需要pass-by-reference-to-const:

```cpp
bool validateStudent(const Student& s);
```

以by reference方式传递参数也可以避免slicing（对象切割）问题。

例如：

```cpp
class Windows {
    ...
};

class WindowWithScrollBars : public Windows {
    ...
};

void printNameAndDisplay(Windows w){
    ...
}

WindowsWithScrollBars wwsb;
printNameAndDisplat(wwsb);
```

调用上述函数，参数w会被构造成为一个windows对象，他是passed by value。解决方式就是以pass by reference-to-const方式传递，传进来的是哪种类型，表现的就是哪种类型。

```cpp
void printNameAndDisplay(const Windows& w)
{
    ...
}
```

某些内置类型（如int）可能pass by value比pass by reference效率更高些。

但并不是所有小型tpes都适合用pass by value。对象小并不意味着它的copy构造函数不昂贵。许多对象--包括大部分STL容器--内含的东西只比一个指针多一些，但复制这种对象却需要承担“复制那些指针所指的每一个东西”。这也是和很昂贵的。

另一个理由是作为一个用户定义的类型，可能目前非常小，但将来也许会变大，甚至当你改用另一个c++编译器都有可能改变type的大小。

一般而言，pass by value的对象只包括内置类型、STL的迭代器和函数对象，其他任何东西都应该pass by reference-to-const

* **尽量以pass by reference-to-const代替pass by value，前者通常比较搞笑，且能避免切割问题。**

* **以上规则并不适用于内置类型，以及STL迭代器和函数对象。对它们而言，pass by value比较合适。**

## 21、必须返回对象时，别妄想返回其reference

很容易错误地使用pass by reference，例如：

```cpp
const Rational& operator (const Rational& lhs, const Rational& rhl)
{
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
    return result;
}
//错误，返回了一个不存在的对象。
```

如果使用new再heap内构造一个新对象然后返回，例如：

```cpp
const Rational& operator (const Rational& lhs, const Rational& rhl)
{
    Rational* result  = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    return *result;
}
//错误，申请的对象无法被delete
```

还有一种错误做法是返回的reference指向被定义于函数内部的static Rational对象，例如：

```cpp
const Rational& operator (const Rational& lhs, const Rational& rhl)
{
    static Rational result;
    result = ...;
    return result;
}
//除了会破坏线程安全，以下的代码也会发生错误
Rational a, b, c, d;
if(a * b == b * c)
    ....
//上述的值总是true
```

因此，返回新对象是唯一正确的做法：

```cpp
inline const Rational operator * (const Rationl& lhs, const Rational& rhs)
{
    Rational result = ...;
    return result;
}
```

当然，这需要承受operator*返回值的构造和析构成本，但这只是为获得正确行为的一个小小代价，改善代码效率的行为应交给编译器去考虑。

* **绝不要返回pointer或reference指向一个local stack对象，或返回reference指向一个heap-allocated对象，或返回pointer或reference指向一个local static对象，因为有可能同时需要多个这样的对象。条款4已经为“在单线程环境中合理返回reference指向一个local static对象”提供了一份设计实例。**

## 22、将成员变量声明为private

理由如下：

首先，从语法一致性而言，如果public内每样东西都是函数，客户就不需要在访问class是思考是否需要加括号，他们只需要加就行了，因为每样东西都是函数。

如果令成员变量为public，每个人都可以读写它，但如果以函数的方式取得或设定其值，你就可以实现“不准访问”、“只读访问”、“读写访问”等，甚至可以实现“唯写访问”。

细微的访问控制是必要的，许多成员变量应该被隐藏起来，并不是所有变量都需要getter函数或者setter函数。

另一个重要的原因是封装，将成员变量隐藏在函数接口的背后，可以为“所有可能的实现”提供弹性。例如可使得成员变量被读或写时轻松通知其他对象、可以验证class的约束条件以及函数的前提和事后状态、可以在多线程环境中执行同步控制等。

成员的封装性与“当成员变量的内容改变时所破坏的代码数量”成反比。所以protected的封装性并非高于public。

当public或者protected的被修改，太多的derived class代码或者客户代码需要被重写。

因此，其实只有两种访问权限：private(提供封装)和其他（不提供封装）。

* **切记将成员变量声明为private。这是赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性。**

* **protected并不比public更具封装性。**

## 23、宁以non-member、non-friend替换member函数

考虑这样一个class，提供清除缓冲区、清除历史记录、清除cookies等函数。

```cpp
class WebBrowser {
public:
    ...
    void clearCache();
    void clearHistory();
    void removeCookies();
    ...
};
```

许多用户向执行上述三个动作，因此WebBrower可以提供这样一个函数：

```cpp
class WebBrowser {
public:
    ...
    void clearEverything(); //调用以上三个函数
} 
```

当然， 这也可以由一个non-member函数调用适当的member函数提供：

```cpp
void clearBrowser(WebBrowser& wb)
{
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
```

member函数带来的封装性比non-member函数更低，且non-member函数对WebBrowser的相关机能有较大的包裹性。

通常，对于一个数据而言，越多的函数可访问它，它的封装性越低。

我们考虑private成员，能访问class的成员只有class的member函数加上friend函数而已。所以，如果要实现相同的功能，而提供选择的有member函数与non-member且non-friend函数，选择non-member的函数能提供较大封装性。

换句话说，如果private成岩发生改变，那么member函数也需要相应发生变化，然而non-member函数由于不能访问private成员，则不需更改。

另一件值得注意的事是，non-member函数并不意味着它不能包括在任何class内部，如果定义一个用于管理额工具类，包括一个static member成员函数，这也是可以的，只要它不会破坏WebBrowser的封装性。

在C++中，比较自然的一个做法是让clearBrower成为一个non-member函数并且位于WebBrower所在的同一个namespace（命名空间）内：

```cpp
namespace WebBrowserStuff {
    class WebBrowser {...};
    void clearBrowser(WebBrowser& wb);
    ...
}
```

这并不是为了仅仅看起来自然，namespace与class不同，前者可以位于跨越多个源码文件，而后者不能。


对于class而言，可能会包括很多个与其有关的便利函数，而我们不必编译所有的函数，分离它们最好的做法就是将它们放在不同的头文件内，但是它们都位于相同的namespace。

这也正是c++标准库的组织方式。并不是一个头文件包括std namespace内的所有内容，而是分别位于不同的头文件中，例如vector、list等。用户可以按需引入。

此外，用户如果要扩展遍历函数，只需再建立一个头文件，且位于相同的namespace下。这是class无法提供的性质。

* **宁可拿non-member non-friend函数替换member函数，这样做可以增加封装性、包裹弹性（packaging flexibility）和机能扩充性。

## 若所有参数皆需类型转换，请为此采用non-member函数

令class支持隐式类型转换是个糟糕的主意，但是这也有例外，最常见的例外是建立数值类型时：

```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1); //构造函数不为explicit
    int numerator() const;
    int denominator() const;
private：
    ...
};
```

如果想要支持乘法,一种方法是再class内部实现operator*：

```cpp
class Rational {
public:
    const Rational operator* (cosnt Rational& rhs) const;
};

//调用
Rational a, b;
Rational result;
result = a * b; //正确
result = a * 2; //正确，正确的原因是再调用 a.operator*(2) 时发生了隐式类型转换
                //如果Rational的构造函数时explicit的，这个也是错误的
result = 2 * a; //错误 调用的是2.operator*(a)
```

编译器也会尝试寻找non-member operator*（也就是再namespace内或者global作用域内）：

```cpp
result = operator*(2, a); //错误,并没有找到
```

解决的办法就是让operator*成为一个non-member函数，允许编译器在每一个实参上都执行隐式类型转换：

```cpp
const Rational operator*(const Rational& lhs, const Rational& rhs){
    return ...;
}
```

这个函数能不成为friend就不成为friend，member函数的反面是non-member函数，而不是friend函数。

当Rational成为一个template而不是class，又会带来一些新争议、新解法以及新的设计牵连，见条款46。

* **如果你需要为某个函数的所有参数（包括this指针指向的那个隐含参数）进行类型转换，那么这个函数必须是non-member。
