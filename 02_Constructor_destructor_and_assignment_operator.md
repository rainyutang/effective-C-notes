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

如下是一个NamedObject template

```cpp
template<typename T>
class NamedObject {
public:
    NamedObject(const char* name, const T& value);
    NamedObject(const std::string& name, const T& value);
    ...

private:
    std::string nameValue;
    T objectValue;
};
```

如果自己声明了一个构造函数，编译器不会再声明defaul构造函数，这很重要，意味着编译器不会为你添加无实参构造函数。

NamedObject既没有copy构造函数，也没有copy assignment操作符，所以编译器会创建那些函数。

copy构造函数的用法：

```cpp
NamedObject<int> no1("Smallest Prime Number", 1);
NamedObject<int> no2(no1) //调用copy构造函数
```

编译器以```no1.nameValue```和```no1.objectValue```为初值初始化no2。nameValue类型为string，string有copy构造函数，所以会先调用string的copy构造函数并以no1.nameValue为实参，另一个成员是int，那是一个内置类型，no2.objectVale会copy no1.objectValue的每一个bits来完成初始化。

如果一个类内部含有reference成员或者const成员，编译器拒绝编译赋值操作，必须自己定义copy assignment操作符。

如果某个base class将copy assignment操作符声明为private，编译器拒绝为其derived classes生成copy assignment操作符。毕竟编译器为derived classes所生成的copy assignment操作符理应处理base class的部分，但是它们无法调用derived class无权调用的成员函数。

* **编译器可以自行为class创建default构造函数、copy构造函数、copy assignment操作符以及析构函数**

## 06、若不想使用编译器自动生成的函数，就应该明确拒绝

如果不想对象被拷贝，例如：

```cpp
HomeForSale h1, h2;
HomeForSale h3(h1); //不该通过编译
h1 - h2; //也不该通过编译
```

可以将copy构造函数和copy assignment声明为private：

```cpp
class HomoForSale {
public:
    ...

private:
    ...
    HomeForSale(const HomeForSale&);
    HomeForSale& operator=(const HomeForSale);
};
```

不需指定参数名称。当客户企图拷贝HomeForSale，编译器会阻挠，如果friend函数或者member函数之内这么做，连接器报错。

将连接期的错误转移至编译期是可能的，需要在一个专门阻止copying动作而设计的base class内，将copy函数和copy assignment声明为private。

```cpp
class Uncopyable{
protected:
    Uncopyable(){};
    ~Uncopyable(){};

private:
    Uncopyable(const Uncopyable);
    Uncopyable& operate=(const Uncopyable);
};

class HomeForSale:private Uncopyable{
    ...
};
```

* **为驳回编译器自动提供的功能，可将相应成员函数声明为private且不予实现。使用像Uncopyable这样的base class也是一种做法。**

## 07、为多态基类生成virtual函数

例如一个记录时间的base class：

```cpp
class TimeKeeper{
public:
    TimeKeeper();
    ~TimeKeeper();
    ...
};

class AtomicClock: public Timekeeper{...};
class WaterClock: public TimeKeeper{...};
class WristWatch: public TimeKeeper{...};
```

可以设计一个factory函数，返回一个指针指向计时对象：

```cpp
TimeKeeper* getTimeKeeper();
```

为了避免泄露，将factory函数返回的每一个对象用delete释放很重要：

```cpp
TimeKeeper* ptk = getTimeKeeper();
...
delete ptk;
```

getTimeKeeper()返回一个derived class对象，而指向它的指针则是base class的，目前base class有一个non-virtual析构函数。c++明确指出：**derived class对象经由base class对象指针释放，而该base class带有一个non-virtual的析构函数，其结果未定义，通常调用base class额析构函数，会导致derived class对象的一部分未被销毁。**

消除方法很简单，给base class一个virtual析构函数：

```cpp
class TimeKeeper{
public:
    TimeKeeper();
    virtual ~TimeKeeper();
    ...
};

TimeKeeper* ptk = getTimeKeeper();
delete ptk;
```

现在行为正确，析构函数在不同的的derived class内有不同的实现。如果class不含virtual函数，通常表示不意图被作为一个base class。同样，**当class不意图被作为base class， 不要令其析构函数为virtual。**

因为，如果想要实现virtual，对象必须包含某些信息，用来决定在运行时期哪一个virtual函数被调用，这份信息通常是一个vptr（virtual table pointer）。vptr指向一个由函数指针构成的数组，称为vtbl（virtual table）。每一个带有virtual函数的class都有一个相应的vtbl。对象调用某一个virtual函数，实际调用的函数取决于该对象vptr指向的vtbl。

许多人的心得是：**只有当class内含至少一个virtual函数，才为它声明virtual析构函数。**

有时候令class带有一个pure virtual函数可能更为便利。pure virtual函数导致abstract classes，也就是不能被实体化。如果希望建立一个抽象class，但是没有pure virtual函数，就可以将析构函数定义为pure virtual。

```cpp
class AWOV{
public:
    virtual ~AWOV() = 0;
}
```

这里有个窍门，你必须为这个pure virtual析构函数提供一份定义：

```cpp
AWOV::~AVOV(){}
```

因为，析构函数的运行方式是，最深层的derived的class的析构函数最先被调用，然后是每一个base class的析构函数被调用。编译器会在AWOV的derived class析构函数中创建一个对~AWOV()的调用，所以必须为这个函数提供一份定义，否则连接器会报错。

给base class一个virtual析构函数，这个规则只适用于polymorphic（带多态的）base class身上，这种base class的设计目的是使用base class的接口处理derived class对象。

并非所有base class的设计目的都是为了多态用途。例如标准的string和STL容器都不被设计作为base class使用。

* **polymorphic base class应该声明一个virtual析构函数，如果class带有任何virtual函数，他就应该拥有一个virtual析构函数。**

* **class的设计目的如果不是作为base class使用，或者不是为了具备polymorphically，就不应该声明virtual析构函数。**

## 08、别让异常逃离析构函数

C++并不禁止析构函数吐出异常，但它并不推荐这样做。析构函数吐出异常，程序可能过早结束或产生不明确行为。

该怎么办呢？举个例子，假如使用一个class负责数据库连接：

```cpp
class DBConnection {
public:
    ...
    static DBConnection create();

    void close(); //关闭联机，失败则抛出异常
};
```

为确保客户不忘记在DBConnection对象身上调用close(),一个合理的想法是创建一个用来管理DBConnection资源的class。

```cpp
class DBConn {
public:
    ...
    ~DBConn()
    {
        db.close();
    }
private:
    DBConnection db;
}
```

客户可以写如下的代码：

```cpp
{
    DBConn dbc(DBConnection::create());
    ...
}
```

如果该调用导致异常，DBConn析构函数会传播该异常，也就是允许它离开这个析构函数，会造成问题。

两个办法可以避免这个问题：

1. 抛出异常就结束程序：

```cpp
DBConn::~DBConn()
{
    try{db.close();}
    catch(...) {
        制作运转记录，记下对close的调用失败；
        std::abort();
    }
}
```

强迫结束程序是一个合理的选项，可以阻止异常从析构程序传播出去

2. 吞下异常：

```cpp
DBConn::~DBConn()
{
    try{db.close();}
    catch(...) {
        制作运转记录，记下对close的调用失败；
    }
}
```

一般而言，吞下异常是个坏主意。

一个较佳的策略是重新设计DBConn接口，使其客户有机会对可能出现的问题作出反应。例如DBConn自己创建一个close()函数，赋予客户一个机会处理“因该操作而发生的异常”。DBConn也可以追踪其管理的DBConnection对象是否被关闭，如果未被关闭则在析构函数中关闭，然而如果DBConnection析构函数调用失败；仍将退回上两个套路。

```cpp
class DBConn {
public:
    ...
    void close()
    {
        db.close();
        closed = true;
    }

    ~DBConn()
    {
        if(!closed){
            try{
                db.close();
            }
            catch(...){
                制作运转记录，记下对closed的调用失败；
                ...
            }
        }
    }
private:
    DBConnection db;
    bool closed;
}
```

* **析构函数绝不要吐出任何异常，如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下它们（不传播）或结束程序。**

* **如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数（而不是在析构函数中）执行该操作。**

## 09、绝不在构造和析构过程中调用virtual函数

重点：你不该在构造函数和析构函数中调用virtual函数，这样调用不会带来预想的结果，这是c++与java或者c#不相同的一个地方。

假设一个class继承体系，模拟股市交易买进、卖出订单等。这样的交易一定经过审计，所以没创建一个交易对象，在审计日志中也需要创建一笔适当的记录。

```cpp
class Transaction {
public:
    Transaction();
    virtual void logTransaction() const = 0; //加const不改变类的数据成员

    ...
};

Transaction::Transaction()
{
    ...
    logTransaction();
}

class BuyTransaction: public Transaction{
public:
    virtual void logTransaction() const;
    ...
};

class SellTransaction: public Transaction {
public:
    virtual void logTransaction() const;
    ...
};
```

现在，当一下被执行，会发生什么事：

```cpp
BuyTransaction b;
```

在创建derived class之前，会先构造base class，也就是执行base class的构造函数，这时derived class的对象还未生成，所以执行base class时调用的虚函数是base class内的虚函数，如果是纯虚函数，会有编译器或连接器的报错，但如果是普通虚函数，则会调用base class内的版本。能够避免此问题的唯一做法是：**确定你的构造和析构函数都没有在对象被创建和销毁期间调用virtual函数，而他们所调用的所有函数都服从这一约束。**

一种解决方案是将logTransaction函数改为non-virtual，然后要求derived class构造函数传递必要信息给Transaction构造函数：

```cpp
class Transaction {
public:
    explicit Transaction(const std::string& logInfo);
    void logTransaction(const std::string& logInfo) const;
    ...
};

Transaction::Transaction(const std::string& logInfo)
{
    ...
    logTransaction(logInfo);
}

class BuyTransaction: public Transaction {
public:
    BuyTransaction(parameters)
    : Transaction(createLogString(parameters)) //将log信息传递给base class构造函数
    {...}
    ...
private:
    static std::string createLogString(parameters);
};
```

换句话说，你无法使用virtual函数从base class向下调用，但可以另derived class将必要的构造信息向上传递至base class构造函数来代替。

注意本例的createLogString函数，比起在成员初值列内给予base class所需的数据，利用辅助函数创建一个值给base class往往更加方便，也比较可读。另此函数为static，避免意外调用未初始化的derived class成员

* **在构造和析构期间不要调用virtual函数，因为这类调用不会下降至derived class（比起当前执行构造函数和析构函数的那层）。**

## 10、令`operator=`返回一个reference to `*this`

关于赋值，你可以把它们写成连锁形式：

```cpp
int x, y, z;
x = y = z = 5;
```

赋值采用右结合律，以上被解析为：

```cpp
x = (y = (z = 5));
```

为了实现连锁复制，赋值操作必须返回一个reference指向操作符的左侧实参，这是为classes实现赋值操作应该遵守的协议：

```cpp
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs) //返回类型是reference，指向当前对象。
    {
        ...
        return *this; //返回左侧对象
    }
    ...
};
```

这个协议不仅适用于以上的标准赋值形式，也适用于所以相关复制运算，例如：

```cpp
class Widget {
public:
    ...
    Widget& operator+=(const Widget& rhs) //这个协议适用于+=、*=等
    {
        ...
        return *this;
    }

    Widget& operator=(int rhs) //此函数也适用，即使此操作符的参数类型不符合协定
    {
        ...
        return *this;
    }
    ...
};
```

## 11、在`operator=`中处理“自我赋值”

“自我赋值”发生在对象被赋值给自己时：

```cpp
class Widget {...}'
Widget w;
...
w = w;
```

自我赋值并不总是被一眼看出来，例如：

```cpp
a[i] = a[j];
*px = *py;
```

这些不明显的自我赋值是 别名 带来的结果。实际上两个对象只要来自同一个继承体系，它们甚至不需要被声明为相同类型。

如果你尝试自行管理资源，就可能造成在停止使用资源前意外释放了它。假设建立一个class用来保存一个指针指向一块动态分配的位图（bitmap）：

```cpp
class Bitmap {...};
class Widget {
    ...
private:
    Bitmap* pb;
};
```

下面是operator=的实现代码，表面上看起来合理，但自我赋值时并不安全：

```cpp
Widget& Widget::operator=(const Widget& this)
{
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}

当rhs是自身是，它会释放自身的pb，且持有一个指针指向已删除的对象。

传统做法是在最前面给加一个证同测试：

```cpp
Widget& Widget::operator=(const Widget &rhs){
    if(&rhs == this)
        return *this;

    delete pb;
    pb = new Bitmap(*rhs.pb);
    return this;
}
```

然而，这个版本不具备“异常安全性”，如果`new Bitmap`出现异常，pb仍会指向一块被删除的内存。

解决方法是复制pb所指东西时别删除pb：

```cpp
Widget& Widget::operator=(const Widget& rhs){
    Bitmap* tmp = pb;
    pb = new Bitmap(*rhs.pb);
    delete tmp;
    return *this;
}
```

如果担心效率，可以把证同测试放到函数起始处，但是证同测试同样需要成本，所以需要估计自我赋值发生的频率有多高。

另一种方法是使用copy and swap技术：

```cpp
class Widget {
    ...
    void swap(Widget& rhs); //交换*this和rhs的数据
    ...
};

Widget& Widget::operator=(const Widget& rhs){
    Widget temp(rhs); //为rhs数据制作一份副本
    swap(temp); //将*this数据和上述副本的数据交换
    return *this
}
```

另一个方法利用一下两个因素：（1）某class的copy assignment操作符可能被声明为以by value的方式接受实参 （2）以by value的方式传递会自动生成副本。

```cpp
Widget& Widget::operator=(Widget rhs) //注意这里是pass by value
{
    swap(rhs);
    return *this;
}
```

这种方法为了巧妙牺牲了清晰性，然而，将coping动作从函数本体内移到“函数参数构造阶段”又是可令编译器生成更高效的代码。

## 12、复制对象时勿忘其每一个成分

如果自己生成copying函数，需要注意。

考虑一个class用来表现顾客，其中手写copying函数，外界对它的调用会被日志记下来：

```cpp
void logCall(const std::string& funcName);
class Customer {
public:
    ...
    Customer(const Customer& rhs);
    Costomer& operator=(const Customer& rhs);
    ...
private:
    std::string name;
};

Customer::Customer(const Customer& rhs)
    : name(rhs.name)
{
    logCall("...");
}

Customer& Customer::operator=(const Customer& rhs)
{
    logCall("...");
    name = rhs.name;
    return *this;
}
```

当一个新的变量被添加，它并不会被复制，且也不会被编译器警告。

而一旦发生继承，会有另一个问题：

```cpp
class PriorityCustomer : public Customer {
public:
    ...
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCustomer& rhs)
    ...
private:
    int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer$ rhs)
 : priority(rhs.priority)
{
    ...
}
...
```

上面构造函数只复制了子类的成员变量，但base class的成员变量均没有复制，这些变量会被Customer的默认构造函数赋值。

所以，应该让divided class调用相应base class的函数：

```cpp
PriorityCustomer::PriorityCustomer(const PriorityCustomer$ rhs)
 :  Customer(rhs) //调用base class的构造函数
    priority(rhs.priority)
{
    ...
}

PriorityCustomer& PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    logCall();
    Customer::operator=(rhs); //对base class 成分进行赋值动作
    priority = rhs.priority;
    return *this;
}
```

当你编写一个copying函数，请确保：

1. 复制所有local成员变量
2. 调用所有base class的适当copying函数

不应该令copy assignment操作符调用copy构造函数。也不应该令copy构造函数调用copy assignment操作符。

当copy构造函数和copy assignment操作符有相同的代码，通常的做法是建立一个新的成员函数给两者调用。这样的函数往往是private的且常被命名为init。

* **copying函数应该确保复制对象内所有成员变量及所有base class成分。**
* **不要尝试用某个copying函数实现另一个copying函数。应该将共同机能放在第三个函数中，并由两个copying函数共同调用。**
