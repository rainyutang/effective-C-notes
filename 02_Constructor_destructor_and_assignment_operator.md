# 构造、析构、赋值运算

## 05、了解C++默认编写了哪些函数

声明一个空类后，编译器会自动为它声明一个拷贝构造函数、一个copy assignment操作符和一个析构函数。此外如果没有声明任何构造函数，编译器会默认声明一个default构造函数。所有这些函数都是public且inline的。

就好像如下代码：

```cpp
class Empty {
public:
    Empty(){};
    Empty(const Empty& rhs) {};
    ~Empty(){};
    Empty& operator= (const Empty& rhs){};
};
```

注意，编译器所产生的析构函数是non-virtual的，除非这个class的base-class有virtual析构函数。

如果自己声明了一个构造函数，编译器不会再声明defaul构造函数，这很重要，意味着编译器不会为你添加无实参构造函数。
