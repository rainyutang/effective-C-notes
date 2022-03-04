# 让自己习惯C++

## 01.视c++为一个语言联邦

## 02.尽量以`const`、`enum`、`inline`代替`#define`

或者说以编译器替换预处理器

```C++
#define ASPECT 1.653
```

`ASPECT`可能从未被编译器看到，如果报错很可能报1.653而不是`ASPECT`

解决办法是以常量代替上面的宏：

```C++
const double Aspect
```

`Aspect`肯定会被编译器看到，并记入记号表内。同时，使用`const`可能会比`#define`产生更少的代码，因为`#define`可能会被预处理器盲目地替换为多份，而`const`不会出现这种情况。

当我们以常量替换`#define`，应考虑两种特殊的情况。

1. 定义常量指针，必须写`const`两次：

    ```C++
    const char* const Name = "Rain";
    ```

    一个更好的办法是使用String：

    ```C++
    const std::string Name("Rain");
    ```

2. class专属常量

    为了将常量的作用域限制在class内部，它必须是一个class成员，为确保此常量至多只有一份实体，它必须是一个`static`成员。

    ```c++
    class GamePlayer{
    private:
        static const int NumTurns = 6; //这是一个声明式
        int scores[NumTurns];
    };
    ```

    如果它是一个class专属常量且是static且为整数类型（int chat bool），只要不取地址，只需声明而无需定义，如果要取地址，需要在实现文件中而不是头文件中定义：
    变量定义：用于为变量分配存储空间，可为变量指定初始值，变量有且只有一个定义。

    ```cpp
    const int GamePlayer::NumTurns; //定义
    ```

    因为已经在声明时获得初值，定义时可以不再赋值。

    **无法用`#define`创建一个class专属常量**，这意味着`#define`无法提供任何封装性。

    旧式编译器也许不支持上述语法，它们不允许static成员在其声明式上获得初值。而且所谓的“in-class 初值设定”也只允许对整数常量进行，如果编译器不支持上述语法，可以将初值放在定义式：

    ```cpp
    class CostEstimate{
    private:
        static const double FudgeFactor; //static class 常量声明
        ... //位于头文件内
    };

    const double CostEstimate::FudgeFactor = 1.35; //位于实现文件内
    ```

    还有一个例外是当在class编译期间需要一个class常量值，万一编译器不允许“in-class初值设定”，可以使用“The enum hack”补偿做法。一个属于枚举类型的数值可权充ints被使用，于是可定义如下：

    ```cpp
    class GamePlayer{
    private:
        enum {NumTurns = 5};

        int scores[NumTurns];
    };
    ```

    * enum hack的行为某方面说比较像`#define`而不是`const`，例如取一个const的地址是合法的，但取一个enum的地址是不合法的，取#define的地址也是不合法的。如果不想让别人用指针或引用指向你的某个整数常量，可以用enum实现这个约束。

    * 使用enum纯粹是为了实用主义，“enum hack”是模板元编程的基础技术。

另一个常见的`#define`误用情况是以它实现宏（macros）。宏看起来像函数，但不会导致函数调用带来的额外开销。例如：

```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

缺点太多。

可以使用template inline函数：

```cpp
template<typename T>
inline void callWithMax(const T& a, const T& b){
    f(a > b ? a : b);
}
```

不需要在函数本体中为参数加上括号，也不需要担心参数被求值多次。

我们对预处理器的需求并未完全消除，`#include`仍是必需品，`ifdef/infdif`仍扮演者控制编译的角色。

总结：

* **对于单纯常量，最好以`const`对象或enums替换`#define`。**

* **对于形似函数的宏（macros），最好用inline函数替换`#define`。**

## 02.尽可能使用`const`

如果关键字`const`出现在星号左边，说明被指物是常量，如果出现在星号右边，说明指针是常量。如果出现在两边，说明都是常量。

以下两种写法意义相同：

```cpp
void f1(const Widget* pw);

void f2(Widget const * pw);

//传入的参数都是一个指针，指向一个常量Widget对象。
```

以下是有关STL的，迭代器的作用相当于T*指针。声明迭代器为const像声明指针为const，表示这个迭代器不能指向其他内容，但它所指的内容是可以改动的。如果希望被迭代器所指的东西是不可改动的，需要`const_iterator`：

```cpp
std::vector<int> vec;

const std::vector<int>::iterator iter = vec.begin(); //iter作用类似 T* const

*iter = 10; //正确
++iter; //错误

std::vector<int>::const_iterator cIter = vec.begin(); //cIter类似const T*

*cIter = 10; //错误
++iter; //正确
```

`const`最具威力的做法是函数声明时的应用。在一个函数声明式内，`const`可以和函数返回值、各参数、函数自身（如果是成员函数）产生关联。

令函数返回是一个常量值，可以降低因客户错误造成的意外：

```cpp
class Rational {...};
const Rational operator* (const Rational& lhs, const Rational& rhs);

//这样声明可以避免以下这种情况：
Rational a,b,c;
(a * b) = c; //在a * b的结果上调用operator*
```

对于const参数，除非有需要改动的参数或local对象，否则请将它们声明为const。

### const成员函数

将const作用于成员函数，是为了确认该成员函数可以用于const对象身上。这类成员函数重要的两个理由：

* 使class接口容易被理解，可以知道哪个函数可以改动对象，哪个不行。

* 使操作const对象成为可能

有一个容易被忽视的事实：**两个成员函数如果只是常量性（constness）不同，可以被重载**

```cpp
class TextBlock {
public:
    ...
    const char& operator[] (std::size_t position) const //operator[] for const 对象
    {return text[positon];}

    char& operator[] (std::size_t position) //operator[] for non-const 对象
    {return text[position];}

private:
    std::string text;
}
```

真实程序中const对象大多用于passed by pointer-to-const 或者 passed reference-to-const，例如：

```cpp
void print(const TextBlock ctb)
{
    std::cout << ctb[0]; //调用const TextBlock::operator[]
    ...
}
```

需要注意，`operator[]`的返回类型是reference to char，而不是 char。如果返回的是char，下面的语句无法通过编译：

```cpp
TextBlock tb("Hello");
tb[0] = 'x';
```

有两种const成员函数的说法。一种是，如果class内部有一个指针。那么不改变指针的函数就可以是const成员函数。另一种是const成员函数需要指针指向的值也不改变。

一个使用方法：

```cpp
class CTextBlock {
public:
    ...
    std::size_t length() const;

private:
    char* pText;
    mutable std::size_t textLength; //用nutable表示这些变量总会被修改，即使是在const成员函数内。
    mutable bool lengthIsValid;
};

std::size_t CTextBlock::length() const
{
    if(!lengthIsValid){
        textLength = std::strlen(pText);
        lengthIsValid = true;
    }
    return textLength;
}
```

### 在const和non-const成员函数中避免重复

对于“bitwise-constness非我所欲”问题，`mutable`是一个解决办法，但不能解决所有问题。

假设`operator[]`执行很多功能，如果像上文编写两个重载函数，代码冗余。因此可以利用常量性转除。令non-const代码调用其const兄弟，这个过程需要一个转型动作。

```cpp
class TextBlock {
public:
    ...
    const char& operator[] (std::size_t position) const{
        ...
        return text[position];
    }

    char& operator[] (std::size_t position) {
        return
            const_cast<char&>( //将op[]返回值的const转除
                static_cast<const TextBlock&>(*this)[position] //为*this加上const并调用const op[]
            );
    }
private:
    std::string text;
}
```

**不要用const成员函数调用non-const成员函数**。

总结：

* 将某些东西声明为const可以帮助编译器侦测出错误用法。const可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。

* 编译器强制实施bitwise-constness，但编写程序时应该使用“概念上的常量性”（conceptual-constness）。

* 当const和non-const成员函数有着实质等价的实现时，令non-const版本调用const版本可避免代码重复。
