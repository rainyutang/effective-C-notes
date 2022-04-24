# 4、设计与声明

## 让接口更容易被正确使用，不易被误用

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

