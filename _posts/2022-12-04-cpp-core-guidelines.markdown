---
layout: post
title:  "Cpp Core Guidelines"
date: 2022-12-04 14:00:00 +0800
categories: jekyll update
---

- [I:接口](#i接口)
- [R:资源管理](#r资源管理)
- [Per:性能](#per性能)
- [CP:并发和并行](#cp并发和并行)

<br/>
## **I:接口**

接口规则摘要：
- [I.1:显式明确接口](#i1显式明确接口)
- [I.2:避免使用非const全局变量](#i2避免使用非const全局变量)
- [I.3:避免使用单例](#i3避免使用单例)
- [I.4:接口设计精确且强类型化](#i4接口设计精确且强类型化)
- [I.5:说明先决条件（如果有的话）](#i5说明先决条件如果有的话)
- [I.6:优先使用Expects来表达先决条件](#i6优先使用expects来表达先决条件)
- [I.7:说明后置条件](#i7说明后置条件)
- [I.8:优先使用Ensures来表达后置条件](#i8优先使用ensures来表达后置条件)
- [I.9:如果一个接口是模版，那么用concepts来记录它的参数](#i9如果一个接口是模版那么用concepts来记录它的参数)
- [I.10:用异常来表示未能执行所需任务](#i10用异常来表示未能执行所需任务)
- [I.11:不要通过裸指针T\*或T&来转移所有权](#i11不要通过裸指针t或t来转移所有权)
- [I.12:将一个不为空的指针声明为not\_null](#i12将一个不为空的指针声明为not_null)
- [I.13:不要把数组作为一个单一的指针来传递](#i13不要把数组作为一个单一的指针来传递)
- [I.22:避免全局对象的复杂初始化](#i22避免全局对象的复杂初始化)
- [I.23:函数参数个数尽量少](#i23函数参数个数尽量少)
- [I.24:避免相邻的参数可以由相同的参数以任意顺序调用，但意义不同](#i24避免相邻的参数可以由相同的参数以任意顺序调用但意义不同)
- [I.25:优先使用空的抽象类作为类层次的接口](#i25优先使用空的抽象类作为类层次的接口)
- [I.26:如果你想要一个跨编译器的ABI，请使用C风格子集](#i26如果你想要一个跨编译器的abi请使用c风格子集)
- [I.27:为了稳定的库ABI，考虑用Pimpl](#i27为了稳定的库abi考虑用pimpl)
- [I.30:封装反规则的行为](#i30封装反规则的行为)

**还有：**
- [F:函数](#f函数)
- [C.concrete:实体类型](#cconcrete实体类型)
- [C.hier:类分层](#chier类分层)
- [C.over:运算符重载](#cover运算符重载)
- [C.con:容器及其他资源句柄](#ccon容器及其他资源句柄)
- [E:错误处理](#e错误处理)
- [T:模版和元编程](#t模版和元编程)

<br/>
### **I.1:显式明确接口**

**原因** 正确性。接口中没有明确的假设很容易被忽视，也很难测试。

**Example,bad** 通过全局（命名空间范围）变量（调用模式）来控制一个函数的行为是隐式的，且造成混乱。
```cpp
int round(double d)
{
    return (round_up) ? ceil(d) : d;    // don't: "invisible" dependency
}
```
对于调用者，两次调用round(7.2)可能会得到不同的结果。这一点并不明显。

**Example,bad** 通过非局部变量(errno)的报告很容易被忽略。
```cpp
// don't: no test of printf's return value
fprintf(connection, "logging: %d %d %d\n", x, y, s);
```

<br/>
### **I.2:避免使用非const全局变量**

**原因** 非const全局变量隐藏了依赖关系，并使依赖关系受到不可预测变化的影响。

**Example**
```cpp
struct Data {
    // ... lots of stuff ...
} data;            // non-const data

void compute()     // don't
{
    // ... use data ...
}

void output()     // don't
{
    // ... use data ...
}
```
还有谁可能修改了数据？

**注意** 全局对象的初始化不是有序的。如果你使用一个全局对象，就用一个常量来初始化它。注意，即使是一个常量对象，也可能是一个为定义的初始化顺序。

**例外** 一个全局对象往往比单例更好。

**注意** 全局常量是有用的。

**替换** 如果你使用全局数据来避免复制，可以考虑将数据用const引用传递。另一个解决方案是将数据定义为一个对象的状态，将操作定义为成员函数。

**警告** 注意data races。如果一个线程可以访问非局部数据（或通过引用传递），而另外一个线程执行calle，那么就会出现data races。每个指针或可变数据的引用都是一个潜在的data race。

**Enforcement** 所有在命名空间的非const变量以及指向非const数据的全局指针或全局引用都应该被标记出来。

<br/>
### **I.3:避免使用单例**

**原因** 单例是伪装的复杂全局对象

**Example**
```cpp
class Singleton {
    // ... lots of stuff to ensure that only one Singleton object is created,
    // that it is initialized properly, etc.
};
```

**注意** 如果你不希望全局对象发生变化，那么就声明为const或constexpr。

**例外** 可以使用最简单的单例模式，在第一次使用的时候获得初始化，如果有的话。
```cpp
X& myX()
{
    static X my_x {3};
    return my_x;
}
```
这是解决与初始化顺序相关问题最有效的办法之一。在多线程环境中，静态对象的初始化不会引入data race，除非你在构造函数中访问一个共享对象。
注意局部静态对象的初始化不会有data race，然而如果x的销毁涉及到需要同步的操作，我们必须使用一个不太简单的解决方案。
```cpp
X& myX()
{
    static auto p = new X {3};
    return *p;  // potential leak
}
```
现在需要有人负责线程安全地delete对象。这很容易出错，所以我们不使用这种技术，除非：
- myX是支持多线程代码
- X对象需要销毁
- X析构函数需要同步

<br/>
### **I.4:接口设计精确且强类型化**

**原因** 类型是最简单和最好的文档，由于其定义明确的含义，提高了可读性，并在编译时进行检查。另外，精确的类型化代码通常会被更好地优化。

**Example,bad**
```cpp
void pass(void* data);    // weak and under qualified type void* is suspicious
```
调用者不确定接口允许什么类型数据，以及数据是否可被修改，因为没有指定const。所有的指针类型都可以隐式地转换为void\*。

**Example,bad**
```cpp
draw_rect(100, 200, 100, 500); // what do the numbers specify?

draw_rect(p.x, p.y, 10, 20); // what units are 10 and 20 in?
```
使用类型系统：
```cpp
void draw_rectangle(Point top_left, Point bottom_right);
void draw_rectangle(Point top_left, Size height_width);

draw_rectangle(p, Point{10, 20});  // two corners
draw_rectangle(p, Size{10, 20});   // one corner and a (height, width) pair
```

**Example,bad**
```cpp
set_settings(true, false, 42); // what do the numbers specify?
```
改为更显式，安全以及可读性
```cpp
alarm_settings s{};
s.enabled = true;
s.displayMode = alarm_settings::mode::spinning_light;
s.frequency = alarm_settings::every_10_seconds;
set_settings(s);
```
对于一组bool情况，可以考虑使用enum，它可以表达一组bool值。
```cpp
enable_lamp_options(lamp_option::on | lamp_option::animate_state_transitions);
```

**Example,bad**
```cpp
time_to_blink means: Seconds? Milliseconds?
void blink_led(int time_to_blink) // bad -- the unit is ambiguous
{
    // ...
    // do something with time_to_blink
    // ...
}

void use()
{
    blink_led(2);
}
```
**Example,good**
```cpp
void blink_led(milliseconds time_to_blink) // good -- the unit is explicit
{
    // ...
    // do something with time_to_blink
    // ...
}

void use()
{
    blink_led(1500ms);
}
```
还有一种可以接受更多时间单位：
```cpp
template<class rep, class period>
void blink_led(duration<rep, period> time_to_blink) // good -- accepts any unit
{
    // assuming that millisecond is the smallest relevant unit
    auto milliseconds_to_blink = duration_cast<milliseconds>(time_to_blink);
    // ...
    // do something with milliseconds_to_blink
    // ...
}

void use()
{
    blink_led(2s);
    blink_led(1500ms);
}
```

<br/>
### **I.5:说明先决条件（如果有的话）**

**原因** 参数的意义可能会制约它们在被调用者中的正确使用。

**Example**
```cpp
double sqrt(double x);
```
x应该是大等于0的。类型系统无法表达这个，所以可以人为加个注释。
```cpp
double sqrt(double x); // x must be non-negative
```
也可以加个assertions
```cpp
double sqrt(double x) { Expects(x >= 0); /* ... */ }
```
理论上Expects应该属于接口的一部分，但是这很难做到。那么我们把它放入定义（函数体）。

**注意** 优先使用正式的需求说明。比如Expects，如果这不可行，那么可以使用注释，比如// [p:q)表达一个有序队列，用<排序。

<br/>
### **I.6:优先使用Expects来表达先决条件**

**原因** 明确该条件是个前提条件，且工具得以使用。

**Example**
```cpp
int area(int height, int width)
{
    Expects(height > 0 && width > 0);            // good
    if (height <= 0 || width <= 0) my_error();   // obscure
    // ...
}
```

**注意** 前提条件可以用很多方式来表述，包括注释、if语句和assert()。这可能会使它们与普通代码难以区分，难以更新，难以被工具操作，而且可能会有错误的语义（你总是想在调试模式下中止，而在产品运行中什么都不检查吗）。

**注意** 前提条件应该是接口的一部分，而不是实现的一部分，但我们还没有语言设施来做到这一点。一旦语言支持变得可用，我们将采用标准版本的前提条件、后置条件和断言。

**注意** Expects()也可以用来检测算法中的一个条件。

<br/>
### **I.7:说明后置条件**

**原因** 检测对结果的误解，并指出错误的实现。

**Example,bad**
```cpp
int area(int height, int width) { return height * width; }  // bad
```
这里我们没有显式限定前置条件，比如宽和高都应该大于0。我们也忽略了后置条件限定，比如相乘的面积可能超过int最大值造成溢出。
```cpp
int area(int height, int width)
{
    auto res = height * width;
    Ensures(res > 0);
    return res;
}
```

**Example,bad**
```cpp
void f()    // problematic
{
    char buffer[MAX];
    // ...
    memset(buffer, 0, sizeof(buffer));
}
```
这里没有使用后置条件判断，造成优化器认定memset是多余的，那memset就会被优化掉。
改为：
```cpp
void f()    // better
{
    char buffer[MAX];
    // ...
    memset(buffer, 0, sizeof(buffer));
    Ensures(buffer[0] == 0);
}
```

**注意** 后置条件通常在非正式的注释中出现。使用Ensures可以更系统，可见以及可检查性。

**Example**
```cpp
mutex m;

void manipulate(Record& r)    // don't
{
    m.lock();
    // ... no m.unlock() ...
}
```
这里我们忽略了说明mutex应该被释放，直接加个后置条件声明会更清晰。
```cpp
void manipulate(Record& r)    // postcondition: m is unlocked upon exit
{
    m.lock();
    // ... no m.unlock() ...
}
```
最好使用RAII:
```cpp
void manipulate(Record& r)    // best
{
    lock_guard<mutex> _ {m};
    // ...
}
```

**注意** 理想情况下，后置条件要在接口/声明中说明，以便用户容易看到他们。只有与用户相关的后置条件可以在接口中说明。\
至于内部状态相关的后置条件在定义/实现中。

<br/>
### **I.8:优先使用Ensures来表达后置条件**

**原因** 明确该条件是后置条件，且工具得以使用。

**Example**
```cpp
void f()
{
    char buffer[MAX];
    // ...
    memset(buffer, 0, MAX);
    Ensures(buffer[0] == 0);
}
```

<br/>
### **I.9:如果一个接口是模版，那么用concepts来记录它的参数**

**原因** 接口更精确地约定，以及编译期间就可以检查。

**Example**
```cpp
template<typename Iter, typename Val>
  requires input_iterator<Iter> && equality_comparable_with<iter_value_t<Iter>, Val>
Iter find(Iter first, Iter last, Val v)
{
    // ...
}
```

<br/>
### **I.10:用异常来表示未能执行所需任务**

**原因** 不应该忽略错误，这会导致系统或计算处于一个为定义的状态。

**Example,bad**
```cpp
int printf(const char* ...);    // bad: return negative number if output fails

template<class F, class ...Args>
// good: throw system_error if unable to start the new thread
explicit thread(F&& f, Args&&... args);
```

**替代方案** 如果你无法使用异常（例如代码中很多老式的裸指针使用），考虑使用返回一对值。
```cpp
int val;
int error_code;
tie(val, error_code) = do_something();
if (error_code) {
    // ... handle the error or exit ...
}
// ... use val ...
```
C++17:
```cpp
auto [val, error_code] = do_something();
if (error_code) {
    // ... handle the error or exit ...
}
// ... use val ...
```

<br/>
### **I.11:不要通过裸指针T*或T&来转移所有权**

**原因** 如果对调用者或者被调用者是否拥有一个对象有疑问，就可能发生泄漏或过早销毁。

**Example**
```cpp
X* compute(args)    // don't
{
    X* res = new X{};
    // ...
    return res;
}
```
谁来负责释放X？如果compute返回一个引用将更难分析，此时可以返回值，如果返回对象比较大，可以使用移动语意。
```cpp
vector<double> compute(args)  // good
{
    vector<double> res(10000);
    // ...
    return res;
}
```

**替代方案** 使用智能指针来传递所有权，unique\_ptr独占所有权，shared\_ptr共享所有权。但直接返回对象本身更优雅，更高效。
只有在需要引用语意的情况下才使用智能指针。

**替代方案** 有时为了ABI兼容，老代码不能被修改，这时候应该使用GSL的owner。
```cpp
owner<X*> compute(args)    // It is now clear that ownership is transferred
{
    owner<X*> res = new X{};
    // ...
    return res;
}
```
这将告诉分析工具res是所有者，返回的值应该被释放或者被转移到其他所有者。

**注意** 每一个作为裸指针被传递的对象都被认为是由调用者拥有所有权，所以它的生命周期也由调用者管理。换个角度看：与指针传递的API相比，
所有权转移的API相当罕见，所以默认是没有所有权转移。

<br/>
### **I.12:将一个不为空的指针声明为not_null**

**原因** 避免解引用nullptr，避免多余的nullptr检查。

**Example**
```cpp
int length(const char* p);            // it is not clear whether length(nullptr) is valid

length(nullptr);                      // OK?

int length(not_null<const char*> p);  // better: we can assume that p cannot be nullptr

int length(const char* p);            // we must assume that p can be nullptr
```
通过在源码中说明意图，实现者和工具可以提供更好的诊断，例如通过静态分析来发现某些类别的错误，并进行优化，例如删除分支和判空测试。

**注意** char字符指针是隐式的，潜在的混淆和错误来源，优先使用zstring，而不是const char\*。
```cpp
// we can assume that p cannot be nullptr
// we can assume that p points to a zero-terminated array of characters
int length(not_null<zstring> p);
```

<br/>
### **I.13:不要把数组作为一个单一的指针来传递**

**原因** 指针风格的接口易错。一个指向数组的指针必须依靠一些其他数据来确定被调用者大小。

**Example**
```cpp
void copy_n(const T* p, T* q, int n); // copy from [p:p+n) to [q:q+n)
```
如果q指向的数组元素个数少于n？如果p指向的数组元素个数少于n?

**替代方案**
```cpp
void copy(span<const T> r, span<T> r2); // copy r to r2
```

另外一个例子：
**Example,bad**
```cpp
Example, bad Consider:
void draw(Shape* p, int n);  // poor interface; poor code
Circle arr[10];
// ...
draw(arr, 10);
```
传10可能是个错误，因为常见的范围是[0,10)。draw无法安全地遍历数组，因为它没法知道元素大小。Circle隐式退化成Shape。

**替代方案**
```cpp
void draw2(span<Circle>);
Circle arr[10];
// ...
draw2(span<Circle>(arr));  // deduce the number of elements
draw2(arr);    // deduce the element type and array size

void draw3(span<Shape>);
draw3(arr);    // error: cannot convert Circle[10] to span<Shape>
```
draw2传入的信息跟draw一样多，而且更显式列出需要一个Circle的range。

**例外** 使用zstring和czstring来表示C风格的，以零结尾的字符串。但是当你这样做的时候，请使用std::string\_view或GSL中的span<char>来防止range错误。

<br/>
### **I.22:避免全局对象的复杂初始化**

**原因** 复杂初始化会导致执行顺序的不确定性。

**Example**
```cpp
// file1.c

extern const X x;

const Y y = f(x);   // read x; write y

// file2.c

extern const Y y;

const X x = g(y);   // read y; write x
```
x和y在不同的翻译单元，f()和g()的调用顺序是未定义的。可能会访问一个未初始化的const。

**注意** 初始化顺序在并发代码中特别难以处理。通常最好避免全局对象（命名范围空间内）。

**Enforcement**
- 调用非const的全局变量初始化应该标记出
- 访问extern的全局变量初始化也应该标记出

<br/>
### **I.23:函数参数个数尽量少**

**原因** 多参数容易混淆，开销也更大。

**讨论** 两个常见原因：
- 缺少抽象。
- 违反one function，one responsibility。

**Example** 
```cpp
template<class InputIterator1, class InputIterator2, class OutputIterator, class Compare>
OutputIterator merge(InputIterator1 first1, InputIterator1 last1,
                     InputIterator2 first2, InputIterator2 last2,
                     OutputIterator result, Compare comp);
```
缺少抽象，STL没有传递一个range，而是传递了迭代器pair。这里我们有4个模版参数和6个函数参数。
首先可以简化的是使用默认比较符<。
```cpp
emplate<class InputIterator1, class InputIterator2, class OutputIterator>
OutputIterator merge(InputIterator1 first1, InputIterator1 last1,
                     InputIterator2 first2, InputIterator2 last2,
                     OutputIterator result);
```
这并没有减少总的复杂度，为了真正减少参数数量，我们需要将参数更高抽象。
```cpp
template<class In1, class In2, class Out>
  requires mergeable<In1, In2, Out>
Out merge(In1 r1, In2 r2, Out result);
```
再举个例子：
```cpp
void f(int* some_ints, int some_ints_length);  // BAD: C style, unsafe
```
改为：
```cpp
void f(gsl::span<int> some_ints);              // GOOD: safe, bounds-checked
```
这种抽象更安全，更健壮，而且自然地减少参数个数。

**注意** 函数参数最好低于4个。

**替代方案** 使用更高的抽象。将参数分组为有意义的分组，并传递对象（传值或传引用）。

**替代方案** 使用默认参数或重载，允许用最少的参数完成最常用形式的调用。

**Enforcement** 如果一个函数声明有两个相同类型的迭代器，而不是range或view，那应当发出警告。

<br/>
### **I.24:避免相邻的参数可以由相同的参数以任意顺序调用，但意义不同**

**原因** 同一类型的相邻参数容易被错误调换。

**Example,bad**
```cpp
void copy_n(T* p, T* q, int n);  // copy from [p:p + n) to [q:q + n)
```
这是个糟糕的K&R式接口的变种，它很容易把from和to参数颠倒。
把from改为const:
```cpp
void copy_n(const T* p, T* q, int n);  // copy from [p:p + n) to [q:q + n)
```
**例外** 如果函数参数顺序无所谓：
```cpp
int max(int a, int b);
```

**替代方案** 不要传数组给参数指针，应该传一个对象来表示范围。
```cpp
void copy_n(span<const T> p, span<T> q);  // copy from p to q
```

**替代方案** 定义一个struct，并把参数字段定义为成员。
```cpp
struct SystemParams {
    string config_file;
    string output_path;
    seconds timeout;
};
void initialize(SystemParams p);
```
这会让未来的读者更清晰的看到这些调用，因为参数往往在调用栈中按名字填写。

<br/>
### **I.25:优先使用空的抽象类作为类层次的接口**

**原因** 空的抽象类（没有非静态成员数据）比有状态的基类更稳定。

**Example,bad**
```cpp
class Shape {  // bad: interface class loaded with data
public:
    Point center() const { return c; }
    virtual void draw() const;
    virtual void rotate(int);
    // ...
private:
    Point c;
    vector<Point> outline;
    Color col;
};
```
每一个派生类都要计算center，即使是non-trivial的，而center从未被使用。同样的，不是每一个shape都有color，许多shapes最好不要用定义为points的序列轮廓来表示。\
使用一个抽象类会更好。
```cpp
class Shape {    // better: Shape is a pure interface
public:
    virtual Point center() const = 0;   // pure virtual functions
    virtual void draw() const = 0;
    virtual void rotate(int) = 0;
    // ...
    // ... no data members ...
    // ...
    virtual ~Shape() = default;
};
```

<br/>
### **I.26:如果你想要一个跨编译器的ABI，请使用C风格子集**

**原因** 不同编译器实现了类、异常处理、函数名称等的各自不同的二进制布局。

**例外** 一些平台出现了通用的ABI，从而让你从严厉的限制中解放。

**注意** 如果你使用单一的编译器，你可以在接口中使用完整的C++。这在编译器升级的时候需要重新编译。

<br/>
### **I.27:为了稳定的库ABI，考虑用Pimpl**

**原因** 私有数据成员参与了类的布局，私有成员函数参与了重载决议，对这些数据修改都会让使用这些类的用户重新编译代码。
一个持有指向实现的指针的非多态接口类（Pimpl）可以将一个类的用户和它的实现分离，代价是一个间接性。

**Example** 接口(widget.h)
```cpp
class widget {
    class impl;
    std::unique_ptr<impl> pimpl;
public:
    void draw(); // public API that will be forwarded to the implementation
    widget(int); // defined in the implementation file
    ~widget();   // defined in the implementation file, where impl is a complete type
    widget(widget&&); // defined in the implementation file
    widget(const widget&) = delete;
    widget& operator=(widget&&); // defined in the implementation file
    widget& operator=(const widget&) = delete;
};
```
实现(widget.cpp)
```cpp
class widget::impl {
    int n; // private data
public:
    void draw(const widget& w) { /* ... */ }
    impl(int n) : n(n) {}
};
void widget::draw() { pimpl->draw(*this); }
widget::widget(int n) : pimpl{std::make_unique<impl>(n)} {}
widget::widget(widget&&) = default;
widget::~widget() = default;
widget& widget::operator=(widget&&) = default;
```

<br/>
### **I.30:封装反规则的行为**

**Example**
```cpp
bool owned;
owner<istream*> inp;
switch (source) {
case std_in:        owned = false; inp = &cin;                       break;
case command_line:  owned = true;  inp = new istringstream{argv[2]}; break;
case file:          owned = true;  inp = new ifstream{argv[2]};      break;
}
istream& in = *inp;
```
没有初始化变量，违反所有权规则，而且还必须记得
```cpp
if (owned) delete inp;
```
```cpp
class Istream { [[gsl::suppress(lifetime)]]
public:
    enum Opt { from_line = 1 };
    Istream() { }
    Istream(zstring p) : owned{true}, inp{new ifstream{p}} {}            // read from file
    Istream(zstring p, Opt) : owned{true}, inp{new istringstream{p}} {}  // read from command line
    ~Istream() { if (owned) delete inp; }
    operator istream&() { return *inp; }
private:
    bool owned = false;
    istream* inp = &cin;
};
```

<br/>
## **R:资源管理**

资源一般需要获取和释放，比如内存，文件句柄，套接字，锁等。为什么需要释放的原因是，资源是有限的。如果不释放甚至延迟释放都可能造成问题。  
本节目的：如何不造成内存泄漏，以及避免持有资源时间过长。
<br/>
- 资源管理规则摘要
	- [R.1:使用资源句柄和RAII来管理资源](#r1使用资源句柄和raii来管理资源)
	- [R.2:接口使用裸指针来代表单个对象](#r2接口使用裸指针来代表单个对象)
	- [R.3:T\*不拥有所有权](#r3t不拥有所有权)
	- [R.4:T&不拥有所有权](#r4t不拥有所有权)
	- [R.5:优先使用scoped对象，非必要不使用堆分配](#r5优先使用scoped对象非必要不使用堆分配)
	- [R.6:避免使用非const全局变量](#r6避免使用非const全局变量)
- 分配释放规则摘要
	- [R.10:避免使用malloc()和free()](#r10避免使用malloc和free)
	- [R.11:避免显式调用new和delete](#r11避免显式调用new和delete)
	- [R.12:资源的分配应立即丢给一个可管理对象](#r12资源的分配应立即丢给一个可管理对象)
	- [R.13:在一个表达式中最多执行一次显式资源分配](#r13在一个表达式中最多执行一次显式资源分配)
	- [R.14:避免使用[]，用span](#r14避免使用用span)
	- [R.15:重载allocation/deallocation要配套](#r15重载allocationdeallocation要配套)
- 智能指针规则摘要
	- [R.20:用unique\_ptr和shared\_ptr来表达所有权](#r20用unique_ptr和shared_ptr来表达所有权)
	- [R.21:优先使用unique\_ptr，除非要共享所有权才用shared\_ptr](#r21优先使用unique_ptr除非要共享所有权才用shared_ptr)
	- [R.22:使用make\_shared来获取shared\_ptr](#r22使用make_shared来获取shared_ptr)
	- [R.23:使用make\_unique来获取unique\_ptr](#r23使用make_unique来获取unique_ptr)
	- [R.24:使用weak\_ptr来避免shared\_ptr循环引用](#r24使用weak_ptr来避免shared_ptr循环引用)
	- [R.30:原则上用T\*或者T&，而不是用smart pointer来传递参数](#r30只有在需要表达生命期语意前提下才用智能指针传递参数)
	- [R.32:使用unique\_ptr\<Widget\>来表达一个函数对Widget的所有权](#r32使用unique_ptrwidget来表达一个函数对widget的所有权)
	- [R.33:使用unique\_ptr\<Widget\>&参数来表达函数reseats Widget](#r33使用unique_ptrwidget参数来表达函数reseats-widget)
	- [R.34:使用shared\_ptr\<T\>参数来表达共享所有权](#r34使用shared_ptrt参数来表达共享所有权)
	- [R.35:shared\_ptr\<T\>&参数用来表达可能reseat这个shared\_ptr](#r35shared_ptrt参数用来表达可能reseat这个shared_ptr)
	- [R.36:传递const shared\_ptr\<T\>&来表示有可能保留对象引用计数](#r36传递const-shared_ptrt来表示有可能保留对象引用计数)
	- [R.37:不要传参一个来自aliased的smart pointer的指针或引用](#r37不要传参来自aliased的smart-pointer的指针或引用)

<br/>
### **R.1:使用资源句柄和RAII来管理资源**

**原因** 为了避免泄漏以及简化管理资源，C++语言层面的构造、析构函数刚好对应资源获取、释放，比如fopen/fclose，lock/unlock，new/delete。\
每当你要处理资源，比如获取、释放资源的函数调用时，应该把它封装到一个对象中，构造函数获取资源，析构函数释放资源。

**Example, bad**
```cpp
void send(X* x, string_view destination)
{
    auto port = open_port(destination);
    my_mutex.lock();
    // ...
    send(port, x);
    // ...
    my_mutex.unlock();
    close_port(port);
    delete x;
}
```
在以上代码中，你必须记得要unlock，close\_port，delete，不管任何一条路径都要处理到，而且，一旦在 ... 中抛出异常，x就会泄漏，my\_mutex就还会锁住。

**Example**
```cpp
void send(unique_ptr<X> x, string_view destination)  // x owns the X
{
    Port port{destination};            // port owns the PortHandle
    lock_guard<mutex> guard{my_mutex}; // guard owns the lock
    // ...
    send(port, x);
    // ...
} // automatically unlocks my_mutex and deletes the pointer in x
```
现在所有的资源都自动清理，所有的路径上都执行一次，无法是否有异常，不仅如此，函数签名也表示它接管了指针所有权。
那Port是如何封装的？
```cpp
class Port {
    PortHandle port;
public:
    Port(string_view destination) : port{open_port(destination)} { }
    ~Port() { close_port(port); }
    operator PortHandle() { return port; }

    // port handles can't usually be cloned, so disable copying and assignment if necessary
    Port(const Port&) = delete;
    Port& operator=(const Port&) = delete;
};
```

<br/>
### **R.2:接口使用裸指针来代表单个对象**

**原因** 数组最好由一个容器来表示，vector表示拥有所有权，span表示非所有权，它们拥有足够的信息来做范围检查。

**Example, bad**
```cpp
void f(int* p, int n)   // n is the number of elements in p[]
{
    // ...
    p[2] = 7;   // bad: subscript raw pointer
    // ...
}
```
编译器不会读注释，如果没有其他代码，那就不知道p到底拥有几个元素。建议用span替换。

**Example**
```cpp
void g(int* p, int fmt)   // print *p using format #fmt
{
    // ... uses *p and p[0] only ...
}
```

例外：C-style一般传递以0结尾的字符串指针，使用zstring而不是char\*，来表示依赖这个约定。
注意：当前很多用指向单元素的指针都可以用引用，但nullptr可以表示一个合法的指针，引用不行。

**Enforcement** 

* 所有非container,view,iterator的指针计算（包括++），都应标记指出，但是这对老代码会产生巨多false positives。
* 所有数组名被当作单一指针传递都应该被标记。

<br/>
### **R.3:T\*不拥有所有权**

**原因** 如果是拥有所有权的指针都会被标识出来，这样就可以可靠且有效的删除所指向对象。

**Example**
```cpp
void f()
{
    int* p1 = new int{7};           // bad: raw owning pointer
    auto p2 = make_unique<int>(7);  // OK: the int is owned by a unique pointer
    // ...
}
```
unique\_ptr保证资源会析构，不会泄漏，即使抛出异常。T\*不会。

**Example**
```cpp
template<typename T>
class X {
public:
    T* p;   // bad: it is unclear whether p is owning or not
    T* q;   // bad: it is unclear whether q is owning or not
    // ...
};
```
优化
```cpp
template<typename T>
class X2 {
public:
    owner<T*> p;  // OK: p is owning
    T* q;         // OK: q is not owning
    // ...
};
```

**Example, bad**
```cpp
Gadget* make_gadget(int n)
{
    auto p = new Gadget{n};
    // ...
    return p;
}

void caller(int n)
{
    auto p = make_gadget(n);   // remember to delete p
    // ...
    delete p;
}
```
返回一个指针意味着调用者不确定这个指针的生命期，谁来负责删除所指向对象呢？
如果Gadget move函数开销不大，比如该对象很小，或者移动该对象很高效，那么直接返回value
就可以。
```cpp
Gadget make_gadget(int n)
{
    Gadget g{n};
    // ...
    return g;
}
```
注意：本规则适用于factory模式。如果确实需要返回指针，比如指向子类的基类指针，那么应该返回smart pointer。

**Enforcement**

- delete一个非所有权指针要给出警告
- 在每条代码路径上都没能reset或者显式delete都应该给出警告
- new出来的对象被一个裸指针指向给出警告
- 函数内构造的对象被当作函数返回值，并且该对象有移动构造，那么应该建议直接返回value。

<br/>
### **R.4:T&不拥有所有权**

**Example**
```cpp
void f()
{
    int& r = *new int{7};  // bad: raw owning reference
    // ...
    delete &r;             // bad: violated the rule against deleting raw pointers
}
```

<br/>
### **R.5:优先使用scoped对象，非必要不使用堆分配**

**原因** 一个scoped对象指的是一个局部、全局对象或者成员，这意味着不用再有额外的allocation和deallocation，scoped对象的成员也是个scoped对象，scoped对象的生命期管理由构造、析构函数管理。

**Example**
以下代码不够高效，因为由非必要的分配和释放内存操作，而且一旦在...部分抛出异常，会造成内存泄漏，而且代码也不简洁。
```cpp
void f(int n)
{
    auto p = new Gadget{n};
    // ...
    delete p;
}
```
Instead
```cpp
void f(int n)
{
    Gadget g{n};
    // ...
}
```

**Enforcement**

- 在函数内，如果对象被分配，然后在各个代码路径上被释放，那么首选stack object。
- 如果unique_ptr对象没有被移动，shared_ptr没有被拷贝，reassigned or reset，那么应当发出警告。

<br/>
### **R.6:避免使用非const全局变量**

**Example**
```cpp
struct Data {
    // ... lots of stuff ...
} data;            // non-const data

void compute()     // don't
{
    // ... use data ...
}

void output()     // don't
{
    // ... use data ...
}
```
还有谁可以修改data？

一个线程读取全局数据，另一个线程修改这个全局数据，就会data race。


<br/>
### **R.10:避免使用malloc()和free()**

**原因** malloc()和free()不支持构造和析构，跟new和delete也不能混着用。

**Example**
```cpp
class Record {
    int id;
    string name;
    // ...
};

void use()
{
    // p1 might be nullptr
    // *p1 is not initialized; in particular,
    // that string isn't a string, but a string-sized bag of bits
    Record* p1 = static_cast<Record*>(malloc(sizeof(Record)));

    auto p2 = new Record;

    // unless an exception is thrown, *p2 is default initialized
    auto p3 = new(nothrow) Record;
    // p3 might be nullptr; if not, *p3 is default initialized

    // ...

    delete p1;    // error: cannot delete object allocated by malloc()
    free(p2);    // error: cannot free() object allocated by new
}
```

<br/>
### **R.11:避免显式调用new和delete**

<br/>
### **R.12:资源的分配应立即丢给一个可管理对象**

**原因** 不这样做，一个异常或返回都会导致泄漏。

**Example,bad**
```cpp
void func(const string& name)
{
    FILE* f = fopen(name, "r");            // open the file
    vector<char> buf(1024);
    auto _ = finally([f] { fclose(f); });  // remember to close the file
    // ...
}
```
**Example**
```cpp
void func(const string& name)
{
    ifstream f{name};   // open the file
    vector<char> buf(1024);
    // ...
}
```

<br/>
### **R.13:在一个表达式中最多执行一次显式资源分配**

**原因** 如果在一个表达式中有两次以上资源分配，因为子表达式的计算顺序比如函数参数，会导致内存泄漏。

**Example**
```cpp
void fun(shared_ptr<Widget> sp1, shared_ptr<Widget> sp2);
```
```cpp
// BAD: potential leak
fun(shared_ptr<Widget>(new Widget(a, b)), shared_ptr<Widget>(new Widget(c, d)));
```
编译器会重组这两个表达式，它不是异常安全的，有可能编译器先分配两段内存，然后再执行Widget构造函数，
如果其中一个构造函数抛出异常，那么就会内存泄漏。
一个简化的解决方案就是不要在一个表达式中有超过一次显式的资源分配。
```cpp
shared_ptr<Widget> sp1(new Widget(a, b)); // Better, but messy
fun(sp1, new Widget(c, d));
```
```cpp
fun(make_shared<Widget>(a, b), make_shared<Widget>(c, d)); // Best
```

<br/>
### **R.14:避免使用[]，用span**

**原因** 数组会退化成指针，因此会丢失其size，造成range错误，使用span，它包含size信息。

**Example**
```cpp
void f(int[]);          // not recommended

void f(int*);           // not recommended for multiple objects
                        // (a pointer should point to a single object, do not subscript)

void f(gsl::span<int>); // good, recommended
```

<br/>
### **R.15:重载allocation/deallocation要配套**

**Example**
```cpp
class X {
    // ...
    void* operator new(size_t s);
    void operator delete(void*);
    // ...
};
```
注意：如果希望内存不被释放，释放操作应该=delete。

<br/>
### **R.20:用unique\_ptr和shared\_ptr来表达所有权**

**原因** 避免资源泄漏

**Example**
```cpp
void f()
{
    X x;
    X* p1 { new X };              // see also ???
    unique_ptr<X> p2 { new X };   // unique ownership; see also ???
    shared_ptr<X> p3 { new X };   // shared ownership; see also ???
    auto p4 = make_unique<X>();   // unique_ownership, preferable to the explicit use "new"
    auto p5 = make_shared<X>();   // shared ownership, preferable to the explicit use "new"
}
```

<br/>
### **R.21:优先使用unique\_ptr，除非要共享所有权才用shared\_ptr**

**Example,bad**
```cpp
void f()
{
    shared_ptr<Base> base = make_shared<Derived>();
    // use base locally, without copying it -- refcount never exceeds 1
} // destroy base
```
**Example**
```cpp
void f()
{
    unique_ptr<Base> base = make_unique<Derived>();
    // use base locally
} // destroy base
```

**Enforcement**

- 如果函数内使用shared\_ptr分配对象，但函数没有返回shared\_ptr，而且没有把shared\_ptr传递给其他函数，那么建议用unique\_ptr替换。


<br/>
### **R.22:使用make\_shared来获取shared\_ptr**

**Example**
```cpp
shared_ptr<X> p1 { new X{2} }; // bad
auto p = make_shared<X>(2);    // good
```

make\_shared更简洁，同时避免了对引用计数的单独allocation，它直接把引用计数放在其对象旁边。


<br/>
### **R.23:使用make\_unique来获取unique\_ptr**

**Example**
```cpp
unique_ptr<Foo> p {new Foo{7}};    // OK: but repetitive

auto q = make_unique<Foo>(7);      // Better: no repetition of Foo
```

简洁，而且在复杂的表达式中更能保证异常安全。


<br/>
### **R.24:使用weak\_ptr来避免shared\_ptr循环引用**

**原因** shared\_ptr依赖引用计数，循环依赖的引用计数永远不会清零。

**Example**
```cpp
#include <memory>

class bar;

class foo {
public:
  explicit foo(const std::shared_ptr<bar>& forward_reference)
    : forward_reference_(forward_reference)
  { }
private:
  std::shared_ptr<bar> forward_reference_;
};

class bar {
public:
  explicit bar(const std::weak_ptr<foo>& back_reference)
    : back_reference_(back_reference)
  { }
  void do_something()
  {
    if (auto shared_back_reference = back_reference_.lock()) {
      // Use *shared_back_reference
    }
  }
private:
  std::weak_ptr<foo> back_reference_;
};
```

<br/>
### **R.30:只有在需要表达生命期语意前提下，才用智能指针传递参数**

见[F.7](#f7一般情况下传参用裸指针或引用而不是智能指针)

**原因** 如果有表达所有权的语意，比如传递或者共享所有权，才使用smart pointer来传递参数。不涉及到生命期修改，应当用裸指针或引用传参。

传递smart pointer限制了函数调用方，一个函数如果需要一个widgt，就应当接受任何widget对象，而不是生命期要智能管理的对象。

传递一个shared\_ptr意味着一个runtime开销。

**Example**
```cpp
// accepts any int*
void f(int*);

// can only accept ints for which you want to transfer ownership
void g(unique_ptr<int>);

// can only accept ints for which you are willing to share ownership
void g(shared_ptr<int>);

// doesn't change ownership, but requires a particular ownership of the caller
void h(const unique_ptr<int>&);

// accepts any int
void h(int&);
```

**Example, bad**
```
// callee
void f(shared_ptr<widget>& w)
{
    // ...
    use(*w); // only use of w -- the lifetime is not used at all
    // ...
};

// caller
shared_ptr<widget> my_widget = /* ... */;
f(my_widget);

widget stack_widget;
f(stack_widget); // error
```

**Example, good**
```cpp
// callee
void f(widget& w)
{
    // ...
    use(w);
    // ...
};

// caller
shared_ptr<widget> my_widget = /* ... */;
f(*my_widget);

widget stack_widget;
f(stack_widget); // ok -- now this works
```

函数参数自然是在函数调用生命周期内，因此极少有生命周期问题。而且dangling指针可以在静态检查检出。

**Enforement**

- 如果一个函数需要一个智能指针(重载->或\*)参数，该智能指针是可拷贝的，但函数只调用(\*,->或get())其中之一，那建议只传递T*或者T&。
- 传递一个智能指针(重载->或\*)当参数，该智能指针是copyable/movable，但是在函数调用中没有用到拷贝/移动，而且也不修改智能指针所指向数据，也不会被传递给另一个可以这样做的函数。这意味着所有权语意没有用到，那么应当被标记改为传递T*或者T&。


<br/>
### **R.32:使用unique_ptr\<Widget\>来表达一个函数对Widget的所有权**

**原因** 以这种方式使用unique\_ptr，既记录又执行了函数调用的所有权转移。 

**Example**
```cpp
void sink(unique_ptr<widget>); // takes ownership of the widget

void uses(widget*);            // just uses the widget
```

**Example, bad**
```cpp
void thinko(const unique_ptr<widget>&); // usually not what you want
```

**Enforcement**

- 如果一个函数传入lvalue左值unique_ptr，但是没有赋值或reset()，那么应该考虑换成T*或T&。
- 如果一个函数传入const unique_ptr<T>&，那应该考虑换成const T*或const T&。

<br/>
### **R.33:使用unique_ptr\<Widget\>&参数来表达函数reseats Widget**

**原因** 使用这种方式，即记录又执行了函数调用的reseat作用。

**Example**
```cpp
void reseat(unique_ptr<widget>&); // "will" or "might" reseat pointer
```

**Example, bad**
```cpp
void thinko(const unique_ptr<widget>&); // usually not what you want
```

**Enforcement**

- 如果传入左值引用unique_ptr<T>当函数参数，但是函数内没有调用赋值或reset，建议用T*或者T&替换。
- 如果一个函数传入const unique_ptr<T>&，建议用const T*或者const T&替换。

<br/>
### __R.34:使用shared_ptr\<T\>参数来表达共享所有权__

**原因** 显式表达共享所有权。

**Example, Good**
```cpp
class WidgetUser
{
public:
    // WidgetUser will share ownership of the widget
    explicit WidgetUser(std::shared_ptr<widget> w) noexcept:
        m_widget{std::move(w)} {}
    // ...
private:
    std::shared_ptr<widget> m_widget;
};
```

**Enforcement**

- 如果函数传递左值shared_ptr<T>参数，但是调用函数内没有对它赋值或reset()，建议用T*或T&替换。
- 如果函数传递const shared_ptr<T>或const shared_ptr<T>&，但是没有任何拷贝或者移动给另外一个shared_ptr，建议用T*或T&替换。
- 如果函数传递shared_ptr<T>右值，那么建议直接传值T。

<br/>
### **R.35:shared_ptr\<T\>&参数用来表达可能reseat这个shared_ptr**

**原因** 显式表达函数可能reseat，reseat表示一个引用或一个智能指针指向一个不同的对象。

**Example, good**
```cpp
void ChangeWidget(std::shared_ptr<widget>& w)
{
    // This will change the callers widget
    w = std::make_shared<widget>(widget{});
}
```

**Enforcement**

- 如果一个函数传入左值引用shared\_ptr<T>，但是函数内没有赋值或reset()，建议用T*或T&替换。
- 如果一个函数传入const shared_ptr<T>，或const shared_ptr<T>&，但是没有任何拷贝或者移动给另外一个shared_ptr<T>，建议用T*或T&替换。
- 如果传入一个右值引用shared_ptr<T>，那么建议直接传递T。

<br/>
### **R.36:传递const shared_ptr\<T\>&来表示有可能保留对象引用计数**


**Example, good**
```cpp
void share(shared_ptr<widget>);            // share -- "will" retain refcount

void reseat(shared_ptr<widget>&);          // "might" reseat ptr

void may_share(const shared_ptr<widget>&); // "might" retain refcount
```

<br/>
### **R.37:不要传参来自aliased的smart pointer的指针或引用**

**原因** 造成引用计数丢失或者悬空指针最大的一个原因就是违反本条规则，函数调用传入指针或者引用应该在同个调用链上，\
在调用链的顶端应该从一个智能指针获得裸指针或引用，并保持对象存在，保证智能指针在整个调用链上没有reset或者被重新赋值。

注意：有时候会先本地获取一个智能指针的拷贝，然后保持对象存活，直到整个调用链。

**Example**
```cpp
// global (static or heap), or aliased local ...
shared_ptr<widget> g_p = ...;

void f(widget& w)
{
    g();
    use(w);  // A
}

void g()
{
    g_p = ...; // oops, if this was the last shared_ptr to that widget, destroys the widget
}
```
以下调用过不了code review
```cpp
void my_code()
{
    // BAD: passing pointer or reference obtained from a non-local smart pointer
    //      that could be inadvertently reset somewhere inside f or its callees
    f(*g_p);

    // BAD: same reason, just passing it as a "this" pointer
    g_p->func();
}
```
修复以上bug，可以先在开头获得一个本地拷贝，这样可以保持引用计数。
```cpp
void my_code()
{
    // cheap: 1 increment covers this entire function and all the call trees below us
    auto pin = g_p;

    // GOOD: passing pointer or reference obtained from a local unaliased smart pointer
    f(*pin);

    // GOOD: same reason
    pin->func();
}
```

**Enforcement**
- 如果从一个不是本地的或者是有别名的智能指针变量，获取的指针或引用，在函数中被使用，那应该发出警告。
- 如果这个智能指针是shared\_ptr，那应该先获取该shared\_ptr的本地拷贝，然后再获取它的指针或引用。

<br/>
## **Per:性能**
	
- [Per.1:不要无故优化](#per1不要无故优化) 
- [Per.2:不要过早优化](#per2不要过早优化) 
- [Per.3:不要优化非关键部分](#per3不要优化非关键部分) 
- [Per.4:不要认为复杂的代码比简单代码快](#per4不要认为复杂的代码比简单代码快) 
- [Per.5:不要认为low-level代码一定比high-level代码快](#per5不要认为low-level代码一定比high-level代码快) 
- [Per.6:不要在没有测量情况下要求性能](#per6不要在没有测量情况要求性能) 
- [Per.7:设计以实现可优化](#per7设计以实现可优化) 
- [Per.10:依赖静态类型系统](#per10依赖静态类型系统) 
- [Per.11:将计算从运行时移到编译时](#per11将计算从运行时移到编译时) 
- [Per.12:消除多余的别名](#per12消除多余的别名) 
- [Per.13:消除多余的indirections](#per13消除多余的indirections) 
- [Per.14:最大限度减少allocations和deallocations](#per14最大限度减少allocations和deallocations)
- [Per.15:不要在关键分支上allocate](#per15不要在关键分支上allocate)
- [Per.16:使用紧凑数据结构](#per16使用紧凑数据结构)
- [Per.17:首先声明时间关键结构中最常用的成员](#per17首先声明时间关键结构中最常用的成员)
- [Per.18:空间就是时间](#per18空间就是时间)
- [Per.19:可预测地访问内存](#per19可预测地访问内存)
- [Per.30:在关键路径上避免上下文切换](#per30在关键路径上避免上下文切换)


<br/>
### **Per.1:不要无故优化**

<br/>
### **Per.2:不要过早优化**

<br/>
### **Per.3:不要优化非关键部分**

<br/>
### **Per.4:不要认为复杂的代码比简单代码快**

**Example,good**
```cpp
// clear expression of intent, fast execution

vector<uint8_t> v(100000);

for (auto& c : v)
    c = ~c;
```
**Example,bad**
```cpp
// intended to be faster, but is often slower

vector<uint8_t> v(100000);

for (size_t i = 0; i < v.size(); i += sizeof(uint64_t)) {
    uint64_t& quad_word = *reinterpret_cast<uint64_t*>(&v[i]);
    quad_word = ~quad_word;
}
```

<br/>
### **Per.5:不要认为low-level代码一定比high-level代码快**

<br/>
### **Per.6:不要在没有测量情况要求性能**

<br/>
### **Per.7:设计以实现可优化**

**Example**
```cpp
void qsort (void* base, size_t num, size_t size, int (*compar)(const void*, const void*));
```
什么时候想对内存进行排序？实际上，我们对元素的序列进行排序，通常存储在容器内。对qsort的调用丢掉了很多有用的信息（例如，元素类型），迫使用户重复已知的信息（例如，元素大小），迫使
用户写额外代码（例如比较两个double函数）。这意味着增加了程序员的工作，容易出错，并使编译器失去了优化所需的信息。
```cpp
double data[100];
// ... fill a ...

// 100 chunks of memory of sizeof(double) starting at
// address data using the order defined by compare_doubles
qsort(data, 100, sizeof(double), compare_doubles);
```
从接口设计角度看，qsort丢失了有用的信息。
以下代码更优(C++98)：
```cpp
template<typename Iter>
    void sort(Iter b, Iter e);  // sort [b:e)

sort(data, data + 100);
```
在这里，我们使用编译器关于数组的大小，元素类型以及如何比较两个double。
C++20:
```cpp
// sortable specifies that c must be a
// random-access sequence of elements comparable with <
void sort(sortable auto& c);

sort(c);
```
这里还有一个小缺点，sort隐含地需要元素间<比较操作。为了完善这个接口，我们需要第二个版本，它接受一个比较标准。
```cpp
// compare elements of c using p
template<random_access_range R, class C> requires sortable<R, C>
void sort(R&& r, C c);
```
标准库的sort提供了这两个版本，甚至更多。

**Example**
```cpp
template<class ForwardIterator, class T>
bool binary_search(ForwardIterator first, ForwardIterator last, const T& val);
```
binary\_search会查找7是否在c中，但不会告诉你7在哪里？或者7有几个？
因此，标准库也提供了：
```cpp
template<class ForwardIterator, class T>
ForwardIterator lower_bound(ForwardIterator first, ForwardIterator last, const T& val);
```
lower\_bound返回了第一个大于val的元素迭代器，如果有的话，没有则指向最后一个迭代器。
因此标准库还提供了：
```cpp
template<class ForwardIterator, class T>
pair<ForwardIterator, ForwardIterator>
equal_range(ForwardIterator first, ForwardIterator last, const T& val);
```
```cpp
auto r = equal_range(begin(c), end(c), 7);
for (auto p = r.first; p != r.second; ++p)
    cout << *p << '\n';
```
显然，这三个接口是由相同的基本代码实现的。它们只是向用户展示了三种二进制搜索算法。从最简单的，到返回完整的，但不一定需要的信息。

**注意**
事情是有成本的。不要对成本过于偏执，现代计算机确实非常快。但是你要对所使用的东西的成本数量级有一个大致的概念。例如，对一次内存访问、一次函数调用、一次字符串比较，一个系统调用、一次磁盘访问和一条网络消息
的成本又一个大致的概念。

<br/>
### **Per.10:依赖静态类型系统**

<br/>
### **Per.11:将计算从运行时移到编译时**

**原因** 减少代码大小以及运行时开销。通过常量来避免data races。在编译期捕捉错误，从而减少错误处理代码。

**Example**
```cpp
double square(double d) { return d*d; }
static double s2 = square(2);    // old-style: dynamic initialization

constexpr double ntimes(double d, int n)   // assume 0 <= n
{
        double m = 1;
        while (n--) m *= d;
        return m;
}
constexpr double s3 {ntimes(2, 3)};  // modern-style: compile-time initialization
```
s2初始化并不罕见，特别是对于square更复杂的初始化，然而相比s3，它有两个问题：
- 运行时函数调用的开销
- s2可能在初始化前就被其他线程访问了

**Example**
```cpp
constexpr int on_stack_max = 20;

template<typename T>
struct Scoped {     // store a T in Scoped
        // ...
    T obj;
};

template<typename T>
struct On_heap {    // store a T on the free store
        // ...
        T* objp;
};

template<typename T>
using Handle = typename std::conditional<(sizeof(T) <= on_stack_max),
                    Scoped<T>,      // first alternative
                    On_heap<T>      // second alternative
               >::type;

void f()
{
    Handle<double> v1;                   // the double goes on the stack
    Handle<std::array<double, 200>> v2;  // the array goes on the free store
    // ...
}
```
在编译期间就是可以计算出是Scoped还是On\_heap。

<br/>
### **Per.12:消除多余的别名**

<br/>
### **Per.13:消除多余的indirections**

<br/>
### **Per.14:最大限度减少allocations和deallocations**

<br/>
### **Per.15:不要在关键分支上allocate**

<br/>
### **Per.16:使用紧凑数据结构**

<br/>
### **Per.17:首先声明时间关键结构中最常用的成员**

<br/>
### **Per.18:空间就是时间**

<br/>
### **Per.19:可预测地访问内存**

**原因**
性能对于缓存非常敏感，而缓存算法更倾向于对相邻数据的简单访问。

**Example**
```cpp
int matrix[rows][cols];

// bad
for (int c = 0; c < cols; ++c)
    for (int r = 0; r < rows; ++r)
        sum += matrix[r][c];

// good
for (int r = 0; r < rows; ++r)
    for (int c = 0; c < cols; ++c)
        sum += matrix[r][c];
```

<br/>
### **Per.30:在关键路径上避免上下文切换**

<br/>
## **CP:并发和并行**

主要阐述使用ISO标准C++设施来表达基本的并发和并行原则和规则。
三大设计目标：
- 帮助编写适合在多线程环境下使用的代码
- 展示标准库提供的线程原语的干净、安全使用
- 如果并发和并行不能提高性能时提供指导

注意的是，C++并发是个未完成的状态，C++11引入了很多核心并发原语，C++14和C++17进一步改进，人们对编写C++并发代码愈发兴趣，我们期望与库相关的向导会随着时间推移而发生重大变化。

- 并发并行原则摘要：
	- [CP.1:写代码时，应假定你的代码会是多线程运行](#cp1假定你的代码会是多线程运行的一部分) 
	- [CP.2:避免数据争用](#cp2避免数据争用) 
	- [CP.3:写数据显式共享最小化](#cp3显式共享写数据最小化) 
	- [CP.4:多用task而不是线程思考问题](#cp4多用task而不是线程思考问题) 
	- [CP.8:不要用volatile来做同步](#cp8不要用volatile来做同步) 
	- [CP.9:可行的情况下，用工具验证代码并发的正确性](#cp9可行的情况下用工具验证代码并发的正确性) 
- 还有一些文章:	
	- [CP.con:并发](#cpcon并发) 
	- [CP.coro:Coroutines](#cpcorocoroutines) 
	- [CP.par:Parallelism](#cpparparallelism) 
	- [CP.mess:Message passing](#cpmessmessage-passing) 
	- [CP.vec:Vectorization](#cpvecvectorization) 
	- [CP.free:Lock-free programming](#cpfreelock-free-programming) 
	- [CP.etc:Etc.并发规则](#cpetcetc并发规则) 

<br/>
### **CP.1:假定你的代码会是多线程运行的一部分**

**Example,bad**
```cpp
double cached_computation(int x)
{
    // bad: these statics cause data races in multi-threaded usage
    static int cached_x = 0.0;
    static double cached_result = COMPUTATION_OF_ZERO;

    if (cached_x != x) {
        cached_x = x;
        cached_result = computation(x);
    }
    return cached_result;
}
```

**Example,good**
```cpp
struct ComputationCache {
    int cached_x = 0;
    double cached_result = COMPUTATION_OF_ZERO;

    double compute(int x) {
        if (cached_x != x) {
            cached_x = x;
            cached_result = computation(x);
        }
        return cached_result;
    }
};
```

<br/>
### **CP.2:避免数据争用**

如果两个线程对一份数据访问（没有同步），且至少有一个是writer，那么就存在数据争用(data race)。关于如何消除data race来同步数据，可以参考一本关于并发性的好书（参见文献）。

**Example,bad**
```cpp
int get_id()
{
  static int id = 1;
  return id++;
}
```
有几处错误：
- 线程A加载id，系统上下文切换，A切出去，其他线程同时创建了几百个id，线程A再被切回来，id数值被写回，但写回数值只是加载时数值+1。
- 线程A和线程B加载和写回同时进行，最终得到的是相同id。

**Example,bad**
```cpp
void f(fstream& fs, regex pattern)
{
    array<double, max> buf;
    int sz = read_vec(fs, buf, max);            // read from fs into buf
    gsl::span<double> s {buf};
    // ...
    auto h1 = async([&] { sort(std::execution::par, s); });     // spawn a task to sort
    // ...
    auto h2 = async([&] { return find_all(buf, sz, pattern); });   // spawn a task to find matches
    // ...
}
```
sort函数同时读写buf，所以再这个栈上存在data race，这不容易被发现。

**Example,bad**
```cpp
// code not controlled by a lock

unsigned val;

if (val < 5) {
    // ... other thread can change val here ...
    switch (val) {
    case 0: // ...
    case 1: // ...
    case 2: // ...
    case 3: // ...
    case 4: // ...
    }
}
```
编译器可能使用5个size的跳转表来实现switch，一旦多线程data race造成val数值超出size，那么会造成严重bug。

**Enforcement:**
可以用一些开源或商业软件可以检测出此类问题，注意的是它也有成本和盲点。静态工具有很多误报，运行态工具又有很大开销。使用多种工具比单一工具可以捕捉更多问题。
- 避免全局数据
- 避免static数据
- 在栈中多用具体类型数据（不过多传递指针）
- 多使用不变量（常量，constexpr，const）

<br/>
### **CP.3:显式共享写数据最小化**

**原因** 如果不共享写数据，就不会有data race。共享越少，就越少忘掉同步访问数据（数据争用）。共享越少，就越少在等待锁（性能提高）。

**Example**
```cpp
bool validate(const vector<Reading>&);
Graph<Temp_node> temperature_gradients(const vector<Reading>&);
Image altitude_map(const vector<Reading>&);
// ...

void process_readings(const vector<Reading>& surface_readings)
{
    auto h1 = async([&] { if (!validate(surface_readings)) throw Invalid_data{}; });
    auto h2 = async([&] { return temperature_gradients(surface_readings); });
    auto h3 = async([&] { return altitude_map(surface_readings); });
    // ...
    h1.get();
    auto v2 = h2.get();
    auto v3 = h3.get();
    // ...
}
```
如果没有标明const，我们就需要review每个异步函数调用是否存在data race。让surface\_readings成为const，对于这个函数来说，就相当于只在这个函数体内使用。

**注意** 不变数据可以安全有效地被共享，这不会发生data race。另外见CP.mess:数据传递，以及CP.31:优先传递value。

<br/>
### **CP.4:多用task而不是线程思考问题**

**原因** 线程是关于机器的实现，task则是关于应用的实现概念。基于应用的概念更容易理解。

**Example**
```cpp
void some_fun(const std::string& msg)
{
    std::thread publisher([=] { std::cout << msg; });      // bad: less expressive
                                                           //      and more error-prone
    auto pubtask = std::async([=] { std::cout << msg; });  // OK
    // ...
    publisher.join();
}
```

**注意** 除了async，标准库的基础设施都是low-level，面向机器，线程和锁的级别。这是基础性的，但我们需要努力提高抽象水平：为了生产力，为了可靠性，为了性能。这是使用high level，更面向应用的库（如果可能的话，建立在标准库上）的有力论据。

<br/>
### **CP.8:不要用volatile来做同步**

**原因** 在C++，不像其他语言，volatile关键字不提供原子性，不保证线程间同步，不保证指令不会乱序（不管是编译器还是机器），它与并发没有关系。

**Example, bad:**
```cpp
int free_slots = max_slots; // current source of memory for objects

Pool* use()
{
    if (int n = free_slots--) return &pool[n];
}
```
data race, fix:
```cpp
volatile int free_slots = max_slots; // current source of memory for objects

Pool* use()
{
    if (int n = free_slots--) return &pool[n];
}
```
没有改变任何并发相关，data race依旧，改为：
```cpp
atomic<int> free_slots = max_slots; // current source of memory for objects

Pool* use()
{
    if (int n = free_slots--) return &pool[n];
}
```
free\_slots--操作是原子的，而不是那种读取-自增-写入模式，其他线程在这三个操作中都可能介入。

**替代** 在使用volatile的地方尽量使用atomic，更复杂的地方用mutex。

**See also** （极少见）[非C++内存时使用volatile](r#非C++内存时使用volatile)

<br/>
### **CP.9:可行的情况下，用工具验证代码并发的正确性**

经验告诉我们并发代码很难写正确，编译时检查、运行时检查、单元测试等发现并发错误，不如在顺序代码中有效，微小的bug都会造成巨大的灾难，包括内存损坏、死锁和安全漏洞。

**注意**
- 静态检测工具：Clang和一些旧版本的GCC都对线程安全的静态annotation提供一些支持。坚持使用这些技术可以将许多class的线程安全错误转化为编译器错误。annotation通常是局部的，
将一个特定的成员变量标记为在一个特定mutex保护下，通常容易被学习。但是像许多静态工具一样，会出现很多false negative，有些错误需要被捕捉到却被通过了。
- 动态检测工具：Clang的Thread SaniTizer(TSAN)是个强有力的工具，它改变了程序的构建和执行以增加对内存访问记录的登记，在二进制文件中特定执行中有绝对的识别。但是代价就是内存增加5-10倍，CPU下降2-20倍。这种动态检测工具适用于集成测试、金丝雀推送和多线程单元测试。当TSAN发现一个问题时，通常就代表它有效地识别出一个真实会发生的data race，但是它只能在一个特定的执行路径上才会识别。

<br/>
### **CP.con:并发**

本节重点讨论通过共享数据进行多线程交互
- 并行算法[parallelism](r#cp并行)
- 不通过显式共享的任务间交互[messing](r#cp消息传递)
- 向量并行[vectorization](r#cp向量并行)
- 无锁编程[lock free](r#cp无锁编程)

并发规则摘要：
- [CP.20:使用RAII，而不直接调用lock()/unlock()](#cp20使用raii而不直接调用lockunlock)
- [CP.21:使用std::mutex或std::scoped\_lock来获得多个mutex](#cp21使用stdmutex或stdscoped_lock来获得多个mutex)
- [CP.22:不要在lock代码段中调用未知代码](#cp22不要在lock代码段中调用未知代码)
- [CP.23:把一个joining线程当成scoped container](#cp23把一个joining线程当成scoped-container)
- [CP.24:把thread当成一个全局的container](#cp24把thread当成一个全局的container)
- [CP.25:优先使用gsl::joining\_thread，而不是thread](#cp25优先使用gsljoining_thread而不是thread)
- [CP.26:不要deatch线程](#cp26不要deatch线程)
- [CP.31:线程间small amount data用value传递，而不用引用或指针传递](#cp31线程间small-amount-data用value传递而不用引用或指针传递)
- [CP.32:unreleated线程之间应该用shared\_ptr共享所有权](#cp32unreleated线程之间应该用shared_ptr共享所有权)
- [CP.40:context switching最小化](#cp40context-switching最小化)
- [CP.41:线程创建与线程销毁最小化](#cp41线程创建与线程销毁最小化)
- [CP.42:不要在没有condition情况下调用wait](#cp42不要在没有condition情况下调用wait)
- [CP.43:关键临界区开销最小化](#cp43关键临界区开销最小化)
- [CP.44:所有的lock\_guard和unique\_lock都要记得命名](#cp44所有的lock_guard和unique_lock都要记得命名)
- [CP.50:如果要guard某份数据，那么定义一个mutex，如果可能的话，使用synchronized\_value\<T\>](#cp50如果要guard某份数据那么定义一个mutex如果可能的话使用synchronized_valuet)

<br/>
### **CP.20:使用RAII，而不直接调用lock()/unlock()**

**原因** 避免忘记释放锁

**Example, bad**
```cpp
mutex mtx;

void do_stuff()
{
    mtx.lock();
    // ... do stuff ...
    mtx.unlock();
}
```
一旦有人在...中return或者抛个异常或其他，都会导致忘记mtx.unlock。
```cpp
mutex mtx;

void do_stuff()
{
    unique_lock<mutex> lck {mtx};
    // ... do stuff ...
}
```

<br/>
### **CP.21:使用std::mutex或std::scoped_lock来获得多个mutex**

**原因** 避免在多个mutex上死锁

**Example**
```cpp
// thread 1
lock_guard<mutex> lck1(m1);
lock_guard<mutex> lck2(m2);

// thread 2
lock_guard<mutex> lck2(m2);
lock_guard<mutex> lck1(m1);
```
以上代码死锁
```cpp
// thread 1
lock(m1, m2);
lock_guard<mutex> lck1(m1, adopt_lock);
lock_guard<mutex> lck2(m2, adopt_lock);

// thread 2
lock(m2, m1);
lock_guard<mutex> lck2(m2, adopt_lock);
lock_guard<mutex> lck1(m1, adopt_lock);
```
C++17更佳
```cpp
// thread 1
scoped_lock<mutex, mutex> lck1(m1, m2);

// thread 2
scoped_lock<mutex, mutex> lck2(m2, m1);
```
还有
```cpp
lock_guard lck1(m1, adopt_lock);
```
m1的mutex类型会被推导出。

<br/>
### **CP.22:不要在lock代码段中调用未知代码**

**原因** 未知代码调用可能导致死锁

**Example**
```cpp
void do_this(Foo* p)
{
    lock_guard<mutex> lck {my_mutex};
    // ... do something ...
    p->act(my_data);
    // ...
}
```
Foo::act可能是一个虚函数，最终调用的是暂未实现的子类函数，它可能重复调用do\_this，造成递归调用死锁。也可能在等待另外一个mutex的锁，它没有在合理时间返回，导致do\_this调用阻塞。

**Example**
调用未知代码的一个常见例子，比如对同个对象的重复锁，这种可以通过recursive\_mutex解决。
```cpp
recursive_mutex my_mutex;

template<typename Action>
void do_something(Action f)
{
    unique_lock<recursive_mutex> lck {my_mutex};
    // ... do something ...
    f(this);    // f will do something to *this
    // ...
}
```

**Enforcement**
- non-recursive锁保护代码中调用虚函数要标记出。
- non-recursive锁保护代码中调用callback要标记出。

<br/>
### **CP.23:把一个joining线程当成scoped container**

**原因** 为保证指针安全和避免泄漏，我们要考虑thread中使用了哪些指针。当一个线程join时，我们可以安全地把指针传递给线程内以及线程外层作用域内的对象。

**Example**
```cpp
void f(int* p)
{
    // ...
    *p = 99;
    // ...
}
int glob = 33;

void some_fct(int* p)
{
    int x = 77;
    joining_thread t0(f, &x);           // OK
    joining_thread t1(f, p);            // OK
    joining_thread t2(f, &glob);        // OK
    auto q = make_unique<int>(99);
    joining_thread t3(f, q.get());      // OK
    // ...
}
```
gsl::joining\_thread表示一个带有析构函数的std::thread，它可以join，但是不可以detach。OK的意思是，只要线程能使用指向某对象的指针，该对象就会在范围内存活。线程并发不影响这里的生命期或所有权问题。

**Enforcement**
确保joining\_thread不会被detach()

<br/>
### **CP.24:把thread当成一个全局的container**

**原因** 为了指针安全以及避免泄漏，我们要考虑thread使用了哪些指针，如果一个thread是detach的，我们可以安全地将指针传递给static对象和free store object。

**Example**
```cpp
void f(int* p)
{
    // ...
    *p = 99;
    // ...
}

int glob = 33;

void some_fct(int* p)
{
    int x = 77;
    std::thread t0(f, &x);           // bad
    std::thread t1(f, p);            // bad
    std::thread t2(f, &glob);        // OK
    auto q = make_unique<int>(99);
    std::thread t3(f, q.get());      // bad
    // ...
    t0.detach();
    t1.detach();
    t2.detach();
    t3.detach();
    // ...
}
```
OK表示线程使用的指针，指针所指的对象，在线程使用指针时都是活着的。BAD表示线程使用指针时，其所指对象可能已经destroyed。线程并发不影响其生命期和所有权。

**注意** 即使是指向static对象也可能存在问题，detach线程可能一直运行到程序结束，那时static对象将会销毁，这时detach线程中可能还有使用指向该对象的指针，那么会造成race。

**注意** 如果不使用detach线程或者仅使用gsl::joining\_thread，那么本规则是多余的。

**Enforcement** 试图将局部变量传递给detach线程会被标记出。

<br/>
### **CP.25:优先使用gsl::joining_thread，而不是thread**

**原因** joining\_thread会在最后join，detach线程不好监控，deatch线程很难保证不会出错。

**Example,bad**
```cpp
void f() { std::cout << "Hello "; }

struct F {
    void operator()() const { std::cout << "parallel world "; }
};

int main()
{
    std::thread t1{f};      // f() executes in separate thread
    std::thread t2{F()};    // F()() executes in separate thread
}  // spot the bugs
```
**Example**
```cpp
void f() { std::cout << "Hello "; }

struct F {
    void operator()() const { std::cout << "parallel world "; }
};

int main()
{
    std::thread t1{f};      // f() executes in separate thread
    std::thread t2{F()};    // F()() executes in separate thread

    t1.join();
    t2.join();
}  // one bad bug left
```

**注意** 把一些不朽的thread做成全局的，把它们放入一个封闭的作用域内，或者把它们放入free store上，而不是detach。

**注意** 因为很多旧代码以及第三方库使用的是std::thread，因此本规则很难引入。

**Enforcement**
- 建议使用gsl::joining\_thread或C++20的std::jthread。
- 如果线程detach，建议把所有权导出到一个封闭scoped。
- 一个线程是否join或detach不明，应发出警告。

<br/>
### **CP.26:不要deatch线程**

**原因** detach线程很难被监控以及交互，很难保证deatch线程按预期完成及生命期。

**Example**
```cpp
void heartbeat();

void use()
{
    std::thread t(heartbeat);             // don't join; heartbeat is meant to run forever
    t.detach();
    // ...
}
```
以上代码，我们如何保证detach线程是否还存活？heartbeat函数中可能出错，心跳丢失可能对一个系统来说是非常严重，所以需要和这个心跳线程交互，（比如通过condition\_variable）。

另一种且更好的解决方案是把它放在更外围的scope，来创建或激活，这样能更好地管理生命期。
```cpp
void heartbeat();

gsl::joining_thread t(heartbeat);             // heartbeat is meant to run "forever"
```
这个心跳线程，除非遇到错误、硬件问题等，不然会随着程序一直运行。

有时候我们需要分离线程创建与线程所有权。
```cpp
void heartbeat();

unique_ptr<gsl::joining_thread> tick_tock {nullptr};

void use()
{
    // heartbeat is meant to run as long as tick_tock lives
    tick_tock = make_unique<gsl::joining_thread>(heartbeat);
    // ...
}
```

<br/>
### **CP.31:线程间small amount data用value传递，而不用引用或指针传递**

**原因** 小数据copy比较便宜，对这小数据进行锁同步反而开销更大，copy天然表示unique所有权，消除了data race。

**Example**
```cpp
string modify1(string);
void modify2(string&);

void fct(string& s)
{
    auto res = async(modify1, s);
    async(modify2, s);
}
```
modify1拷贝，modify2不是，modify1单线程多线程都可用，modify2多线程则需要同步。如果string比较短，比如10个字符，那么调用modify1非常快，开销最多的将是线程切换。如果string长度超过100万个字符，拷贝显然不合适。

<br/>
### **CP.32:unreleated线程之间应该用shared_ptr共享所有权**

<br/>
### **CP.40:context switching最小化**

<br/>
### **CP.41:线程创建与线程销毁最小化**

**原因** 线程创建开销很大

**Example**
```cpp
void worker(Message m)
{
    // process
}

void dispatcher(istream& is)
{
    for (Message m; is >> m; )
        run_list.push_back(new thread(worker, m));
}
```
每一条message都要创建一个线程，run\_list最终也要负责将所有线程都销毁。

**Example**
```cpp
Sync_queue<Message> work;

void dispatcher(istream& is)
{
    for (Message m; is >> m; )
        work.put(m);
}

void worker()
{
    for (Message m; m = work.get(); ) {
        // process
    }
}

void workers()  // set up worker threads (specifically 4 worker threads)
{
    joining_thread w1 {worker};
    joining_thread w2 {worker};
    joining_thread w3 {worker};
    joining_thread w4 {worker};
}
```

**注意** 如果你的系统有更好的线程池，或者有更好的消息队列，那么直接用这取代。

<br/>
### **CP.42:不要在没有condition情况下调用wait**

**原因** wait如果没有等待在某个condition，可能会错过wakeup，或者被wakeup之后发现没有事可做。

**Example,bad**
```cpp
std::condition_variable cv;
std::mutex mx;

void thread1()
{
    while (true) {
        // do some work ...
        std::unique_lock<std::mutex> lock(mx);
        cv.notify_one();    // wake other thread
    }
}

void thread2()
{
    while (true) {
        std::unique_lock<std::mutex> lock(mx);
        cv.wait(lock);    // might block forever
        // do work ...
    }
}
```
如果其他线程消费了thread1的通知，那么thread2将永远等待。

**Example**
```cpp
template<typename T>
class Sync_queue {
public:
    void put(const T& val);
    void put(T&& val);
    void get(T& val);
private:
    mutex mtx;
    condition_variable cond;    // this controls access
    list<T> q;
};

template<typename T>
void Sync_queue<T>::put(const T& val)
{
    lock_guard<mutex> lck(mtx);
    q.push_back(val);
    cond.notify_one();
}

template<typename T>
void Sync_queue<T>::get(T& val)
{
    unique_lock<mutex> lck(mtx);
    cond.wait(lck, [this] { return !q.empty(); });    // prevent spurious wakeup
    val = q.front();
    q.pop_front();
}
```
现在如果get线程wakeup时发现queue为空（比如另外一个线程在之前先调用了get），它将立即sleep，等待wakeup。

**Enforcement** 所有wait没有等在某个condition将会被标记。

<br/>
### **CP.43:关键临界区开销最小化**

**原因** mutex临界区越小，其他线程越少等待。线程暂停和恢复开销比较大。

**Example**
```cpp
void do_something() // bad
{
    unique_lock<mutex> lck(my_lock);
    do0();  // preparation: does not need lock
    do1();  // transaction: needs locking
    do2();  // cleanup: does not need locking
}
```
改为
```cpp
void do_something() // bad
{
    do0();  // preparation: does not need lock
    my_lock.lock();
    do1();  // transaction: needs locking
    my_lock.unlock();
    do2();  // cleanup: does not need locking
}
```
最好用RAII
```cpp
void do_something() // OK
{
    do0();  // preparation: does not need lock
    {
        unique_lock<mutex> lck(my_lock);
        do1();  // transaction: needs locking
    }
    do2();  // cleanup: does not need locking
}
```

<br/>
### **CP.44:所有的lock_guard和unique_lock都要记得命名**

**原因** 所有未命名的local对象是一个立即超出scope的临时对象。

**Example**
```cpp
unique_lock<mutex>(m1);
lock_guard<mutex> {m2};
lock(m1, m2);
```

<br/>
### **CP.50:如果要guard某份数据，那么定义一个mutex，如果可能的话，使用synchronized_value\<T\>**

**原因** 使用synchronized\_value\<T\>可以确保数据有一个mutex，并且在数据访问时有正确的mutex锁定。参见WG21在未来TS或C++修订版中加入synchronized\_value\<T\>的建议。

**Example**
```cpp
struct Record {
    std::mutex m;   // take this mutex before accessing other members
    // ...
};

class MyClass {
    struct DataRecord {
       // ...
    };
    synchronized_value<DataRecord> data; // Protect the data with a mutex
};
```

<br/>
## **CP.coro:Coroutines**

Coroutine规则摘要：
- [CP.51:不要使用coroutines的捕获lambda](#cp51不要使用属于coroutines的捕获lambda)
- [CP.52:不要在横跨suspend points上持有锁或其他同步原语](#cp52不要在across-suspension-points上加锁或其他同步原语)
- [CP.53:coroutine的参数不应该传引用](#cp53coroutine的参数不应该传引用)

<br/>
### **CP.51:不要使用属于coroutines的捕获lambda**

**原因** 普通lambda所使用的模式在coroutine lambda中会变得危险。因为coroutine lambda有可能访问了freed memory在第一次suspension point之后，即使是smart pointer或者是可拷贝的类型。一个lambda会产生一个带有存储空间的闭包对象，它将在某个时间点上超出范围。闭包对象超出范围后，所捕获对象也会超出范围。普通的lambda在这个时间已经完成执行，所以不是个问题，而coroutine lambda可以在闭包对象析构后从暂停中恢复，这时所有的捕获都是free的内存访问。

**Example,bad**
```cpp
int value = get_value();
std::shared_ptr<Foo> sharedFoo = get_foo();
{
  const auto lambda = [value, sharedFoo]() -> std::future<void>
  {
    co_await something();
    // "sharedFoo" and "value" have already been destroyed
    // the "shared" pointer didn't accomplish anything
  };
  lambda();
} // the lambda closure object has now gone out of scope
```
**Example,better**
```cpp
int value = get_value();
std::shared_ptr<Foo> sharedFoo = get_foo();
{
  // take as by-value parameter instead of as a capture
  const auto lambda = [](auto sharedFoo, auto value) -> std::future<void>
  {
    co_await something();
    // sharedFoo and value are still valid at this point
  };
  lambda(sharedFoo, value);
} // the lambda closure object has now gone out of scope`
```
**Example,best**
```cpp
std::future<void> Class::do_something(int value, std::shared_ptr<Foo> sharedFoo)
{
  co_await something();
  // sharedFoo and value are still valid at this point
}

void SomeOtherFunction()
{
  int value = get_value();
  std::shared_ptr<Foo> sharedFoo = get_foo();
  do_something(value, sharedFoo);
}
```

**Enforcement** 一个lambda是coroutine，且有个非空的捕获列表，那应该标记出。

<br/>
### **CP.52:不要在across suspension points上加锁或其他同步原语**

**原因** 如果这样的话会有死锁风险。有一些wait需要当前线程做更多的工作，直到异步操作完成。如果持有锁的线程所做的工作需要相同的锁，那将会死锁，因为已经持有一个相同的锁。如果coroutine持有一个线程的锁，然后在另外一个线程处理完成那将是未定义的。coroutine即使在同个线程返回，但是在resume前抛出异常，那么lock guard不会被解构。

**Example,bad**
```cpp
std::mutex g_lock;

std::future<void> Class::do_something()
{
    std::lock_guard<std::mutex> guard(g_lock);
    co_await something(); // DANGER: coroutine has suspended execution while holding a lock
    co_await somethingElse();
}
```

**Example,good**
```cpp
std::mutex g_lock;

std::future<void> Class::do_something()
{
    {
        std::lock_guard<std::mutex> guard(g_lock);
        // modify data protected by lock
    }
    co_await something(); // OK: lock has been released before coroutine suspends
    co_await somethingElse();
}
```

**注意** 这种模式对性能也不好。如果达到一个suspension point，比如co\_await，那么当前函数就会停止，其他代码开始运行。而且\
这个coroutine恢复前可能要等待很久时间，这整个过程如果有锁，那么会一直被持有，而其他地方要获取这个锁却获取不上。

**Enforcement** 所有lock guard在coroutine suspend前没有destructed都应该被标记出来。

<br/>
### **CP.53:coroutine的参数不应该传引用**

**原因** 一旦coroutine达到一个suspend point，比如co\_wait，同步部分就会返回。那之后，任何通过引用传递的参数都是dangling的。任何对此操作都是未定义的，可能包括对已释放内存的操作。

**Example, Bad**
```cpp
std::future<int> Class::do_something(const std::shared_ptr<int>& input)
{
    co_await something();

    // DANGER: the reference to input may no longer be valid and may be freed memory
    co_return *input + 1;
}
```

**Example,Good**
```cpp
std::future<int> Class::do_something(std::shared_ptr<int> input)
{
    co_await something();
    co_return *input + 1; // input is a copy that is still valid here
}
```

**注意** 这个问题不单单是出现在第一次suspend point，后续代码如果添加或者删除suspend point都会重现这个bug，有些coroutine甚至在第一行代码执行前都有suspend point，这种情况下同样是unsafe的。所以一般传value，value拷贝后在整个coroutine内都是安全访问的。

**Enforcement** 所有传引用给coroutine都应该被标记。

<br/>
## **CP.mess:Message passing**

标准库的设施是low-level的，它专注于使用thread、mutex、atomic等接近硬件编程的需求。大多数人不应该工作在这一层面，因为它容易出错，而且开发效率不高。如果可能的话，使用更高层次的设施，比如消息库，并行算法，vectorization等。本节讨论消息传递，以便程序员不必进行显式同步。

消息传递原则摘要：
- [CP.60:使用future从并发的task中返回value](#cp60使用future从并发的task中返回value)
- [CP.61:使用async来新建并发任务](#cp61使用async来新建并发任务)
- 消息队列
- 消息传递库

<br/>
### **CP.60:使用future从并发的task中返回value**

<br/>
### **CP.61:使用async来新建并发任务**

**原因** 尽量避免用原生thread，promise。使用工厂函数比如std::async，它来处理创建和复用一个thread，而不必要得暴露原生thread。

**Example**
```cpp
int read_value(const std::string& filename)
{
    std::ifstream in(filename);
    in.exceptions(std::ifstream::failbit);
    int value;
    in >> value;
    return value;
}

void async_example()
{
    try {
        std::future<int> f1 = std::async(read_value, "v1.txt");
        std::future<int> f2 = std::async(read_value, "v2.txt");
        std::cout << f1.get() + f2.get() << '\n';
    } catch (const std::ios_base::failure& fail) {
        // handle exception here
    }
}
```

**注意** 不幸的是std::async不是完美的，它没有使用一个线程池，这意味着如果资源耗尽将会失败，而没有把你的任务排队起来，留着之后来处理。但是即使不能使用std::async，你也应该倾向于编写自己的返回future的工厂函数，而不是使用原生的promise。

**Example,bad**

这个例子展示了两个不同的方式，成功地使用了std::future，但在避免使用原生thread管理上失败了。
```cpp
void async_example()
{
    std::promise<int> p1;
    std::future<int> f1 = p1.get_future();
    std::thread t1([p1 = std::move(p1)]() mutable {
        p1.set_value(read_value("v1.txt"));
    });
    t1.detach(); // evil

    std::packaged_task<int()> pt2(read_value, "v2.txt");
    std::future<int> f2 = pt2.get_future();
    std::thread(std::move(pt2)).detach();

    std::cout << f1.get() + f2.get() << '\n';
}
```

**Example,good**
```cpp
void async_example(WorkQueue& wq)
{
    std::future<int> f1 = wq.enqueue([]() {
        return read_value("v1.txt");
    });
    std::future<int> f2 = wq.enqueue([]() {
        return read_value("v2.txt");
    });
    std::cout << f1.get() + f2.get() << '\n';
}
```

WorkQueue背后隐藏了调用read\_value所产生的线程创建等操作，用户只处理future，而不用面对原生的thread，promise，packaged\_task等。

<br/>
## **CP.free:Lock-free programming**

用mutex和condition\_variable来同步相对比较昂贵，此外还可能导致死锁。为了提高性能以及消除死锁可能性，不得不使用low-level的"lock-free"设施，这些设施依赖于短时独占内存（atomic）。无锁编程也用来实现更高级别的并发机制，比如thread和mutex。

无锁编程规则摘要：
- [CP.100:不要使用无锁编程，除非你绝对需要这样做](#cp100不要使用无锁编程除非你绝对需要这样做)
- [CP.101:不要信任你的硬件/编译器组合](#cp101不要信任你的硬件编译器组合)
- [CP.102:仔细研究文献](#cp102仔细研究文献)
- 怎么/什么时候用atomic
- 避免饥饿
- 使用无锁数据结构，而不是自己手工特定的无锁访问
- [CP.110:不要为初始化编写双重检查锁](#cp110不要为初始化编写双重检查锁)
- [CP.111:如果你真的需要双重检查锁，请使用常规模式](#cp111如果你真的需要双重检查锁请使用常规模式)
- 怎么/什么时候进行比较和互换

<br/>
### **CP.100:不要使用无锁编程，除非你绝对需要这样做**

**原因** 容易出错，需要对语言特性，机器结构和数据结构有专业水准的掌握。

**Example,bad**
```cpp
extern atomic<Link*> head;        // the shared head of a linked list

Link* nh = new Link(data, nullptr);    // make a link ready for insertion
Link* h = head.load();                 // read the shared head of the list

do {
    if (h->data <= data) break;        // if so, insert elsewhere
    nh->next = h;                      // next element is the previous head
} while (!head.compare_exchange_weak(h, nh));    // write nh to head or to h
```
此类代码中bug，通过测试来发现它真的很难。阅读关于ABA问题的文章。

**例外** 只要你使用顺序一致的内存模型（memory\_order\_seq\_cst），就可以简单安全使用atomic变量。

**注意** higher-level的并发机制，比如thread和mutex，是使用无锁编程实现的。

**替代方案** 使用他人实现的无锁数据结构作为某些库的一部分。

<br/>
### **CP.101:不要信任你的硬件/编译器组合**

**原因** 无锁编程所使用的low-level硬件接口是最难实现的，也是最微妙的出现可移植问题。

**注意** 指令reordering（静态和动态）使我们很难在这个层次上有效思考（尤其使用relaxed内存模型）。

<br/>
### **CP.102:仔细研究文献**

**原因** 除了原子和其他一些标准模式外，无锁编程实际上是一个专家话题。在给别人编写无锁代码前先成为专家。

**参考资料**
- Anthony Williams: C++ concurrency in action. Manning Publications.
- Boehm, Adve, You Don’t Know Jack About Shared Variables or Memory Models , Communications of the ACM, Feb 2012.
- Boehm, “Threads Basics”, HPL TR 2009-259.
- Adve, Boehm, “Memory Models: A Case for Rethinking Parallel Languages and Hardware”, Communications of the ACM, August 2010.
- Boehm, Adve, “Foundations of the C++ Concurrency Memory Model”, PLDI 08.
- Mark Batty, Scott Owens, Susmit Sarkar, Peter Sewell, and Tjark Weber, “Mathematizing C++ Concurrency”, POPL 2011.
- Damian Dechev, Peter Pirkelbauer, and Bjarne Stroustrup: Understanding and Effectively Preventing the ABA Problem in Descriptor-based Lock-free Designs. 13th IEEE Computer Society ISORC 2010 Symposium. May 2010.
- Damian Dechev and Bjarne Stroustrup: Scalable Non-blocking Concurrent Objects for Mission Critical Code. ACM OOPSLA’09. October 2009
- Damian Dechev, Peter Pirkelbauer, Nicolas Rouquette, and Bjarne Stroustrup: Semantically Enhanced Containers for Concurrent Real-Time Systems. Proc. 16th Annual IEEE International Conference and Workshop on the Engineering of Computer Based Systems (IEEE ECBS). April 2009.
- Maurice Herlihy, Nir Shavit, Victor Luchangco, Michael Spear, “The Art of Multiprocessor Programming”, 2nd ed. September 2020

<br/>
### **CP.110:不要为初始化编写双重检查锁**

**原因** C++11开始，static局部变量的初始化是线程安全实现的。当与RAII模式结合时，static局部变量可以取代自己编写双重检查初始化。std::call\_once同样也可以达到目的。

**Example** 
```cpp
void f()
{
    static std::once_flag my_once_flag;
    std::call_once(my_once_flag, []()
    {
        // do this only once
    });
    // ...
}
```
或者
```cpp
void f()
{
    // Assuming the compiler is compliant with C++11
    static My_class my_object; // Constructor called only once
    // ...
}

class My_class
{
public:
    My_class()
    {
        // do this only once
    }
};
```

<br/>
### **CP.111:如果你真的需要双重检查锁，请使用常规模式**

**Example,bad**
```cpp
mutex action_mutex;
volatile bool action_needed;

if (action_needed) {
    std::lock_guard<std::mutex> lock(action_mutex);
    if (action_needed) {
        take_action();
        action_needed = false;
    }
}
```

**Example,good**
```cpp
mutex action_mutex;
atomic<bool> action_needed;

if (action_needed) {
    std::lock_guard<std::mutex> lock(action_mutex);
    if (action_needed) {
        take_action();
        action_needed = false;
    }
}
```
以下更好，fine-tuned内存顺序比顺序一致的内存顺序，对于acquire load更有益。
```cpp
mutex action_mutex;
atomic<bool> action_needed;

if (action_needed.load(memory_order_acquire)) {
    lock_guard<std::mutex> lock(action_mutex);
    if (action_needed.load(memory_order_relaxed)) {
        take_action();
        action_needed.store(false, memory_order_release);
    }
}
```

<br/>
## **CP.etc:Etc.并发规则**

- [CP.200:非C++内存时使用volatile](#cp200非c内存时使用volatile)
- [CP.201:Signals](#cp201signals)

<br/>
### **CP.200:非C++内存时使用volatile**

**原因** volatile是用来指与"非C++"代码或不遵循C++内存模型的硬件共享的对象。

**Example**
```cpp
const volatile long clock;
```
这描述了一个由时钟电路持续更新的寄存器。时钟是易变的，使用它的C++程序没有任何动作情况下，它的值也会发生变化。\
例如，读取两次时钟往往会产生两个不同的值，所以优化器最好不要在这段代码中优化掉第二次读取。 
```cpp
long t1 = clock;
// ... no use of clock here ...
long t2 = clock;
```
clock是const的因为程序不会尝试往clock中写。

**注意** 除非你正在编写直接操作硬件的最底层的代码，将volatile视为最好可以避免的深奥问题。

**Example** 通常情况下，C++代码会收到属于其他地方（硬件或其他语言）的volatile内存。
```cpp
int volatile* vi = get_hardware_memory_location();
    // note: we get a pointer to someone else's memory here
    // volatile says "treat this with extra respect"
```
有时，C++代码分配了volatile内存，并通过转义指针与其他地方（硬件或其他语言）分享。
```cpp
static volatile long vl;
please_use_this(&vl);   // escape a reference to this to "elsewhere" (not C++)
```

**Example, bad**

volatile局部变量几乎总是错误的。如果它们是短暂的，那怎么可能与其他硬件或语言共享？这一点原因同样适用于成员变量。
```cpp
void f()
{
    volatile int i = 0; // bad, volatile local variable
    // etc.
}

class My_type {
    volatile int i = 0; // suspicious, volatile member variable
    // etc.
};
```

**注意** C++与其他语言不同，volatile与同步没有关系。

**Enforcement** 所有volatile的局部变量或成员变量都应该标记出，几乎可以肯定的是，可以使用atomic<T>来替代。

<br/>
### **CP.201:Signals**

可能更值得提醒的是很少代码是异步信号安全的，以及如何与信号处理函数交互。（最好的可能是根本不需要）

<br/>
### **F.60:当no argument也是一个选项时，优先使用T\*而不是T&**

原因：指针可能为空nullptr，但引用不会，不存在空引用。有时用nullptr来代表no object，如果不是这样，用引用会让代码更简单。

Example
```cpp
string zstring_to_string(zstring p) // zstring is a char*; that is a C-style string
{
    if (!p) return string{};    // p might be nullptr; remember to check
    return string{p};
}

void print(const vector<int>& r)
{
    // r refers to a vector<int>; no check needed
}
```
